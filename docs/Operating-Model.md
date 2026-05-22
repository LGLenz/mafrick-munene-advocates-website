# Operating Model — CI/CD and Deployment Governance

This document is the single source of truth for how changes flow from an
assistant-drafted edit to a verified production deployment of the
**Mafrick & Company Advocates** website
(`mafrick.elbconsultingtech.com`, served via GitHub Pages from
`LGLenz/mafrick-munene-advocates-website`).

The intent is to move governance from "Copilot/assistant remembers to run
the right thing" to **mandatory, codified workflows enforced by GitHub
branch protection, environments, and scheduled smoke tests**.

This is the same governance model used across the LGLenz/ELB Consulting
Tech first-wave repos (e.g.
[`bridgeaxis-consulting` PR #23](https://github.com/LGLenz/bridgeaxis-consulting/pull/23)),
adapted to a single-page static site.

## 1. Flow at a glance

```
  Copilot / assistant drafts change on a branch
                |
                v
        Open Pull Request to main
                |
                v
   ┌────────────────────────────────────┐
   │  PR Mandatory Checks               │  ← required by branch protection
   │  - install-validation              │
   │  - build-validation                │
   │  - lint                            │
   │  - link-sanity                     │
   │  - secrets-config                  │
   └────────────────────────────────────┘
                |
                v
        Reviewer approves & merges
                |
                v
   ┌────────────────────────────────────┐
   │  Deploy GitHub Pages (Production)  │  ← uses `production` environment
   │  - build artifact                  │     (optional reviewers/wait timer)
   │  - actions/deploy-pages            │     records GitHub deployment status
   │  - enforce HTTPS                   │
   └────────────────────────────────────┘
                |
                v
   ┌────────────────────────────────────┐
   │  Site Health (post-deploy)         │  ← triggered by workflow_run
   │  - DNS source-of-truth diff        │     + scheduled every 6h
   │  - HTTPS / TLS / status / title    │     + manual workflow_dispatch
   │  - Pages default URL + custom dom. │
   └────────────────────────────────────┘
```

## 2. Workflow inventory

| Workflow file                          | Trigger                                       | Purpose                                                  |
|----------------------------------------|-----------------------------------------------|----------------------------------------------------------|
| `.github/workflows/pr-checks.yml`      | `pull_request`, `push:main`, `workflow_dispatch` | Mandatory PR checks (jobs listed below).             |
| `.github/workflows/deploy-pages.yml`   | `push:main`, `workflow_dispatch`              | Build + deploy Pages artifact via official actions; records deployment status against the `production` environment. |
| `.github/workflows/site-health.yml`    | `workflow_dispatch`, `schedule` (6h), `workflow_run` (after deploy) | DNS, TLS, HTTP, and title smoke tests of live site. |

## 3. PR Mandatory Checks — required-status-checks list

The following job names from `pr-checks.yml` should be marked **required**
in the `main` branch protection rule:

- `Install / tooling validation`
- `Build / artifact validation`
- `Lint / static validation`
- `Link / site artifact sanity`
- `No-secrets / config sanity`

To configure (Settings → Branches → main):

1. Require a pull request before merging
2. Require status checks to pass before merging
3. Require branches to be up to date before merging
4. Add the job names above to the required-checks list.
5. Require linear history (recommended).
6. Require conversation resolution before merging (recommended).

## 4. Production environment

The deploy workflow targets a GitHub **environment** named `production`.
Configure it under **Settings → Environments → production**:

- **Required reviewers**: add the maintainers who should approve a deploy.
- **Wait timer**: optional, e.g. 5 minutes to allow a final cancel.
- **Deployment branches**: restrict to `main`.

When `deploy-pages.yml` runs, GitHub will:

1. Record a deployment with status `in_progress`.
2. Block on the environment's approval rules (if any).
3. Run the deploy.
4. Update the deployment status to `success` or `failure`.

This eliminates the "deployments tab is stale" problem because every
`push:main` and every manual dispatch produces a tracked deployment.

## 5. Post-deploy site health

`site-health.yml` runs:

- automatically after `deploy-pages.yml` completes (via `workflow_run`),
- on a 6-hour cron schedule,
- on demand via `workflow_dispatch`.

What it checks, for each target URL:

1. **DNS** — `dig` returns at least one A/AAAA/CNAME record.
2. **TLS** — `openssl s_client` completes without verification errors.
3. **HTTP** — `curl` returns a 2xx/3xx status within 15s.
4. **Title** — response body contains a `<title>` mentioning "Mafrick".

Targets:

| Label          | URL                                                                | Required?       |
|----------------|--------------------------------------------------------------------|-----------------|
| pages-default  | https://lglenz.github.io/mafrick-munene-advocates-website/         | no (warn-only)  |
| custom-domain  | https://mafrick.elbconsultingtech.com                              | no (warn-only)  |

Both targets are **warn-only** at the moment because:

- The custom domain `mafrick.elbconsultingtech.com` is in the process of
  being provisioned. The CNAME may still be propagating and GitHub's
  TLS certificate issuance for the host can take 24h+ after the CNAME
  goes live.
- The Pages default URL is a fallback that is only meaningful if Pages
  is configured for workflow-source builds; until that is verified,
  failures should not block the workflow.

When DNS / TLS / HTTPS for the custom domain is verified working, flip
the `custom-domain` target to `required: "true"` in `site-health.yml`
so that regressions become blocking.

When the scheduled run fails, an issue is opened with label `site-health`.

## 6. DNS source of truth

Expected DNS for `mafrick.elbconsultingtech.com` lives in
`dns/records.yaml`. The `dns-check` job in `site-health.yml` consumes
this file and diffs it against live DNS via
`scripts/check_dns.py`.

Expected:

- `mafrick.elbconsultingtech.com` is a CNAME → `lglenz.github.io.`
  (project-Pages with a subdomain custom domain — see the GitHub
  Pages docs).

Note: The parent zone `elbconsultingtech.com` is managed under ELB
Consulting Tech outside this repository. Coordinate any changes to
the CNAME with that zone's owner.

While the custom domain is being provisioned, `dns-check` drift is
**warn-only** in `site-health.yml`. Switch it to a hard failure once
DNS + TLS are confirmed live.

## 7. Operating principles

- **Copilot/assistant drafts; humans approve.** The assistant opens a PR;
  PR checks run automatically; a human reviewer merges.
- **Branch protection is the enforcer, not memory.** No reliance on the
  assistant "remembering" to run checks — they run because GitHub
  requires them.
- **Production deploys use environment approvals.** Manual deploys via
  `workflow_dispatch` go through the same approval gate as automatic
  push-to-main deploys.
- **Smoke tests verify reality.** A green deploy that produces a broken
  site fails the post-deploy `site-health` workflow and opens an issue.
- **DNS is code.** Live DNS is diffed against `dns/records.yaml` on every
  scheduled run.
- **No secrets in PR checks.** All required PR checks are read-only and
  run on forks safely.

## 8. Troubleshooting

| Symptom                                          | Likely cause                                                       | Where to look |
|--------------------------------------------------|--------------------------------------------------------------------|---------------|
| Deployments tab is stale                         | A previous Pages workflow failed without recording a deployment.   | `deploy-pages.yml` run log + Settings → Environments → production. |
| `mafrick.elbconsultingtech.com` shows cert error | GitHub Pages certificate for the custom host still being issued.   | `site-health.yml` → `http-check` (custom-domain target, warn-only). |
| `dns-check` job warns                            | Live DNS doesn't yet match `dns/records.yaml`.                     | Parent zone `elbconsultingtech.com` DNS console + `scripts/check_dns.py` output in job summary. |
| PR check `No-secrets / config sanity` fails      | A secret-shaped string was committed.                              | Job log lists the file and pattern. |

## 9. How to make changes safely

```bash
# 1. Start from main.
git switch main && git pull

# 2. Branch.
git switch -c feat/your-change

# 3. Edit. Commit. Push.
git push -u origin feat/your-change

# 4. Open a PR. Mandatory checks run automatically.
# 5. After approval + merge, deploy-pages.yml runs.
# 6. site-health.yml runs post-deploy and on schedule.
```

## 10. Required follow-up (UI-only, not code)

These cannot be set via committed files and need to be done in the
GitHub UI by a repo admin:

1. **Branch protection on `main`** — add the five `pr-checks.yml` job
   names from §3 as required status checks.
2. **Environment `production`** (Settings → Environments) — required
   reviewers and/or wait timer; restrict deployment branches to `main`.
3. **GitHub Pages source** — Settings → Pages → set "Source" to
   "GitHub Actions" so `deploy-pages.yml` is the source of truth
   (rather than the legacy branch-source build).
4. **Custom domain DNS / TLS** — once
   `mafrick.elbconsultingtech.com` resolves and GitHub has issued a
   certificate, flip the `custom-domain` target in
   `site-health.yml` from `required: "false"` to `required: "true"`.
