# Architecture — AML/KYC/sanctions screening (RC-02)

> **BMAD/SA artefact — the technical spine** (Path-C). Assembles the C4 views, data design, API contract, NFR
> register, threat-model summary and the ADR into one blueprint. **Status: DRAFT for review — no build until
> validated** (`AGENTS.md`). Decision of record: **ADR-001** (adopt yente, build the gate). Owner = SA/TL + SEC.
> Synthetic-enterprise: built to *demonstrate* the capability, not a live legal obligation.

Three audience views (read the one you need): **§1–2 Conceptual** (PM/stakeholder) · **§3–7 Component**
(engineers) · **§8–13 Operational** (DevOps/SRE/SEC). Requirement IDs (`FR/NFR/SEC/AI-…`) thread to the SPEC.

## 1. Overview (Conceptual)
RC-02 closes two 🔴 register gaps — **LCB-FT/AML-CFT + KYC/KYB** and **Sanctions / gel des avoirs** — by making
**screening a GATE** in the party lifecycle: before a party is onboarded, bound to a policy, or paid, it is
screened against sanctions/PEP/adverse-media/risk-country/UBO lists; a **hit raises an alert a human decides**
(accept/refuse, with a risk + vigilance level); every screening is **audited**. The book is **re-screened nightly**
because watchlists change daily. **The AI triage layer** pre-reasons over alerts to cut false-positive review
load — recommending, never deciding.

**Technical → user-outcome translation** (NFRs for non-engineers):

| Technical | User outcome |
|---|---|
| Sync screening gate < 2s p95 at bind/pay | onboarding/binding isn't slowed to a crawl |
| List-staleness alarm < 24h + data-version logged per screen | "we can prove the lists were current when we screened" (the auditor's question) |
| Human-in-the-loop on every REF | no customer wrongly refused — and no true sanctions hit silently auto-cleared |
| Append-only audit of who/what/when/why | a regulator can reconstruct every check + decision |

## 2. C4 — Level 1: System Context (Conceptual)
```mermaid
flowchart LR
  officer(["Compliance officer / analyst<br/>(reviews alerts, decides)"])
  subs(["Submission / Underwriting / Billing<br/>(call the gate at intake / bind / pay)"])
  screening["Screening capability<br/>(RC-02): gate · alerts · cases · audit"]
  yente["yente (OpenSanctions)<br/>self-hosted matching engine"]
  lists[("Official watchlists<br/>OFAC · EU · UN · UK — free")]
  ai["Governed AI triage<br/>(LiteLLM/Qdrant/Langfuse, HITL)"]
  bus["NATS (events)"]
  officer -->|review + decide ACC/REF| screening
  subs -->|screen(party) → ALLOW/HOLD| screening
  screening -->|match query (name/DOB/nat)| yente
  yente -->|reindex (daily)| lists
  screening -->|alert events| bus --> ai
  ai -->|recommendation + rationale| screening
  classDef p fill:#08427b,color:#fff; classDef s fill:#1168bd,color:#fff; classDef ext fill:#6b6b6b,color:#fff
  class officer,subs p; class screening,ai s; class yente,lists,bus ext
```

## 3. C4 — Level 2: Containers (Component)
```mermaid
flowchart TB
  subgraph core["ktayl-core (modular monolith)"]
    scr["screening MODULE<br/>domain · application · api(ScreeningGate) · infra(web/persistence/YenteClient)"]
    pol["policy module"]; bil["billing module"]
  end
  uw["ktayl-underwriting<br/>(Python svc)"]
  sub["Submission Hub #27"]
  yente["yente svc<br/>(+ OpenSearch index)"]
  pg[("screening schema<br/>CNPG")]
  cron["nightly re-screen<br/>CronJob (same image)"]
  aiw["AI triage worker<br/>(Python, governed)"]
  pol -->|in-process gate| scr
  bil -->|in-process gate| scr
  uw -->|REST /v1/screening/screen + M2M (ADR-016)| scr
  sub -->|REST| scr
  scr --> yente
  scr --> pg
  cron --> scr
  scr -->|alert events NATS| aiw -->|recommendation| scr
```

## 4. The `screening` module (ktayl-core) — components (Component)
CLOSED Spring Modulith, hexagonal (ADR-013), schema-per-module (ADR-006). Sibling to policy/billing/complaints.
- **domain** (pure): `Party` (name/DOB/nationality/residence/IDs/relationships incl. UBO), `ScreeningResult` with
  **hit buckets** `{SANCTION, PEP, AME, RISK_COUNTRY, UBO, ORIAS}`, `Alert`, `Case` (status `ACC|REF|PENDING` ·
  risk `WEAK|MED|STRONG` · vigilance `NORMAL|ENHANCED`), ports `ScreeningEngine` (outbound) + `ScreeningAuditLog`.
  **Settlement/refuse/block authority is domain logic** (a REF/HOLD is a control, not a UI concern). `FR-SCR-01..06`.
- **application**: `ScreenParty` (sync) · `DecideAlert` (human decision, authority-checked) · `RescreenBook` (batch).
- **api** (`@NamedInterface`): `ScreeningGate.screen(party) → ALLOW | HOLD(alertId)` — in-process for policy/billing.
- **infrastructure**: `web` (REST `/v1/screening/*`), `persistence` (the `screening` schema), `engine` =
  `YenteClient` (HTTP → yente `/match`), `events` (NATS alert/lifecycle emit).

## 5. The gates — where screening is invoked (Component)
| Gate (`FR`) | Caller | Call | On HIT |
|---|---|---|---|
| **Intake** `FR-SCR-10` | Submission Hub #27 | REST | flag the submission; don't progress to UW |
| **Bind** `FR-SCR-11` | policy module (in-process) · underwriting (REST + M2M) | `screen(party)` before `POST /v1/policies` | **HOLD blocks the bind** until a human clears |
| **Payment** `FR-SCR-12` | billing module (in-process) | `screen(party)` before Stripe collection | block the collection + run **asset-freeze** `FR-SCR-13` |
| **Nightly re-screen** `FR-SCR-14` | CronJob | `RescreenBook` (delta) | new alerts → queue → AI triage → analyst |

```mermaid
sequenceDiagram
  participant UW as Underwriting
  participant S as screening module
  participant Y as yente
  UW->>S: POST /v1/screening/screen (party) [M2M token]
  S->>Y: /match (name,DOB,nationality)
  Y-->>S: scored candidates (+ dataset version)
  S->>S: decide ALLOW / HOLD; write audit (data-version)
  alt clean
    S-->>UW: ALLOW → UW proceeds to POST /v1/policies
  else hit
    S-->>UW: HOLD(alertId) → bind blocked; analyst reviews
  end
```

## 6. Data design — the `screening` schema (Component) — `FR-SCR-20`
Flyway-owned, schema-per-module (ADR-006). ERD:
```mermaid
erDiagram
  PARTY ||--o{ SCREENING : "screened in"
  SCREENING ||--o{ ALERT : "raises"
  ALERT ||--o| CASE : "resolved by"
  PARTY { uuid id PK; text external_ref; text name; date dob; text nationality; text residence; jsonb ids; text ubo_of }
  SCREENING { uuid id PK; uuid party_id FK; text gate; timestamptz at; text dataset_version; text verdict }
  ALERT { uuid id PK; uuid screening_id FK; text bucket; int score; jsonb matched_entry }
  CASE { uuid id PK; uuid alert_id FK; text status; text risk; text vigilance; text actor_id; text rationale; timestamptz decided_at }
  SCREENING_AUDIT { bigint id PK; uuid party_id; text action; text actor_id; text dataset_version; jsonb detail; timestamptz at }
```
`dataset_version` on every `SCREENING` row is the **"lists were current" evidence**. `SCREENING_AUDIT` is append-only (LCB-FT/DORA).

## 7. API contract (Component) — `FR-SCR-30`; committed `api/openapi.yaml` at build
- `POST /v1/screening/screen` → `{verdict: ALLOW|HOLD, alertId?, datasetVersion}` (auth: M2M scope `screening:write`).
- `GET /v1/screening/parties/{id}` → current status (the "Inventory" pull).
- `GET /v1/screening/alerts?status=&bucket=` → list (keyset, like policy BF-POL-04).
- `POST /v1/screening/alerts/{id}/decision` → `{status, risk, vigilance, rationale}` (human; group-gated).
- Webhook (push) out on decision — downstream sync. Error model 4xx-never-5xx; idempotent screen by (party, gate, day).

## 8. NFR register (Operational) — `NFR-SCR-*`
| NFR | Target |
|---|---|
| `NFR-SCR-01` latency | sync gate p95 < 2s (yente match + decide) |
| `NFR-SCR-02` availability | gate 99.5% dev-lab SLO; **fail-closed** on sanctions path (a gate outage HOLDs, never silently ALLOWs) |
| `NFR-SCR-03` freshness | sanctions dataset age < 24h; **staleness alarm** + data-version logged per screen |
| `NFR-SCR-04` retention | audit ≥ 5y (LCB-FT); read-model/alerts live |
| `NFR-SCR-05` scale | nightly re-screen the whole book within the window |
| `NFR-SCR-06` cost | fits the ns quota; LLM triage tokens budgeted (LiteLLM team) |

## 9. Security + threat model (Operational, STRIDE-lite) — `SEC-SCR-*`
Trust boundary: party PII (**P2/P3**) + the sanctions decision = a **control gate**.
- `SEC-SCR-01` **AuthN/Z** — every API SSO/M2M (Authentik, ADR-016); alert-decision **group-gated** (Compliance Officer); the **REF/HOLD/freeze authority is server-side** (never client-only).
- `SEC-SCR-02` **PII** — no unmasked PII to any LLM (**Presidio-masked**, ADR-005); P2/P3 data stays in-house (no vendor egress — the sovereignty reason yente was chosen).
- `SEC-SCR-03` **Egress (default-deny)** — only the module → yente; only yente → the list sources; nothing else.
- `SEC-SCR-04` **Freshness-as-control** (T: *tampering/availability*) — a failed reindex keeps serving the last good index (looks healthy, silently stale) → the staleness alarm is the mitigation, not the config.
- `SEC-SCR-05` **Audit integrity** — append-only; the AI rationale is logged with the human decision.
- Threats mapped: name-collision false-negatives, injection into the match query (bound params), an auto-cleared true hit (→ HITL hard-rule), a stale-list miss (→ freshness alarm).

## 10. AI triage (Component/Operational) — `AI-SCR-*`
A Python worker (governed-AI pattern, `llmops.md`): consumes alert events → LLM corroborates (DOB/nat/ID) → emits
**recommendation + written rationale** (clear-FP / escalate) + adverse-media summary. `AI-SCR-01` **HITL hard-rule**:
never auto-clears a true sanctions hit; a human decides every REF/confirmation. `AI-SCR-02` PII Presidio-masked;
identity-propagated; Langfuse-traced; the rationale audited. `AI-SCR-03` **EU AI-Act tier = limited-risk** (HITL, not
an autonomous decision on a person — classify via `ai-act-gate`); GDPR Art. 22 is the sharper constraint.

## 11. Observability (Operational)
Metrics (screenings/alerts/HOLD-rate/latency), the **dataset-age gauge** (freshness alarm), logs→Loki, OTel traces
(ADR-012), LLMOps Langfuse project for the AI worker. A Grafana panel = HOLD-rate + dataset age + nightly-delta size.

## 12. Deployment (Operational)
- **yente** → gitops wrapper-chart (+ OpenSearch), `YENTE_AUTO_REINDEX` on, **pinned image tag**, manifest → the
  official free lists; the controller/MinIO pattern if a list cache is needed. Default-deny egress netpols.
- **screening module** → part of the ktayl-core deploy (Kargo dev→prod); the `screening` schema via Flyway on boot.
- **nightly re-screen** → a same-image CronJob (`twelve-factor` #12). **AI worker** → a Python Deployment (like the agents).
- Secrets (list tokens if any, yente creds) via **ESO→Vault**.

## 13. Failure modes + rollback (Operational)
- yente down → gate **fails CLOSED** (HOLD), never silent-allow (`NFR-SCR-02`); alert ops.
- reindex fails → staleness alarm fires (`SEC-SCR-04`); screenings continue on last-good but flagged stale.
- AI worker down → alerts queue for manual review (the AI is advisory; the gate still works).
- Bad module release → Kargo canary + rollback (the standard); read-model is CDC-rebuildable.

## 14. Compliance mapping
LCB-FT / CMF + the AMLD/AML package (KYC/KYB · PEP · ongoing vigilance · TRACFIN) · **EU restrictive measures /
gel des avoirs** (sanctions + freeze) · **GDPR** (party PII, Art. 30 register + Art. 22 for the AI) · **EU AI Act**
(the triage tier) · DORA (the audit trail). Applicability caveat: AML is **lighter for pure non-life/IARD** — scope
to the showcased activity. Controls accrue to the RC-05 control library (`Risk→Control→Owner→Evidence→Testing→…`).

Related: **ADR-001** (the decision), `software-evaluation.md`, `architecture-strategy.md` (the ktayl-core module),
`bmad-compliance.md` (regulatory layer + AI-Act gate), `llmops.md` / `ADR-005` (governed AI), the obligations
register (RC-05), ktayl-core ADR-013/015/016, the SPEC `docs/specs/spec-rc02-screening/SPEC.md`.
