# ADR-001 — AML/KYC/sanctions screening: build approach (adopt the engine, build the gate) — RC-02

**Status:** Proposed / DRAFT for review (no build until validated — `AGENTS.md`) · **Owner:** SA/TL + SEC ·
**Epic:** RC-02 (AML/KYC (LCB-FT) compliance, P1) · **Capability owner:** Regulatory & Compliance (#15).
This ADR applies the IS **software-evaluation chain** (`software-evaluation.md`) to RC-02's screening engine
and places the custom gate in `ktayl-core`. **Evaluation record:** this ADR + the filled scorecard live in
**this repo** (`ktayl-compliance/docs/`) — **deliberately NOT routed through Retrieva** (owner decision).

## 1. Need-first — the named gap
The obligations register (`docs/obligations-register.md`) carries two 🔴 **gap** rows this closes:
- **LCB-FT / AML-CFT + KYC/KYB** — client & beneficial-owner ID/verification, ongoing vigilance, **PPE/PEP**,
  suspicious ops (TRACFIN).
- **Sanctions / gel des avoirs** — *"screen clients/brokers/BOs/countries/payments vs EU+national lists, freeze"*
  with a **"screening gate before bind + before payment."**

RC-02 scope = KYC onboarding checks (identity/PEP/sanctions) · ongoing monitoring + risk scoring · TRACFIN SAR ·
audit trail. **Synthetic-enterprise caveat:** ktayl is a fictional insurer — this is *built to demonstrate the
capability*, not a live legal obligation; the register also notes AML scope is lighter for pure non-life/IARD.

## 2. Adopt-vs-build (`architecture-strategy.md`) — split the problem
A screening system is three parts with different build/adopt answers:

| Part | Decision | Why |
|---|---|---|
| **Matching ENGINE + watchlist data** (fuzzy name/DOB match vs sanctions/PEP lists) | **ADOPT** | a commodity + a hard, maintained dataset — never hand-roll the matching rigour or the list ingestion |
| **The screening GATE + alert/case workflow + audit + risk scoring** | **BUILD** | the differentiated, insurer-specific glue (where in the lifecycle, what blocks a bind/payment, the decision model, the audit an examiner accepts) |
| **The AI triage layer** (false-positive reduction, adverse-media summarisation) | **BUILD** (governed) | the value-add on top; reuses the platform AI + governance rails |

## 3. Shortlist + 4. Weighted score (sovereign-first scorecard, template `software-evaluation-scorecard.md`)
Three realistic engine options, scored on the IS criteria (self-hosted/sovereign · FOSS/licence · golden-path
fit · observability · DORA-third-party + data-class · exit · TCO):

| Criterion (weight) | **yente + OpenSanctions** (self-host FOSS) | Commercial SaaS (APScan / World-Check / ComplyAdvantage) | Pure DIY (OpenSearch + Splink) |
|---|---|---|---|
| Need-fit | ✅ does the core job (POST entity → scored matches) | ✅ full-featured | 🟡 build the matching yourself |
| **Self-hostable / sovereignty** | ✅ Docker, data stays on-cluster | ❌ vendor SaaS, data leaves | ✅ |
| **FOSS / licence** | ✅ engine MIT · ⚠️ *consolidated data CC-BY-NC* (mitigation §8) | ❌ proprietary + per-seat/volume | ✅ |
| **SSO-able** (Authentik) | ✅ (gate enforces; engine internal) | 🟡 vendor SSO | ✅ |
| **Golden-path fit** (Vault/ESO · GitOps · netpol) | ✅ wrapper-chart deployable | ❌ off-platform | ✅ |
| Observability hooks | ✅ metadata/version endpoints → staleness monitor | 🟡 vendor dashboards | ✅ |
| **DORA third-party-ICT + data-class** | ✅ no 3rd-party-ICT; PII stays P2/P3 in-house | ❌ a material ICT third party + PII egress | ✅ |
| Exit / portability | ✅ FtM standard, swap lists | ❌ high lock-in | ✅ |
| Operational **TCO** (solo) | 🟡 run yente + OpenSearch + list ingestion | ❌ € + integration | ❌ you own matching quality forever |
| **Verdict** | **WINNER** | rejected (sovereignty/licence/TCO + a new material ICT dependency) | runner-up (more build for no engine gain) |

## 5. Decision
1. **Adopt `yente` (OpenSanctions' open matching API) as the screening engine**, self-hosted on the cluster
   (wrapper-chart + the `.Files.Get` shim where needed; OpenSearch backing index). It does the fuzzy
   name/DOB/nationality matching vs the watchlists.
2. **Data = the official free lists first** (OFAC · EU consolidated · UN · UK OFSI — free for any use) ingested
   via yente's manifest; the OpenSanctions *consolidated* dataset only under its non-commercial terms, which is
   **fine for synthetic ktayl** (see §8). Sanctions coverage is excellent free; PEP (Wikidata) + adverse-media
   (GDELT) are weaker — accepted for the demo, with the buy-path noted for any real-production claim.
3. **Build a `screening` module in `ktayl-core`** (the gate + workflow + audit) — see §6.
4. **AI triage is a governed, human-in-the-loop fast-follow** (§7), not v1-blocking.

## 6. Where the gate sits — a `screening` module in ktayl-core (CLOSED Spring Modulith, hexagonal)
Sibling to `policy`/`billing`/`complaints`. It owns the business logic; yente is just an outbound adapter.

```
ktayl-core/src/main/java/com/ktayl/core/screening/
  domain/         Party, ScreeningResult (hit buckets: SANCTION/PEP/AME/RISK_COUNTRY/UBO/ORIAS),
                  Alert, Case (ACC/REF/PENDING + risk WEAK/MED/STRONG + vigilance NORMAL/ENHANCED),
                  ScreeningEngine port, ScreeningAuditLog port (DORA/LCB-FT append-only trail)
  application/    ScreenParty use case, AlertDecision use case (server-side authority — a REF is a control)
  api/            @NamedInterface ScreeningGate — screen(party) → ALLOW | HOLD(alert)  (in-process)
  infrastructure/ web: POST /v1/screening/screen + the alert/case endpoints
                  persistence: screening schema (schema-per-module, ADR-006)
                  engine: YenteClient (HTTP → the yente match API)
```

**Consumed at the register's gates — in-process AND cross-service (same pattern as `policy`):**
- **Submission/intake** (Submission Hub #27) → screen at intake.
- **Bind** — `ktayl-core` policy/underwriting path: the policy module calls `ScreeningGate` **in-process**;
  `ktayl-underwriting` (separate Python service) calls the **REST** `/v1/screening/screen` before `POST /v1/policies`
  (reuse the multi-issuer M2M auth, ADR-016). A **HOLD blocks the bind** until a human clears the alert.
- **Payment** — `ktayl-core` billing calls `ScreeningGate` in-process **before** a Stripe collection + runs the
  **asset-freeze** path on a confirmed sanctions hit.
- **Nightly re-screen** — a CronJob re-screens the book (watchlists change daily) → a delta of new alerts.

**Events:** lifecycle/alert events on NATS → downstream (compliance dashboards, the AI triage worker).

## 7. The AI triage layer (governed — reuses the platform AI + the `ADR-005`/`llmops.md` rails)
A Python worker consumes alert events → an LLM reasons over the matched record vs the watchlist entry
(DOB/nationality/ID corroboration) → emits a **recommendation + written rationale** (clear-FP / escalate) +
an adverse-media summary. **HARD rule: human-in-the-loop for every REF / sanctions confirmation** — the AI
never auto-clears a true sanctions hit (a wrongly auto-cleared match is a serious breach). PII **Presidio-masked
before any LLM**; identity-propagated; **Langfuse-traced**; the rationale is logged with the decision (audit).
**EU AI-Act tier:** *limited-risk* given mandatory HITL (not an autonomous decision on a person) — classify via
the `ai-act-gate`; GDPR Art. 22 is the sharper constraint.

## 8. Risks / caveats (honest, for the security review)
- **Licence:** OpenSanctions *consolidated* data is **CC-BY-NC** — non-commercial. Fine for synthetic ktayl
  (disclaim it); for a real-production claim, point the manifest at the **official lists** (free for any use)
  and/or buy the data licence. Decided path for ktayl: **official free lists**, so the licence is clean even in
  a "production-shaped" demo.
- **Freshness is a compliance control, not hygiene:** a screener on stale lists silently passes newly-sanctioned
  parties. Leave yente auto-reindex on, **pin the image tag**, and add a **staleness monitor** (alert if the
  loaded sanctions dataset age > 24–48h) + **log the data-version with every screening** (the examiner's question:
  "how do you know the lists were current when you screened?").
- **PII trust boundary:** the party data (name/DOB/ID/nationality) is **P2/P3** — the screening module + yente
  sit behind default-deny egress (only the module may reach yente; yente may reach the list sources); SSO-gated
  APIs; no PII to any LLM unmasked.
- **Authority server-side:** a REF / block-bind / block-payment is a **control enforced in the module**, never
  only client-side.

## 9. Consequences
- **Positive:** closes two 🔴 register gaps with a sovereign, self-hosted, licence-clean engine + an
  insurer-specific gate woven into bind + payment + onboarding; a credible compliance-engineering showcase; the
  AI triage is the differentiator the commercial tools don't give you. No new material ICT third-party dependency.
- **Negative / accepted:** we own the gate + the list ingestion + the freshness monitoring; free PEP/adverse-media
  data is weaker than paid (accepted for the demo; buy-path documented). A synthetic-enterprise disclaimer applies.
- **Next (post-approval):** `/bmad-architecture` + `/bmad-spec` for RC-02; the `screening` module skeleton + the
  yente deployment (gitops) + the gate at the policy/billing/underwriting bind-and-pay points; then the AI worker.

Related: `software-evaluation.md` (the chain), `architecture-strategy.md` (adopt-vs-build + the ktayl-core module),
`bmad-compliance.md` (the regulatory layer + AI-Act gate), `ai-act-gate` / `ADR-005` governed-AI + `llmops.md`,
the obligations register (RC-05), ktayl-core ADR-015/016 (the policy module + M2M auth the gate reuses).
