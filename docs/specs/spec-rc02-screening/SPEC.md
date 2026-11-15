# SPEC — AML/KYC/sanctions screening (RC-02)

**Status:** DRAFT for review (no build until validated) · **Epic:** RC-02 (P1) · **Owner:** TL + SEC ·
**Decision:** ADR-001 · **Spine:** `../../architecture/rc02-screening-architecture.md`. The compact
implementation contract: the requirement set (observable, ID'd), the thin-slice plan, and the gates. Build
target = a `screening` module in **ktayl-core** + the **yente** engine (gitops) + a governed AI worker.
**Synthetic-enterprise:** built to demonstrate the capability; AML scope is lighter for pure non-life/IARD.

## Requirement set (every requirement is observable — a testable AC, not "must be good")
**Functional (FR-SCR):**
- `01` screen a party → a `ScreeningResult` with hit buckets {SANCTION,PEP,AME,RISK_COUNTRY,UBO,ORIAS}.
- `02` a hit raises an `Alert`; `03` an alert resolves to a `Case` (ACC/REF/PENDING + risk + vigilance).
- `04` **REF/HOLD/freeze authority enforced server-side** (not client-only). `05` idempotent screen by (party,gate,day).
- `06` every screening writes an **append-only audit row** with actor + **dataset_version** + verdict.
- `10` intake gate · `11` **bind gate — a HOLD blocks the bind** · `12` payment gate · `13` asset-freeze on a confirmed sanctions hit · `14` nightly re-screen produces a delta of new alerts.
- `20` the `screening` schema (Flyway, schema-per-module). `30` the REST + in-process gate API (`openapi.yaml`).

**Non-functional (NFR-SCR):** `01` gate p95 < 2s · `02` **fail-CLOSED** (gate outage → HOLD, never silent-allow) ·
`03` sanctions dataset age < 24h + **staleness alarm** + data-version logged per screen · `04` audit ≥ 5y ·
`05` nightly re-screen within the window · `06` fits ns quota + budgeted LLM tokens.

**Security (SEC-SCR):** `01` every API SSO/M2M (ADR-016); alert-decision group-gated; authority server-side ·
`02` **no unmasked PII to any LLM** (Presidio); P2/P3 stays in-house · `03` default-deny egress (module→yente,
yente→lists only) · `04` freshness-as-control (failed reindex → alarm, not silent-stale) · `05` append-only audit integrity.

**AI (AI-SCR):** `01` **HITL hard-rule — never auto-clears a true hit** · `02` PII-masked + Langfuse-traced +
rationale audited · `03` EU AI-Act **limited-risk** tier (classify via `ai-act-gate`; GDPR Art. 22 the sharper check).

## Thin-slice plan (Property, one flow first — mirrors the claims thin-slice discipline)
> Each slice ships behind the full gate set (`testing.md` tier-A: L0–L4 + the live QA gate L5) and the governance
> + security review (`bmad-compliance.md`) — this is a boundary-crossing, PII + sanctions + AI capability.

| Slice | Scope | Key ACs | DoD gate |
|---|---|---|---|
| **S0 — engine** | yente deployed (gitops wrapper-chart + OpenSearch); official free lists ingesting; **freshness monitor** | a known sanctioned name returns a scored match live; dataset-age gauge + alarm green; `NFR-SCR-03`/`SEC-SCR-04` | engine live on dev; staleness alarm proven |
| **S1 — gate core** | `screening` module skeleton + `ScreenParty` + the `screening` schema + append-only audit | `FR-SCR-01/02/06`, `20`; screen→ALLOW/HOLD persists + audits with dataset_version; L2 real-PG + L3 contract | module live on dev; QA gate clean (4xx-never-5xx, 401/403) |
| **S2 — bind gate** | wire `FR-SCR-11` — policy in-process + underwriting via REST+M2M; a HOLD **blocks** the bind | a live UW bind on a HIT is HELD; on clean, proceeds; `SEC-SCR-01` authority server-side | end-to-end bind-with-screening on dev |
| **S3 — case workflow** | `FR-SCR-03`, `30` decision API + a group-gated review surface (Homer/Backstage or a thin UI) | analyst accepts/refuses with risk+vigilance+rationale → audited; group-gated (401/403) | human-decision loop live |
| **S4 — payment gate + freeze** | `FR-SCR-12/13` in billing | a payment to a sanctioned party is blocked + freeze-flagged | billing gate live |
| **S5 — nightly re-screen** | `FR-SCR-14` same-image CronJob → delta | watchlist change → new alert next run; idempotent | CronJob live |
| **S6 — governed AI triage** | the AI worker (`AI-SCR-*`) | FP recommendation + rationale on an alert; **no auto-clear of a true hit**; Langfuse-traced; AI-Act tier recorded | AI advisory live, HITL enforced |

## Acceptance gates (before any slice merges to prod)
- **Governance gate** (ADR + the architecture spine review) + **Security review** (PII flow, egress, authority, the
  freshness control, the AI tier) — mandatory, this is boundary-crossing (`bmad-compliance.md`).
- **CI** L0–L4 (tier-A) + the **live QA gate** (L5) against dev — green CI is not sufficient.
- **AI-Act gate** cleared for S6 (`ai-act-gate`); **data classification** (party = P2/P3) recorded.
- Controls accrue to the **RC-05 control library** (`Risk→Control→Owner→Evidence→Testing→Finding→Remediation→Audit`).

## Out of scope / deliberate
- **Transaction monitoring** (the APScan "AP Monitoring" sibling — bank-shaped) — not an insurer need v1.
- **Paid PEP / adverse-media data** — free sources (Wikidata/GDELT) are weaker; accepted for the demo, buy-path noted (ADR-001).
- **The OpenSanctions consolidated (CC-BY-NC) dataset in a commercial claim** — ktayl uses the **official free lists** so the licence is clean.
- **Autonomous AI decisioning** — never; HITL is a hard requirement.
- **Full KYC document capture/IDV** (passport OCR etc.) — a later slice (reuses the doc-extraction pattern), not v1.
