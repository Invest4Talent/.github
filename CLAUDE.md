# invest4talent/.github — project instructions

## What this repo is

The Invest4Talent org-level `.github` repository. It hosts:

- **Reusable GitHub Actions workflows** (`workflow_call`) that every Invest4Talent repo calls from a thin `ci.yml` wrapper
- **Community-health defaults** that GitHub applies to every org repo that does not ship its own (`SECURITY.md`, `CODE_OF_CONDUCT.md`, `CONTRIBUTING.md`, PR + issue templates)
- **`profile/README.md`** — the org's public landing page at `github.com/invest4talent`
- **Its own automation** — self-CI for the YAML shipped here, plus a release workflow that cuts semver tags and moves a floating major tag

**Consumers do not clone this repo.** They reference the workflows by path and tag: `uses: invest4talent/.github/.github/workflows/<name>.yml@v1`.

## Stack

- GitHub Actions (YAML workflows, `workflow_call` triggers)
- Third-party actions SHA-pinned with a `# vX.Y.Z` comment; Dependabot bumps weekly
- Tools invoked inside the reusable workflows: `opentofu`, `tflint`, `trivy`, `uv`, `ruff` (including `S` rules), `mypy`, `pytest`, `pip-audit`, `gitleaks` (CLI, not the license-gated action), `commitlint`, `amannn/action-semantic-pull-request`
- Self-CI uses `actionlint` and `yamllint`
- No application code, no runtime dependencies — YAML and Markdown only

## Layout

```
.github/
├── workflows/
│   ├── tofu-module-ci.yml     reusable (workflow_call) — for consumer Tofu repos
│   ├── python-ci.yml          reusable — for consumer Python repos
│   ├── secret-scan.yml        reusable — baseline (every consumer)
│   ├── pr-title-lint.yml      reusable — baseline
│   ├── commit-lint.yml        reusable — baseline
│   ├── self-ci.yml            this repo's own CI (actionlint, yamllint)
│   └── release.yml            this repo's release automation (workflow_dispatch)
├── dependabot.yml             bumps this repo's own action pins
├── pull_request_template.md   org default
└── ISSUE_TEMPLATE/            org default
profile/README.md              org landing page (rendered at github.com/invest4talent)
SECURITY.md, CODE_OF_CONDUCT.md, CONTRIBUTING.md    org defaults
docs/superpowers/specs/        design docs, one per feature
```

**Two placement rules to remember:**
- Reusable workflows *must* be in `.github/workflows/`. That is why the consumer path has the doubled `.github`.
- Community-health MD files sit at repo root here (chose root over `.github/` for discoverability).

## How the pieces connect

- A consumer repo has one `.github/workflows/ci.yml` with a **baseline** block (secrets, pr-title, commits) plus **conditional** blocks per stack (python, tofu). Both baseline and conditional call our reusable workflows.
- The consumer marks the resulting check names as required in a branch protection ruleset. Because Invest4Talent is on a free org plan with private repos, this ruleset is configured per-repo, not centrally — the exact click-through is in `CONTRIBUTING.md`.
- Consumers pin to `@v1` (floating major). SHA-pinning is available for regulated repos. Never `@main`.

## Commands

**Cutting a release** (once `release.yml` is shipped in Phase 6):
- Go to Actions → Release → Run workflow → enter `vX.Y.Z`
- The workflow creates the immutable tag, moves `v1` (or `v2` etc.) to it, and drafts a GitHub Release with auto-generated notes

**Bootstrap release for v1.0.0** (Phase 4 — one-time manual):
```bash
git tag -a v1.0.0 -m "release v1.0.0"
git push origin v1.0.0
git tag -f v1 v1.0.0
git push -f origin v1
# then create a GitHub Release from v1.0.0 with a manual changelog
```

**Local dev on the reusable workflows themselves:**
- `pre-commit install` (once per clone)
- `actionlint` and `yamllint` are the CI gates; run them locally if you want fast feedback

## Key decisions (why, not what)

- **Floating major tag (`v1`) alongside immutable semver tags.** Consumers get non-breaking updates automatically via `@v1`; breaking changes go to `v2`, callers opt in. Moving the major tag is done by `release.yml`; the tag ruleset explicitly allows force-updates on `v*`.
- **All actions SHA-pinned in this repo.** The `github-actions-cicd` skill uses tag pins for normal projects; we diverge because a shared org-level workflow's supply-chain blast radius is org-wide.
- **No GitHub Advanced Security.** Free org plan on private repos. So no CodeQL, no native secret scanning. We compensate with `gitleaks` invoked as a CLI in a workflow (avoids the `gitleaks/gitleaks-action` license gate on org accounts), and with `pip-audit` + `trivy config` scans in the language workflows.
- **No standalone `bandit` step.** Ruff's `S` rules cover it. Consumer `pyproject.toml` must include `S` in ruff `select`. Documented in `CONTRIBUTING.md`.
- **Manual `v1.0.0` cut, then automated releases.** Bootstrap proves the mechanics; `release.yml` (Phase 6) takes over from `v1.1.0` onwards.
- **Baseline vs conditional consumer pattern.** A single `ci.yml` per consumer with clearly delimited blocks. Baseline (secrets/pr-title/commits) is required for every repo; python and tofu blocks are added per stack. Same shape used in the canonical `.pre-commit-config.yaml`.

## Explicit non-goals

- **Deploy workflow** — future work following the `github-actions-cicd` skill's Azure OIDC pattern. Not in v1.
- **Template repos** (`python-service-template`, `tofu-module-template`) — separate projects. Consume the docs this repo produces.
- **`release-please` or other release-PR tooling** — bespoke `workflow_dispatch` in `release.yml` is enough at our release cadence.
- **TypeScript CI** — not currently used at Invest4Talent.
- **Composite actions** — not needed in v1; the reusable workflows cover the ground.

## Where to look

- **The design behind everything here:** `docs/superpowers/specs/2026-07-01-shared-ci-github-repo-design.md`
- **What consumers see and do:** `CONTRIBUTING.md` (once shipped in Phase 1)
- **How Claude should write commits, branches, PRs anywhere:** the user-level `git-conventional-commits` skill
- **GitHub Actions conventions Claude should follow when writing YAML here:** the user-level `github-actions-cicd` skill (mind the SHA-pinning divergence documented above)
