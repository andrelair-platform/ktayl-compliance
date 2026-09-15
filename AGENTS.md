# AGENTS.md — ktayl-compliance

Tiny repo-specific context. Org rules (`minicloud-gitops/.claude/rules/*`) still apply.

## Policy
- **ktayl-solution IS** product (board **#15**, Regulatory & Compliance) — GDPR, AML/KYC (LCB-FT),
  sanctions, PIA/AIPD, BCP, ACPR obligations, the compliance function. **NEVER involve Retrieva**
  (separate product; it only *proves* the DORA third-party slice — this repo owns the org obligation map).
- **Insurer ≠ bank** — CRR/CRD/PSD2 do NOT apply. Spine = Solvency II + transversal EU/FR frameworks.
- **The obligations register is the single source of truth** for what applies to ktayl:
  [`docs/obligations-register.md`](docs/obligations-register.md) (delivers epic RC-05). The delivery
  method (`bmad-compliance.md` *The regulatory layer*) points here — don't restate obligations elsewhere.
- Two applicability caveats always: **by activity** (IARD/vie/réassurance differ) and **group-IFRS vs
  local-statutory**.
- Control-library shape (evidence): `Risk→Control→Owner→Evidence→Testing→Finding→Remediation→Audit`.

## Status
Scaffold + obligations register seeded. BMAD per-product (RC-01…RC-05). Controls implemented per
capability as each domain is built (business-tools-first).
