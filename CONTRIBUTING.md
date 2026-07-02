# Contributing to Invest4Talent repositories

This document covers both:

1. **Contributor conventions** (branches, commits, PRs, local dev setup) — applies to anyone opening a PR against any Invest4Talent repository.
2. **Consumer integration** — how a repository wires in the shared CI workflows this repo publishes.

---

## 1. Branch naming

```
<type>/<short-description>
```

- Use the same type vocabulary as commits (below).
- Description: 3–5 words, lowercase, hyphen-separated.

Examples: `feat/user-authentication`, `fix/api-timeout`, `chore/update-dependencies`, `ci/add-deployment-workflow`.

## 2. Commit messages

Follows the [Conventional Commits](https://www.conventionalcommits.org) spec.

```
<type>(<optional scope>): <description>

[optional body — the why, not the what]

[optional footer — Closes #123 / BREAKING CHANGE: ...]
```

Description is **lowercase**, imperative mood ("add login" not "adds login"), no trailing period. `!` after the type/scope marks a breaking change: `feat(api)!: rename user endpoint`.

**Allowed types (do not introduce others):**

| Type | When to use |
| :--- | :--- |
| `feat` | A new feature or capability |
| `fix` | A bug fix |
| `docs` | Documentation only |
| `style` | Formatting, naming, whitespace — no logic change |
| `refactor` | Restructure with no behaviour change |
| `test` | Tests only |
| `chore` | Maintenance: deps, config, tooling |
| `ci` | CI/CD workflow changes |
| `revert` | Reverting a previous commit (include reverted SHA in body) |

When a commit fits two types, prefer the one with higher user impact (a feature that fixes a related bug is `feat`).

## 3. Pull requests

- **Title:** same `type(scope): description` format as a commit — if squash-merging, the title becomes the merged commit
- **Description:** auto-populated from the [org-default PR template](.github/pull_request_template.md) — keep it proportional (a chore PR does not need all four sections; a feature PR does)

Both are checked in CI (see `pr-title / lint` and `commits / lint` below).

## 4. Local dev setup — `.pre-commit-config.yaml`

Every Invest4Talent repository should ship a `.pre-commit-config.yaml`. Same **baseline vs conditional** shape as `ci.yml` (§6). Delete blocks per stack.

```yaml
# .pre-commit-config.yaml
# Canonical Invest4Talent pre-commit config.
# Run `pre-commit install` once per clone. Hooks run on commit.
#
# The rev: values below are placeholders. After copying this file, run
# `pre-commit autoupdate` to pin every hook to its current release, then
# commit the updated file. Do not ship a repo with stale pins.

repos:
  # ─── BASELINE — required for every repo ──────────────────────
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: <TODO: run pre-commit autoupdate>
    hooks:
      - id: end-of-file-fixer
      - id: trailing-whitespace
      - id: check-yaml
      - id: check-added-large-files

  - repo: https://github.com/compilerla/conventional-pre-commit
    rev: <TODO: run pre-commit autoupdate>
    hooks:
      - id: conventional-pre-commit
        stages: [commit-msg]
        args: [--strict, feat, fix, docs, style, refactor, test, chore, ci, revert]

  - repo: https://github.com/gitleaks/gitleaks
    rev: <TODO: run pre-commit autoupdate>
    hooks:
      - id: gitleaks

  # ─── PYTHON — delete if the repo has no Python ───────────────
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: <TODO: run pre-commit autoupdate>
    hooks:
      - id: ruff
      - id: ruff-format

  # ─── OPENTOFU — delete if the repo has no HCL ────────────────
  - repo: https://github.com/tofuutils/pre-commit-opentofu
    rev: <TODO: run pre-commit autoupdate>
    hooks:
      - id: tofu_fmt
```

**Setup steps:**
1. Copy the file into the repo root.
2. Run `pre-commit autoupdate` to resolve the `rev:` fields; commit the result.
3. Each developer runs `pre-commit install` **once per clone** — hooks then run on every commit.

**Deliberately excluded from pre-commit:** `mypy`, `pytest`, `pip-audit`, `bandit` (via ruff `S`), `tofu validate`, `tofu test`, `tflint`, `trivy`. Rationale: too slow for every commit, already covered by ruff, or too repo-specific. CI is the gate for those.

## 5. Python projects — required `pyproject.toml` config

Consumer Python projects **must** include `S` (flake8-bandit) in their ruff `select`. This is what covers Python SAST in CI without a separate bandit step.

```toml
[tool.ruff.lint]
select = ["E", "F", "I", "S", ...]  # S = flake8-bandit

[tool.ruff.lint.per-file-ignores]
"tests/**/*.py" = ["S101"]  # pytest uses `assert`
```

Without `S`, security lint coverage is not present in the shared `python-ci.yml` workflow.

## 6. Consumer `.github/workflows/ci.yml`

One `ci.yml` per repo. **Baseline** block is required for every repo; **Python** and **OpenTofu** blocks are conditional per stack — delete the block if the repo does not have that stack.

```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]

# Callers MUST grant these top-level scopes explicitly — GitHub reusable
# workflows can only narrow the caller's permissions, never expand them.
# `pull-requests: read` is required by pr-title-lint and commit-lint so they
# can inspect the PR context. `contents: read` covers checkout for every job.
permissions:
  contents: read
  pull-requests: read

jobs:
  # ─── BASELINE — required for every repo ──────────────────────
  secrets:
    uses: invest4talent/.github/.github/workflows/secret-scan.yml@v1

  pr-title:
    if: github.event_name == 'pull_request'
    uses: invest4talent/.github/.github/workflows/pr-title-lint.yml@v1

  commits:
    if: github.event_name == 'pull_request'
    uses: invest4talent/.github/.github/workflows/commit-lint.yml@v1

  # ─── PYTHON — delete if the repo has no Python ───────────────
  python:
    uses: invest4talent/.github/.github/workflows/python-ci.yml@v1
    # with:
    #   python-version: "3.13"
    #   working-directory: "backend"

  # ─── OPENTOFU — delete if the repo has no Terraform/OpenTofu ─
  tofu:
    uses: invest4talent/.github/.github/workflows/tofu-module-ci.yml@v1
    # with:
    #   working-directory: "infra"
```

**Pin recommendations:**
- **Default:** `@v1` — floating major, non-breaking updates automatically
- **Regulated / high-security repos:** `@<40-char-sha>` — Dependabot handles both patterns
- **Never** `@main` — no stability contract

## 7. Consumer `.github/dependabot.yml`

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
  # add python, npm, terraform ecosystems as appropriate for the project
```

## 8. Optional: `.gitleaks.toml`

Drop a `.gitleaks.toml` at the repo root to allowlist false-positive matches. Honoured by both the local `scan-secrets.sh` Claude hook and the reusable `secret-scan.yml` workflow — one file covers both.

## 9. Branch protection ruleset (per repo)

Invest4Talent is on a free org plan, so we cannot ship a central ruleset. Every repository owner runs this once per repo.

**Plan constraint (read this first):** GitHub's free plan only allows branch rulesets / protection on **public** repositories. If your repo is public, follow the ruleset below. If your repo is **private and the org is still on the free plan**, the CI checks (`secrets / scan`, `pr-title / lint`, `commits / lint`, and any language-specific ones) still run on every PR — but you cannot mark them as required for merge. Rely on team discipline (do not merge red PRs, use the Conventional Commits git hook, review before merging) until the org upgrades to a paid plan (Team / Enterprise Cloud), at which point rulesets on private repos unlock and the section below applies unchanged. Shared workflow repos like `Invest4Talent/.github` itself are typically public for this reason.

```
Go to repo → Settings → Branches → Add branch ruleset targeting `main`:

 ☑ Require a pull request before merging
   - Require approvals: 1 (adjust to team)
   - Dismiss stale approvals on new commits
 ☑ Require status checks to pass before merging
   - ☑ Require branches to be up to date before merging
   - Add the specific checks by name (only the ones your repo actually runs):
       BASELINE (every repo):
       - secrets / scan
       - pr-title / lint
       - commits / lint
       PYTHON (Python repos):
       - python / lint-python
       - python / typecheck-python
       - python / test-python
       - python / deps-audit
       OPENTOFU (Tofu repos):
       - tofu / validate
 ☑ Require conversation resolution before merging
 ☑ Do not allow bypassing the above settings (or restrict bypass to admins only)
 ☑ Restrict force pushes on main
 ☑ Restrict deletions of main

Optional:
 ☐ Require signed commits — adds friction; only if the team is set up for it
 ☐ Require linear history — pair with squash-merge for a clean main
```

**Check names look doubled** (e.g. `python / lint-python`) because they follow `<caller-job-name> / <reusable-job-name>`. That is stable — safe to grep for and pin against.

## 10. Deliberate divergence from `github-actions-cicd` skill

Consumer repos may pin actions by tag (per skill: `actions/checkout@v4`). **This shared `.github` repo pins actions to SHA** with a `# vX.Y.Z` comment. Rationale: the blast radius of a compromised action tag is org-wide when called from a shared workflow. Consumer repos are individual — tag pins are fine there.
