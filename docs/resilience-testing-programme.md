# Digital Operational Resilience Testing Programme (DORA Pillar 3, Art. 24–27)

> **Scope caveat.** ktayl-solution is a *simulated* insurer (`synthetic-enterprise.md`) — this programme
> is **engineering evidence** that the platform's resilience-testing *practices and controls* meet DORA
> Pillar 3, **not** a legal attestation to a competent authority. Results are real; the entity is not.

**Owner:** Security Engineer (SEC), approved by Tech Lead (SA/TL). **Companion docs:**
`minicloud-gitops/.claude/rules/reliability-and-gamedays.md` (Game Day method + SLO register),
`obligations-register.md` (the master DORA obligations), the postmortems in
`minicloud-gitops/docs/reliability/postmortems/`. **Cert evidence:** RNCP39583 Bloc 2 / Bloc 3.

## 1. Scope (what is tested) — Art. 24
Test scope is derived from the **critical/important-functions basis** in
`insurance-platform/capability-map` (**31 Critical · 80 High · 39 Medium**). In-scope ICT systems
supporting critical functions: **ktayl-policy-service (PAS), ktayl-claims, ktayl-underwriting, ktayl-iam,
authentik (SSO), the data layer (CNPG/Postgres)**, and the shared platform capabilities they depend on.

## 2. Methods → controls (what we test with) — Art. 25, verified live 2026-10-17
| Pillar-3 test type | Control / tool (verified running) | Evidence |
|---|---|---|
| Vulnerability assessment & scanning | Trivy-operator (cluster) + Trivy in CI (57 refs); govulncheck on Go | continuous |
| Open-source / SCA analysis | SBOM (163) + syft (56) + cosign-signed (125) + Renovate/Dependabot | continuous |
| Source-code review (SAST) | gosec + checkov + kubeconform in CI | per-PR |
| Network security assessment | **388 NetworkPolicies + 62 CiliumNetworkPolicies** (default-deny) + Hubble + the 2026-08 pentest | posture ✅ |
| Admission + runtime hardening | **Gatekeeper (27 constraints)**, **Falco (8 pods)**, Polaris, kube-bench **CIS 16/16** | continuous |
| Penetration testing | internal pentest 2026-08-16 — all findings **remediated/accepted (closed)** | report |
| Scenario / resilience (chaos, DR) | Chaos Mesh (installed) + Game Day catalog + **executed DR drill (below)** | postmortems |
| Performance testing | k6 in CI + the live L5 QA gate | per-PR |
| Physical security review | **accepted boundary** — BYOD / no-managed-endpoints (`project-governance.md`); not a performed review | documented |

## 3. Cadence (target — demonstrative for a solo-operated lab) — Art. 24
| Class | Cadence | Mechanism |
|---|---|---|
| Vuln / SCA / SAST | continuous | CI + Trivy-operator (in place) |
| Basic-test review (confirm scans green, triage) | quarterly | checklist issue on the compliance board |
| Internal penetration test | annual (next ~2027-08) | scheduled issue |
| **DR restore drill + a chaos Game Day** | quarterly (rotate the catalog) | scenario → run → **postmortem** |
| TLPT | 3-yearly **only if designated significant** (see §6) | N/A today |

## 4. Results log (HONEST — *exercised* evidence only, not "capability exists") — Art. 24
| Date | Test | Target | Result |
|---|---|---|---|
| 2026-08-16 | Penetration test | whole cluster | 1 CRIT + 4 HIGH + MED/LOW — **all remediated/accepted (closed)** |
| **2026-10-17** | **DR restore drill (Game Day #1, scenario 08)** | ktayl-iam / ktayl / retrieva CNPG | **FOUND broken** (`WAL ends before end of online backup`) → **FIXED** (backup + `pg_switch_wal`) → **RE-VERIFIED**: restore **iam 71s · ktayl 53s · retrieva 53s**, full table parity. Postmortem: `2026-10-17-cnpg-dr-restore-wal-gap.md` |
| pending | Chaos Game Day (DB-down / NATS-down) | per catalog | capability live (Chaos Mesh), first run pending |

> Rows marked *pending* are **capability-live-not-yet-exercised** — deliberately NOT counted as passed.

## 5. Findings → remediation → **re-test** loop — Art. 24
Every finding gets: severity → owner → fix (PR) → **re-verify on the converged system** (not "fixed on
paper"). Proven twice: the 2026-08 pentest (re-verified live 2026-10-16) and the DR drill
(found→fixed→re-drilled, same day). The **re-test step is mandatory** — a remediation isn't closed until re-run.

## 6. Independence + Advanced TLPT (Art. 26–27) — stated plainly, not dressed up
- **Tester independence: ABSENT.** Tests are run by the solo operator. The CI gates (no single author can
  bypass Trivy/checkov/CODEOWNERS/cosign) reduce single-author risk but **do not constitute independence**
  in the DORA sense (organisationally separate testers). **Revisit trigger:** engage an external pentester
  before any real-production / real-PII claim.
- **TLPT (TIBER-EU red-team): OUT OF SCOPE BY CONSTRUCTION.** TLPT applies only to entities a competent
  authority *designates significant*. ktayl is a simulated entity with **no competent authority to
  designate it** — so TLPT is not applicable (not a proportionality exemption; simply no real regulator).
  **Revisit trigger:** a real entity designation.

## 7. Known gotcha captured (so it can't silently recur)
**`archive_timeout` does NOT guarantee a restorable CNPG backup for a low-write DB** (tested 2026-10-17:
plain backup still unrestorable after a 6-min wait). A **periodic `pg_switch_wal` is a required DR control**
(reliable design: exec as `postgres` via a scoped CronJob — app-role TCP is unreliable due to CNPG
managed-role password drift). **Action: WAL-switch CronJob per low-write cluster — pending (gitops).**
Detective backstop: the quarterly DR drill (§3).
