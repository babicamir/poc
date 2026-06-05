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

**Trigger:** push to `main`

```
push to main
     │
     ▼
build image
     │
     ▼
push to ECR (tagged :main-<sha>)
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

```
manual trigger (workflow_dispatch)
     │
     ▼
semantic-release on main
→ analyzes all commits since last tag
→ creates GitHub tag + Release (e.g. v1.3.0)
     │
     ▼
re-tag existing ECR image :main-<sha> → :v1.3.0  (no rebuild)
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

- Batches all `feat:` and `fix:` commits since the last release into one version
- Same image that was tested in DEV/QA is promoted — no new build
- If only `chore:` commits exist since last tag, semantic-release exits with no release created

---

## GitHub Environments Setup (Settings → Environments)

| Environment | Workflow | Auto-deploy | Required reviewers |
|---|---|---|---|
| `dev` | dev-pipeline | yes | none |
| `qa` | dev-pipeline | no | 1 reviewer |
| `uat` | release-pipeline | yes | none |
| `prd` | release-pipeline | no | 1 reviewer |

---

## Key Tools / Actions

- [`semantic-release`](https://github.com/semantic-release/semantic-release) — version bumping and changelog generation
- `@semantic-release/github` — creates GitHub Releases and tags
- `@semantic-release/changelog` — updates `CHANGELOG.md`
- `aws-actions/configure-aws-credentials` — OIDC-based AWS auth (no long-lived keys)
- `aws-actions/amazon-ecr-login` — ECR authentication
- GitHub Environments — gate promotions with approval workflows

---

## Secrets / Variables Needed

```
# Per environment (set in GitHub Environment secrets)
AWS_ROLE_ARN          # IAM role ARN for OIDC assumption
AWS_REGION            # e.g. eu-west-1
ECR_REGISTRY          # AWS ECR registry URL

# Repository-level
GITHUB_TOKEN          # auto-provided — used by semantic-release
```

---

## Minimal `release.config.js`

```js
module.exports = {
  branches: ['main'],
  plugins: [
    '@semantic-release/commit-analyzer',
    '@semantic-release/release-notes-generator',
    '@semantic-release/changelog',
    ['@semantic-release/github', { assets: [] }],
    ['@semantic-release/git', { assets: ['CHANGELOG.md', 'package.json'] }],
  ],
};
```

---

## Golden Rules

1. **Never commit directly to `main`** — always via short-lived branch + PR
2. **Build once** — the image deployed to PRD is the same one that ran in DEV and QA
3. **Semantic Release owns the version** — never bump versions manually
4. **Release on a schedule** — run `release-pipeline` manually every 2 weeks, not on every merge
5. **Environment gates live in GitHub Environments** — not in branch protection rules
