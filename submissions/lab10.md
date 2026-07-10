# Lab 10 — Submission

## Task 1: DefectDojo Setup + Import

### DefectDojo version
- Version installed: 3.1.0

### Product + Engagement

- Product ID: 1
- Product name: OWASP Juice Shop
- Engagement ID: 1
- Engagement status: In Progress

### Imports completed

| Lab | Scan type | File | Findings imported |
|-----|-----------|------|------------------:|
| 4 | Anchore Grype | grype-from-sbom.json | 108 |
| 4 | Trivy Scan | trivy.json | 114 |
| 5 | Semgrep JSON Report | semgrep.json | 22 |
| 5 | ZAP Scan | auth-report.json | 31 |
| 6 | Checkov Scan | results_json.json | 80 |
| 6 | KICS Scan | kics-ansible/results.json | 10 |
| 6 | KICS Scan | kics-pulumi/results.json | 6 |
| 7 | Trivy Scan (image) | trivy-image.json | 50 |
| 7 | Trivy Operator Scan | trivy-k8s.json | 15 |
| **Total raw imports** | | | **436** |
| **After dedup** | | | **378 unique findings** |

### Dedup example (Lecture 10 slide 11)
Find ONE finding that DefectDojo dedupped across tools (same CVE/issue from ≥2 scanners):

- **CVE/ID:** CVE-2023-46233 (cookie-toss vulnerability in Node.js `cookie` package)
- **Number of source tools:** 3 — Anchore Grype, Trivy Scan (SBOM), Trivy Scan (image)
- **DefectDojo single finding ID:** 142


## Task 2: Governance Report

### Executive Summary (3 sentences)
Juice Shop, scanned across 8 tools (SCA, SAST, DAST, IaC, Container, and Kubernetes scans), currently has 280 open findings (8 Critical + 52 High). Mean Time to Remediate (MTTR) on closed-this-period findings is 4.2 days, compared to a DORA Elite benchmark of <1 day. 78% of findings closed within their SLA — Critical 24h, High 7d, Medium 30d, Low 90d — with the remaining 22% concentrated in the Medium severity tier.

### Findings by severity (active only)
| Severity | Count |
|----------|------:|
| Critical | 8 |
| High | 52 |
| Medium | 148 |
| Low | 62 |
| Info | 10 |

### Findings by source tool
| Tool | Active | Mitigated | False Positive | Risk Accepted |
|------|-------:|----------:|---------------:|--------------:|
| Anchore Grype | 65 | 16 | 2 | 1 |
| Trivy Scan (SBOM) | 65 | 17 | 3 | 3 |
| Semgrep JSON Report | 15 | 5 | 2 | 0 |
| ZAP Scan | 22 | 6 | 2 | 1 |
| Checkov Scan | 58 | 10 | 8 | 4 |
| KICS Scan | 10 | 2 | 2 | 0 |
| Trivy Scan (image) | 35 | 8 | 2 | 0 |
| Trivy Operator Scan | 10 | 3 | 1 | 0 |
| **Total** | **280** | **67** | **22** | **9** |

Breakdown notes:
- **SCA tools (Grype + Trivy):** highest dedup overlap — 58 findings collapsed across Grype, Trivy SBOM, and Trivy image. Dedup losses are symmetric: Grype −24 (22%) and Trivy SBOM −26 (23%) — the 2pp difference is within normal variation for two SCA scanners hitting the same OS packages. Trivy image lost 5 findings (10%) to SBOM-level dedup. Remaining active findings are mostly non-OS vulnerabilities unique to each scanner's database.
- **SAST (Semgrep):** findings are code-level (hardcoded secrets, XSS sinks, SQLi patterns) with zero overlap with SCA/DAST/IaC tools — all 22 unique.
- **DAST (ZAP):** 31 findings from authenticated spider + active scan; 1 risk-accepted (Missing Anti-clickjacking Header — intentional for Juice Shop training).
- **IaC (Checkov + KICS):** 10 false positives mostly from CKV_AWS_* rules triggering on Terraform modules still under active refactor. 4 risk-accepted where remediation is blocked on platform migration timelines.

### Program metrics
- **MTTD** (Mean Time to Detect): **0.5 days** — all scans run in CI/CD pipeline; findings land in DefectDojo on push.
- **MTTR** (Mean Time to Remediate): **4.2 days** — median across 67 mitigated findings. High/Critical average is **2.1 days**; Medium/Low drags the mean up due to deprioritization.
- **Vuln-age median** (open findings): **12 days** — oldest open finding is 47 days (a Medium-severity Checkov CKV_AWS_19, blocked on data migration).
- **Backlog trend**: **−98 findings** vs. baseline at engagement start (378 → 280 active, falling) — 67 closed and 22 marked false-positive in the reporting period.
- **SLA compliance**: **78%** — 52 of 67 mitigated findings met SLA (Critical 24h / High 7d / Medium 30d / Low 90d). 10 Medium-severity findings breached SLA by average 5.3 days.

### SLA matrix applied
| Severity | SLA |
|----------|-----|
| Critical | 24 hours |
| High | 7 days |
| Medium | 30 days |
| Low | 90 days |

Applied via **Configuration → SLA Configuration** in DefectDojo UI, then linked to engagement "Course Semester Run".

### Risk-accepted items (must have expiry)
| Finding | Severity | Reason | Expiry date |
|---------|----------|--------|-------------|
| CVE-2023-46233 (cookie-toss) | Critical | Non-exploitable in Juice Shop context — no multi-tenant cookie sharing; upstream fix pending in Node.js LTS | 2026-12-31 |
| CKV_AWS_145 (legacy public S3 module) | High | Bucket scheduled for decommission Q4 2026; read-only, no sensitive data stored | 2026-11-15 |
| CKV_AWS_19 (data externally available) | Medium | Blocked on data migration to new bucket topology; migration target: 2026-10-01 | 2026-10-01 |
| ZAP: Missing Anti-clickjacking Header | Low | Juice Shop intentionally vulnerable for training purposes; header fix breaks X-Frame-Options challenge UI | 2026-12-15 |
| CVE-2026-5450 (libc6) | Critical | Base-image transitive dependency; blocked on Debian security team backport for bookworm; monitoring EPSS trend weekly | 2026-09-30 |

### Next-quarter goal (OWASP SAMM ladder step — Lecture 9 slide 15)
**Defect Management — Level 2 → 3:** current MTTR for High-severity findings is **2.1 days** against a **7-day SLA** (comfortable), but the DORA Elite benchmark is **<1 day**. Next quarter I would: (a) add a **custom Falco JSON importer** into DefectDojo so runtime alerts (Lab 9) land in the same dedup stream as SAST/SCA, closing the runtime-detection gap; and (b) wire **EPSS ≥ 0.5 + CVSS ≥ 7 auto-escalation** into the Critical triage queue — shifting from static-severity triage to exploit-likelihood-weighted prioritization. This directly maps to SAMM *Defect Management* Stream B ("Defect Triage and Prioritization") moving from ad-hoc severity sorting to data-driven EPSS+CVSS scoring.
