# Regulatory Obligations Register — ktayl-solution IS

> **The authoritative map of every regulatory obligation that applies to ktayl** — one framework per
> block, with owner, systems/data touched, required controls, ACPR/EIOPA reference, applicability
> caveat, and status. It is the reference the [compliance-by-design method](https://andrelair-platform.github.io/minicloud-platform-docs/insurance-platform/enterprise-architecture-blueprint)
> points back to ("you trigger DORA → here are the controls DORA demands") and the first thing an
> **ACPR** inspection asks for. This delivers epic **RC-05 (compliance register + evidence hub)**.

> **Insurer ≠ bank.** ktayl is a **commercial-lines / large-risk IARD** insurer under **Solvency II** +
> the transversal EU/FR frameworks below. **CRR/CRD/PSD2 do NOT apply.** ACPR itself separates
> *banque / assurance / transverse*.

> **Two-layer.** This register is a **ktayl-solution IS** artefact (the org's obligation map). **Retrieva**
> is a *separate product* that *proves* one slice of it (the DORA ICT third-party register) — its DORA
> row **points to** Retrieva; the two are not merged.

## How to read it

**Status:** 🟢 met · 🟡 partial · 🔴 gap · ⚪ N/A (with justification).
**Two applicability caveats run through everything:**
- **By activity** — IARD vs vie vs réassurance differ (e.g. some AML/LCB-FT duties are lighter for pure
  non-life; ktayl is IARD/non-life + inward reinsurance).
- **Group vs statutory** — IFRS 17 / consolidated group reporting vs local French statutory (Code des
  assurances / PCG) are distinct obligations.

## The register

### Prudential & governance (the spine)
| Framework | Key obligations | Owner | Systems / data | Required controls | Ref | Applicability | Status |
|---|---|---|---|---|---|---|---|
| **Solvency II — Pillar 1** | SCR/MCR, technical provisions, asset valuation | Actuarial | UW→pricing→exposure→claims→reserves→reinsurance→Finance (#12/#14) | actuarial models, data quality, reserving | Dir. 2009/138, Code assurances | all | 🔴 gap |
| **Solvency II — Pillar 2** | governance, 4 key functions (Risk/Actuarial/Compliance/Audit), ORSA, internal control, fit&proper | Risk/Compliance | cross-domain | key-function mandates, ORSA process, internal-control loop | ACPR gouvernance | all | 🔴 gap |
| **Solvency II — Pillar 3** | supervisory reporting (QRT/XBRL), SFCR, RSR | Finance/Actuarial | Data/Reporting (#5) | reporting pipeline, disclosure controls | EIOPA/ACPR | all | 🔴 gap |

### Digital resilience & third-party (the IT spine)
| Framework | Key obligations | Owner | Systems / data | Required controls | Ref | Applicability | Status |
|---|---|---|---|---|---|---|---|
| **DORA — ICT risk** | ICT risk mgmt, asset/app inventory, incident mgmt, BCP/DR, resilience testing, security | IT (#3) | all ICT assets | inventory, incident process, DR, pen/resilience testing | Reg. 2022/2554 | all financial entities | 🟡 partial (platform strong; formal ICT-risk framework TBD) |
| **DORA — third-party ICT** | supplier register, criticality, subcontractors, exit/BCP, contract clauses | IT/Compliance | every SaaS/cloud/vendor | **the ICT third-party register → Retrieva**, exit strategies | Art. 28–30 | all | 🟡 partial (**Retrieva** is the product for this) |

### Distribution & customer protection
| Framework | Key obligations | Owner | Systems / data | Required controls | Ref | Applicability | Status |
|---|---|---|---|---|---|---|---|
| **IDD / DDA** | product governance (POG), advice/adequacy, info duties, conflicts, remuneration transparency, competence | Distribution (#13) | CRM, broker portal, quote, product catalogue, comms, complaints | POG checks, disclosures, complaints trail | Dir. 2016/97, Code assurances | distribution | 🔴 gap |
| **Protection clientèle (ACPR)** | information, claims handling, complaints, commercial practices | Compliance/#13 | quote/policy/claims/comms | complaints register, fair-practice controls | ACPR recommandations | all | 🔴 gap |

### Financial crime
| Framework | Key obligations | Owner | Systems / data | Required controls | Ref | Applicability | Status |
|---|---|---|---|---|---|---|---|
| **LCB-FT / AML-CFT + KYC/KYB** | client & beneficial-owner ID/verification, risk assessment, ongoing vigilance, PPE, suspicious ops, TRACFIN | Compliance (#15) | onboarding, submissions, counterparties | KYC/KYB, BO resolution, PPE screening, TRACFIN reporting | CMF, AMLD/AML package, ACPR | **scope varies by activity** (lighter for pure non-life; check per line) | 🔴 gap |
| **Sanctions / gel des avoirs** | screen clients/brokers/BOs/countries/payments vs EU+national lists, freeze | Compliance (#15) | submissions, payments, parties | screening gate before bind + before payment, freeze process | EU restrictive measures, ACPR | all, esp. international | 🔴 gap |

### Data & AI
| Framework | Key obligations | Owner | Systems / data | Required controls | Ref | Applicability | Status |
|---|---|---|---|---|---|---|---|
| **GDPR + Loi I&L** | lawfulness, purpose limitation, minimisation, retention, DSAR, security, DPIA, breach, processors, transfers | DPO/Compliance | every PII system (insureds, claimants, brokers, employees, experts) | ROPA (Art. 30), DPIA, **Presidio** PII masking, retention, breach process | Reg. 2016/679, CNIL | all (insurer = controller for the contract) | 🟡 partial (Presidio live; ROPA/DPIA process TBD) |
| **EU AI Act** | risk tiering, governance, oversight, logging, evaluation, monitoring per AI system | AI (#4)/Compliance | every AI use case | **[AI-Act gate](https://andrelair-platform.github.io/minicloud-platform-docs/ai-ml/ai-act-gate)** (tier → controls), system cards, Langfuse | Reg. 2024/1689 | AI systems (most ktayl UW/pricing = *limited*, not high — B2B/IARD) | 🟡 partial (gate defined; per-use-case cards TBD) |

### Finance, sustainability, international, reinsurance, tax
| Framework | Key obligations | Owner | Systems / data | Required controls | Ref | Applicability | Status |
|---|---|---|---|---|---|---|---|
| **IFRS 17 + FR statutory** | insurance-contract measurement (group IFRS); local statutory accounting | Finance (#14) | policy+premium+claims+reserves+reinsurance | accounting engine, reconciliations | IFRS 17; PCG/Code assurances | **group IFRS vs local statutory — distinct** | 🔴 gap |
| **SFDR / Taxonomy / CSRD** | ESG disclosures, sustainability of investments/reporting | Finance/Risk | investment + reporting data | ESG data, disclosure controls | Reg. 2019/2088, CSRD | **scope-dependent** (size/listing thresholds) | ⚪ likely out-of-scope now (document threshold) |
| **International programs** | local admitted policies, cross-border premium/claims, local compliance + sanctions + tax | Intl (#23) | master + local policies | network controls, local-compliance checks | local + SII group | international business | 🔴 gap |
| **Reinsurance** | cessions, counterparty risk, capital effect, bordereaux, recoveries reporting | Reinsurance (#22) | cessions, treaties | counterparty limits, bordereaux, SII treatment | SII, Code assurances | ceded + inward | 🔴 gap |
| **Insurance taxation** | premium taxes (TCA), international operations, local taxes | Finance (#14) | premium, policy, country | tax calc + reporting | fiscalité assurance | all, esp. international | 🔴 gap |
| **Facturation électronique + e-reporting** (réforme 2026/27) | **receive** structured e-invoices via a PDP (all cos, **1 Sep 2026**); **issue** e-invoices for VAT-taxable B2B (grandes/ETI, 1 Sep 2026; PME/TPE 1 Sep 2027); **e-reporting** of transaction data | Finance (#14) + Compliance | AP/AR, invoices, VAT-taxable services | PDP connectivity, Factur-X (ERPNext `erpnext_facturx`, partial ✅), e-reporting | CGI / DGFiP; réforme facturation électronique | **⚠️ needs Compliance scoping — insurance is VAT-exempt**, so premium invoices are largely outside the *invoice* mandate but **receiving is universal** + e-reporting applies + VAT-taxable services must be issued | 🟡 partial (receiving + PDP + e-reporting = gap; outbound Factur-X partial via ERPNext) |

### Internal control (the evidence backbone)
| Framework | Key obligations | Owner | Systems / data | Required controls | Ref | Applicability | Status |
|---|---|---|---|---|---|---|---|
| **Internal control / compliance function** | compliance plan, control library, testing, findings, remediation, audit | Compliance/Audit (#15) | all | the **control loop** below | SII Pillar 2, ACPR | all | 🔴 gap |

## The control library (internal-control loop)

Every obligation's controls live here and accrue evidence:
```
Risk → Control → Control owner → Evidence → Testing → Finding → Remediation → Audit
```
This is **structurally the same graph as Retrieva** (entities → controls → evidence) — reuse the
pattern, keep the layers separate. Owned by **Regulatory & Compliance (#15)**; RC-05 is its home.

## Not a silo — one capability, many frameworks

A single **AI Underwriting Assistant** simultaneously triggers **AI Act + GDPR + DORA + Solvency II +
IDD + Sanctions/AML + Outsourcing**. The design artefact must list **all** of them and the **combined**
control set — see the worked example in the [underwriting FDE playbook §12](https://github.com/andrelair-platform/ktayl-underwriting/blob/main/docs/fde-underwriting-playbook.md).

## Maintenance

- **This is the single source** — the delivery method (`bmad-compliance.md` *The regulatory layer*)
  points here; don't restate obligations elsewhere.
- **It grows as domains are built** — each capability's regulatory mapping feeds a row/status back.
- Seeded 2026-09-15 (structure + high-priority frameworks accurate; statuses reflect current build).
  Cert evidence: **BC01 (piloter/gouvernance)** + **BC03 (déployer & sécuriser)**.
