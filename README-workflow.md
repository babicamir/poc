# GitHub Workflow — Trunk-Based Development with Semantic Releases

## Overview

Four AWS environments: **DEV → QA → UAT → PRD**  
Branch strategy: **Trunk-based** (single `main` branch, short-lived feature branches)  
Versioning: **Semantic Release** (automated via Conventional Commits, triggered manually)  
Release cadence: **every 2 weeks**

---

## Branch Strategy

```
feature/* ──┐
fix/*      ─┤──► main ──► DEV → QA  (on every merge)
chore/*    ─┘       │
                    └──► UAT → PRD  (manual release trigger, every 2 weeks)
```

- `main` is the trunk — all feature/fix branches merge here via PR
- No long-lived environment branches
- Semantic Release is never triggered automatically — a team member runs it manually when ready to release

---

## Commit Convention (Conventional Commits)

Semantic Release reads all commits since the last tag when the release is manually triggered:

| Prefix | Example | Bump |
|---|---|---|
| `feat:` | `feat: add payment endpoint` | minor (1.**x**.0) |
| `fix:` | `fix: null pointer on login` | patch (1.x.**x**) |
| `feat!:` / `BREAKING CHANGE:` | `feat!: rename user API` | major (**x**.0.0) |
| `chore:`, `docs:`, `ci:` | `chore: update deps` | no release |

---

## Workflow 1 — Development (`dev-pipeline.yml`)

**Trigger:** push to `main` (skips commits containing `[skip ci]`)

```
push to main
     │
     ▼
build image
     │
     ▼
push to ECR  :main-<sha>
     │
     ▼
deploy to DEV  (automatic)
     │
     ▼
manual approval  ◄── GitHub Environment: qa
     │
     ▼
deploy to QA
     │
    end
```

- Runs on every merge — `feat:`, `fix:`, `chore:`, anything
- Image tagged `:main-<sha>` for traceability
- No versioning, no semantic-release

---

## Workflow 2 — Release (`release-pipeline.yml`)

**Trigger:** manual (`workflow_dispatch`) — run every 2 weeks when ready

**Inputs:**
- `image-sha` *(optional)* — short SHA of the image QA approved (e.g. `abc1234`). If omitted, auto-selects the most recently pushed `main-*` image from ECR.

```
manual trigger (workflow_dispatch)
     │
     ▼
semantic-release on main
→ analyzes all commits since last tag
→ creates GitHub tag + Release (e.g. v1.3.0)
→ commits [skip ci] version bump to main
     │
     ▼
re-tag ECR image  :main-<sha> → :v1.3.0  (no rebuild)
     │
     ▼
deploy to UAT  (automatic)
     │
     ▼
manual approval  ◄── GitHub Environment: prd
     │
     ▼
deploy to PRD
     │
    end
```

- Provide `image-sha` to pin exactly the image QA tested — recommended
- Batches all `feat:` and `fix:` commits since last release into one version
- Semantic-release commits back to `main` with `[skip ci]` — does not re-trigger dev-pipeline
- If only `chore:` commits exist since last tag, semantic-release exits with no release created

---

## Workflow 3 — Rollback (`rollback.yml`)

**Trigger:** manual (`workflow_dispatch`)

**Inputs:**
- `image-tag` *(required)* — ECR tag to deploy (e.g. `v1.2.0` or `main-abc1234`)
- `dev` / `qa` / `uat` / `prd` — boolean checkboxes, select one or more environments

```
manual trigger (workflow_dispatch)
  image-tag = v1.2.0
  uat = true
  prd = true
     │
     ▼
verify image tag exists in ECR  (fails fast on typo)
     │
     ├──────────────────────┐
     ▼                      ▼
rollback DEV (if checked)   rollback QA (if checked)
                            manual approval ◄── GitHub Environment: qa

rollback UAT (if checked, runs in parallel with DEV/QA)
     │
     ▼  (waits for UAT to complete or be skipped)
rollback PRD (if checked)
manual approval ◄── GitHub Environment: prd
     │
    end
```

- No semantic-release, no rebuild, no version bump
- ECR image is verified once before any environment is touched
- DEV, QA, UAT roll back in parallel after verify passes
- PRD waits for UAT — if UAT was selected and failed, PRD is blocked; if UAT was skipped, PRD proceeds
- Use versioned tags (`v1.2.0`) to roll back UAT/PRD releases
- Use SHA tags (`main-abc1234`) to roll back DEV/QA to a specific build
- QA and PRD rollbacks require the same manual approval as normal deployments

---

## GitHub Environments Setup (Settings → Environments)

| Environment | Workflow | Auto-deploy | Required reviewers |
|---|---|---|---|
| `dev` | dev-pipeline, rollback | yes | none |
| `qa` | dev-pipeline, rollback | no | 1 reviewer |
| `uat` | release-pipeline, rollback | yes | none |
| `prd` | release-pipeline, rollback | no | 1 reviewer |

---

## Key Tools / Actions

- [`cycjimmy/semantic-release-action`](https://github.com/cycjimmy/semantic-release-action) — runs semantic-release with outputs (`version`, `released`)
- `@semantic-release/changelog` — updates `CHANGELOG.md`
- `@semantic-release/git` — commits version bump back to `main` with `[skip ci]`
- `@semantic-release/github` — creates GitHub Releases and tags
- `aws-actions/configure-aws-credentials` — OIDC-based AWS auth (no long-lived keys)
- `aws-actions/amazon-ecr-login` — ECR authentication

---

## Repository Variables & Secrets

```
# GitHub Actions Variables (non-sensitive, set at repo level)
AWS_REGION            # e.g. eu-west-1
ECR_REPOSITORY        # ECR repository name, e.g. my-app

# GitHub Environment Secrets (set per environment: dev / qa / uat / prd)
AWS_ROLE_ARN          # IAM role ARN for OIDC assumption

# Repository-level (auto-provided)
GITHUB_TOKEN          # used by semantic-release
```

---

## `release.config.js`

```js
module.exports = {
  branches: ['main'],
  plugins: [
    '@semantic-release/commit-analyzer',
    '@semantic-release/release-notes-generator',
    '@semantic-release/changelog',
    ['@semantic-release/github', { assets: [] }],
    ['@semantic-release/git', {
      assets: ['CHANGELOG.md', 'package.json'],
      message: 'chore(release): ${nextRelease.version} [skip ci]\n\n${nextRelease.notes}',
    }],
  ],
};
```

The `[skip ci]` in the commit message prevents the version bump commit from re-triggering `dev-pipeline`.

---

## Golden Rules

1. **Never commit directly to `main`** — always via short-lived branch + PR
2. **Build once** — the image deployed to PRD is the same one that ran in DEV and QA
3. **Semantic Release owns the version** — never bump versions manually
4. **Release on a schedule** — run `release-pipeline` manually every 2 weeks, not on every merge
5. **Optionally provide `image-sha` when releasing** — omitting it auto-selects the latest ECR image; provide it explicitly to pin the exact image QA signed off on
6. **Environment gates live in GitHub Environments** — not in branch protection rules
