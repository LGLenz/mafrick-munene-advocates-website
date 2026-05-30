# Mafrick & Company Advocates — Website & Digital Presence

> Partnership project managed by **Elias Lenz MBA Beratung** under **ELB Consulting Tech**

## Overview

This repository contains all website development and digital presence assets for **Mafrick & Company Advocates**, a Kenya-based law firm and NGO cooperation entity specializing in legal services, civic education, youth and women empowerment, human rights, and data protection.

The project is part of the **Partnership Proposal: Mafrick & Company Advocates & NGO Cooperation** framework, managed under Elias Lenz's freelance consulting umbrella (mini-GmbH).

## Partner

- **Lead:** Mafrick Munene (mafrickmunene@gmail.com)
- **Firm:** Mafrick & Company Advocates
- **Location:** Kenya
- **Specializations:**
  - Legal services & civic education
  - Youth & women empowerment
  - Human rights
  - Data protection training & assessments
- **Partnership Contact:** ELB Consulting Tech (elenz@elbconsultingtech.com)

## Project Scope

- [ ] Law firm website (practice areas, team, contact)
- [ ] NGO cooperation page
- [ ] Data protection services page
- [ ] Civic education resources section
- [ ] Case studies & publications
- [ ] Online consultation booking
- [ ] Funding Application 2026 — Digital Presence Component

## Drive Artifacts

| Document | Type | Description |
|---|---|---|
| Project proposal @mafrick munene and Advocates | Google Doc | Full partnership proposal (accepted) |
| Project proposal @mafrick munene and Advocates.txt | Text | Plain-text version of proposal |
| Funding_Application_Submission_Package_2026 | Google Doc | Complete funding application (May 10) |
| Funding_Application_Submission_Package_2026.docx | Word Doc | Funding application (multiple versions) |
| Milestones and Action Items Yet To Be Done and Status | Google Doc | Active milestone tracker |
| Unedited Complete Chat Mafrick Business Site | Google Doc | Raw session notes — business site planning |
| funding-application-submission-package-2026 | Google Doc | Supporting documentation |

## Legal Partners Network

| Partner | Role |
|---|---|
| Mafrick Munene / Mafrick & Company | NGO cooperation, legal/data services, website dev |
| Tamino Kamau Rosenberger & Family (rep. Nickson Lenz) | Family/individual partner |
| Andreas Lenz | Strategic/legal support |
| Inviolata Kuna Taaban & Family (rep. Jacob Njiru) | Website/digital branding for Beauty Salon |
| Marion Samba & Beth Njiru / TechAssit | Relaunch TechAssit / DigiAssit (immigrants' IT/visa aid in Germany) |

## Status

| Milestone | Status |
|---|---|
| Partnership Agreement | Accepted |
| Proposal Documentation | Complete |
| Funding Application 2026 | Submitted (May 10) |
| GitHub Repository | Initialized |
| Website Design | Pending |
| Website Development | Pending |
| Go-Live | Pending |

## CI/CD — Operating Model

Governance lives in GitHub Actions, branch protection, and environment
approvals — not in assistant memory. See
[`docs/Operating-Model.md`](docs/Operating-Model.md) for the full picture.

Flow:

1. **Copilot/assistant drafts** changes on a feature branch and opens a PR.
2. **PR mandatory checks** run automatically (`.github/workflows/pr-checks.yml`):
   install/tooling validation, build/artifact validation, lint, link/site
   sanity, and no-secrets/config sanity. Branch protection should mark
   these jobs as required for merging to `main`.
3. **Production deployment** (`.github/workflows/deploy-pages.yml`) runs on
   `push:main` and on `workflow_dispatch`, targets the `production`
   GitHub environment (which can require human approval), and records a
   GitHub deployment status for every run.
4. **Post-deploy smoke tests** (`.github/workflows/site-health.yml`) verify
   DNS, TLS, HTTP status, and page title for the GitHub Pages default URL
   and the custom domain. Runs after every deploy, on a 6-hour schedule,
   and on demand. Failures open a tracking issue.
5. **DNS as code**: expected records live in `dns/records.yaml`
   (machine-readable, consumed by `scripts/check_dns.py`). The
   site-health workflow diffs expected vs live records.

### DNS

Custom domain: `mafrick-munene-advocates.com` (CNAME →
`lglenz.github.io.`). The parent zone `elbconsultingtech.com` is
managed under ELB Consulting Tech outside this repository. Both DNS
propagation and the GitHub-issued TLS certificate may take time after
the CNAME is added; the `site-health` workflow treats failures on the
custom domain as **warn-only** until the host is verified live. Once
end-to-end is confirmed, flip the `custom-domain` target in
`site-health.yml` to `required: "true"`.

### Repository structure

```
.
├── index.html              # Single-page site
├── CNAME                   # GitHub Pages custom domain
├── dns/
│   └── records.yaml        # DNS source of truth (CNAME expectation)
├── scripts/
│   ├── check_dns.py        # Diff dns/records.yaml against live DNS
│   └── site_health.sh      # DNS/TLS/HTTP/title smoke probe
├── docs/
│   └── Operating-Model.md  # CI/CD + deployment governance
└── .github/
    └── workflows/
        ├── pr-checks.yml      # Mandatory checks for PRs/pushes
        ├── deploy-pages.yml   # Production GitHub Pages deploy
        └── site-health.yml    # Post-deploy + scheduled smoke tests
```

## Related Projects

- [kuna-beauty-salon-website](https://github.com/LGLenz/kuna-beauty-salon-website) — Digital branding for partner beauty salon
- [digiassist-website](https://github.com/LGLenz/digiassist-website) — IT/visa aid relaunch (TechAssit → DigiAssit)
- [ELB Consulting Tech](https://elbconsultingtech.com) — Parent consulting entity

---
*Managed by ELB Consulting Tech · elenz@elbconsultingtech.com*