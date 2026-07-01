# Shared CI (`.github`) Repo Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship the Invest4Talent org-level `.github` repository with five reusable CI workflows (secret-scan, pr-title-lint, commit-lint, python-ci, tofu-module-ci), org-default community-health files, self-CI, and a `workflow_dispatch` release automation — versioned via floating major tags so consumer repos can pin `@v1` and get non-breaking updates.

**Architecture:** All CI logic lives in reusable workflows called from a thin `ci.yml` in every consumer repo. Foundation MDs (`SECURITY.md`, `CODE_OF_CONDUCT.md`, `CONTRIBUTING.md`) at the org level become defaults for every repo without its own. `v1.0.0` is cut manually as a bootstrap; `release.yml` then takes over via `workflow_dispatch` — validates the version format, creates the immutable tag, moves the `v1` floating tag, and drafts a GitHub Release with auto-generated notes.

**Tech Stack:** GitHub Actions (YAML, `workflow_call`, `workflow_dispatch`), Git, `gh` CLI, `actionlint`, `yamllint`. Tools invoked inside reusable workflows: `opentofu`, `tflint`, `trivy`, `uv`, `ruff`, `mypy`, `pytest`, `pip-audit`, `gitleaks` (CLI), `commitlint`.

## Global Constraints

Every task's requirements implicitly include this section. Values are copied verbatim from the spec.

- **SHA-pin every third-party action** with a trailing `# vX.Y.Z` comment. Never use tag-only pins here (blast radius is org-wide). Dependabot bumps the pins weekly.
- **Every workflow declares an explicit `permissions:` block.** Never rely on the default token scope.
- **Every job declares `timeout-minutes`.** Values from spec: tofu-module-ci 10, python-ci 5 per lint/typecheck, 15 for test, secret-scan 5, pr-title-lint 2, commit-lint 3.
- **Reusable workflows use `on: workflow_call`.** No other triggers.
- **Default runner: `ubuntu-latest`.** Every reusable workflow exposes a `runs-on` input so a consumer can override with a self-hosted runner label.
- **Language workflows** expose `working-directory` as an input; it's applied via `defaults.run.working-directory` at the job level.
- **Python:** `uv sync --locked --dev` (not `--frozen`) — fails the job on stale lockfile.
- **Gitleaks in CI is the CLI**, not `gitleaks/gitleaks-action` — the action requires a `GITLEAKS_LICENSE` secret for organization accounts on private repos; the CLI is unrestricted.
- **Consumer path format:** `invest4talent/.github/.github/workflows/<name>.yml@v1` — doubled `.github` is the repo name + folder.
- **Check-name pattern:** `<caller-job-name> / <reusable-job-name>`. Job names in reusable workflows are contract; changing them is breaking.
- **Versioning:** floating major tag (`v1`) + immutable semver tags (`v1.0.0`). Moving `v1` requires tag force-push to be allowed on `v*`.
- **Free org plan, private repos.** No GHAS, no CodeQL, no native private-repo secret scanning, no org-level rulesets. All controls live in workflows or in per-repo setup docs.
- **All `<TODO: supply real value>` markers must be resolved before v1 is declared done.** Enumerated in spec §10.

## Prerequisites (before Task 6)

Before shipping any reusable workflow, confirm:
- [ ] The **pilot consumer repo** created in Task 5.5 exists and is accessible.
- [ ] `gh` CLI installed and authenticated (`gh auth status` should show a valid token).
- [ ] `pre-commit` installed locally (`pre-commit --version`), so this repo's own hooks work.

**SHA-pinning approach:** For each third-party action referenced in a workflow, resolve the current stable release's commit SHA before writing the file. Recipe:
```bash
# Example for actions/checkout at its latest release:
gh api repos/actions/checkout/releases/latest --jq '.tag_name'
gh api repos/actions/checkout/git/ref/tags/v4.2.2 --jq '.object.sha'
```
Every `uses:` line in this plan is written as `uses: <owner>/<repo>@<SHA> # <tag>`. Replace `<SHA>` and `<tag>` at write time with the fresh values.

---

### Task 1: PR template + issue templates + Dependabot config

**Files:**
- Create: `.github/pull_request_template.md`
- Create: `.github/ISSUE_TEMPLATE/bug_report.yml`
- Create: `.github/ISSUE_TEMPLATE/feature_request.yml`
- Create: `.github/ISSUE_TEMPLATE/config.yml`
- Create: `.github/dependabot.yml`

**Interfaces:**
- Consumes: nothing.
- Produces: org-default templates that GitHub will apply to every org repo without its own. Sets the shape any later CONTRIBUTING.md points at.

- [ ] **Step 1: Write PR template**

Create `.github/pull_request_template.md`:
```markdown
## What

## Why

## How tested

## Notes
```

- [ ] **Step 2: Write bug report template**

Create `.github/ISSUE_TEMPLATE/bug_report.yml`:
```yaml
name: Bug report
description: Report something that is not working as expected
labels: ["bug"]
body:
  - type: textarea
    id: what-happened
    attributes:
      label: What happened?
      description: What did you observe?
    validations:
      required: true
  - type: textarea
    id: expected
    attributes:
      label: What did you expect to happen?
    validations:
      required: true
  - type: textarea
    id: repro
    attributes:
      label: Steps to reproduce
      description: Minimal steps that reliably reproduce the issue.
    validations:
      required: true
  - type: textarea
    id: environment
    attributes:
      label: Environment
      description: OS, versions, anything relevant to the reproduction.
    validations:
      required: false
  - type: dropdown
    id: severity
    attributes:
      label: Severity
      options:
        - Low — cosmetic or workaround exists
        - Medium — degraded functionality
        - High — feature unusable
        - Critical — blocks work
    validations:
      required: true
```

- [ ] **Step 3: Write feature request template**

Create `.github/ISSUE_TEMPLATE/feature_request.yml`:
```yaml
name: Feature request
description: Propose a new capability or improvement
labels: ["enhancement"]
body:
  - type: textarea
    id: problem
    attributes:
      label: What problem are you solving?
    validations:
      required: true
  - type: textarea
    id: proposal
    attributes:
      label: Proposed solution
    validations:
      required: true
  - type: textarea
    id: alternatives
    attributes:
      label: Alternatives considered
    validations:
      required: false
```

- [ ] **Step 4: Write issue template config**

Create `.github/ISSUE_TEMPLATE/config.yml`:
```yaml
blank_issues_enabled: false
contact_links:
  - name: Report a security vulnerability
    url: https://github.com/invest4talent/.github/blob/main/SECURITY.md
    about: Do not open a public issue for security reports. See our security policy.
```

- [ ] **Step 5: Write Dependabot config**

Create `.github/dependabot.yml`:
```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 5
```

- [ ] **Step 6: Commit**

```bash
git add .github/pull_request_template.md .github/ISSUE_TEMPLATE/ .github/dependabot.yml
git commit -m "chore: add org-default PR/issue templates and dependabot config"
```

- [ ] **Step 7: Push and verify visually on GitHub**

Push to `main` (or a feature branch + PR if branch protection is already on from Task 5 — it is not yet at Task 1).
```bash
git push origin main
```
Then in the GitHub UI:
- Open a fresh test issue in this repo — verify the two templates show up in the picker, the blank-issue option is gone, and the SECURITY link appears
- Open a draft PR in this repo — verify the PR body auto-populates with the What/Why/How tested/Notes structure

---

### Task 2: `SECURITY.md` + `CODE_OF_CONDUCT.md`

**Files:**
- Create: `SECURITY.md`
- Create: `CODE_OF_CONDUCT.md`

**Interfaces:**
- Consumes: nothing.
- Produces: org-default community-health documents. GitHub applies them to any org repo without its own.

- [ ] **Step 1: Write `SECURITY.md`**

Create `SECURITY.md`:
```markdown
# Security policy

## Reporting a vulnerability

Report vulnerabilities privately by emailing **<TODO: supply real value — e.g. security@invest4talent.com>**.

Do **not** open a public GitHub issue for security reports.

## Scope

This policy applies to every public and private Invest4Talent repository. If you are unsure whether something is in scope, err on the side of reporting.

## What to expect

- Acknowledgment of your report within **3 business days**
- A status update within **10 business days**
- Coordinated disclosure once a fix is available

We do not currently operate a bug bounty program.
```

- [ ] **Step 2: Write `CODE_OF_CONDUCT.md`**

Create `CODE_OF_CONDUCT.md` using Contributor Covenant v2.1 verbatim. Fetch the current canonical text:
```bash
curl -sSL https://www.contributor-covenant.org/version/2/1/code_of_conduct.md -o CODE_OF_CONDUCT.md
```
Then open the file and replace the placeholder enforcement contact (`[INSERT CONTACT METHOD]` in the source) with `<TODO: supply real value — enforcement contact email>`.

- [ ] **Step 3: Commit**

```bash
git add SECURITY.md CODE_OF_CONDUCT.md
git commit -m "docs: add org-default SECURITY.md and CODE_OF_CONDUCT.md"
```

- [ ] **Step 4: Verify on GitHub**

Push, then visit **Insights → Community Standards** on this repo. Both files should be checked off. On any other org repo that has no security policy of its own, the org-level `SECURITY.md` should now appear when a user opens the Security tab.

---

### Task 3: `CONTRIBUTING.md` (org-wide contributor and consumer guide)

**Files:**
- Create: `CONTRIBUTING.md`

**Interfaces:**
- Consumes: the shape of Task 1's PR template (referenced in section 3), and the workflow names/inputs decided in Tasks 6–10 (referenced in sections 5–7).
- Produces: the single authoritative reference the whole org points at for branches, commits, PRs, pre-commit setup, `ci.yml` template, and per-repo branch protection.

- [ ] **Step 1: Write `CONTRIBUTING.md`**

Create `CONTRIBUTING.md`:

````markdown
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
````

- [ ] **Step 2: Commit**

```bash
git add CONTRIBUTING.md
git commit -m "docs: add CONTRIBUTING.md covering contributor and consumer conventions"
```

- [ ] **Step 3: Verify on GitHub**

Push and visit **Insights → Community Standards** — CONTRIBUTING.md should now be checked. Open a fresh PR in another org repo without a CONTRIBUTING of its own — GitHub should surface a "Contributing guidelines" link pointing back here.

---

### Task 4: `README.md` (this repo) + `profile/README.md` (org landing page)

**Files:**
- Modify: `README.md` (currently a stub — replace)
- Create: `profile/README.md`

**Interfaces:**
- Consumes: names of the workflows shipped in Tasks 6–10, mentioned in the README's "Reusables catalogue" section.
- Produces: two public-facing docs — this repo's README (what it is + how consumers integrate at a glance) and the org's landing page.

- [ ] **Step 1: Rewrite `README.md`**

Overwrite `README.md`:
```markdown
# invest4talent/.github

Org-shared GitHub Actions workflows and community-health defaults for every Invest4Talent repository.

## What is here

- **Reusable workflows** — called via `workflow_call` from every consumer repo's `ci.yml`
- **Org defaults** — `SECURITY.md`, `CODE_OF_CONDUCT.md`, `CONTRIBUTING.md`, PR + issue templates
- **Org landing page** — `profile/README.md` (rendered at [github.com/invest4talent](https://github.com/invest4talent))

## Reusables catalogue

| Workflow | Purpose | Applies to |
| :--- | :--- | :--- |
| `secret-scan.yml` | Full-history gitleaks scan | Every repo |
| `pr-title-lint.yml` | Enforce Conventional Commits on PR title | Every repo (PRs) |
| `commit-lint.yml` | Enforce Conventional Commits on every commit in a PR | Every repo (PRs) |
| `python-ci.yml` | Lint + typecheck + test + deps audit for uv-based Python | Python repos |
| `tofu-module-ci.yml` | Fmt + tflint + validate + test + trivy for OpenTofu modules | Tofu repos |

## Consuming these workflows

See [`CONTRIBUTING.md`](CONTRIBUTING.md) — full runbook including `ci.yml` template, canonical `.pre-commit-config.yaml`, and per-repo branch protection ruleset.

Short version:
```yaml
# consumer-repo/.github/workflows/ci.yml
jobs:
  secrets:
    uses: invest4talent/.github/.github/workflows/secret-scan.yml@v1
  python:  # if the repo has Python
    uses: invest4talent/.github/.github/workflows/python-ci.yml@v1
```

## Versioning

- Pin `@v1` (floating major) — non-breaking updates automatically
- Breaking changes cut a new major (`v2`); consumers opt in

## Security

See [`SECURITY.md`](SECURITY.md) — do not open public issues for vulnerabilities.
```

- [ ] **Step 2: Create `profile/README.md`**

Create `profile/README.md` (renders at `github.com/invest4talent`):
```markdown
# Invest4Talent

<TODO: supply real value — one-sentence mission statement>

## What we build

<TODO: supply real value — short paragraph on products / services>

## Repositories

<TODO: supply real value — list of key public repos with one-line descriptions, or a link to the org's repositories page>

## Get in touch

- Careers: <TODO: supply real value — careers URL>
- Contact: <TODO: supply real value — public contact email or contact form URL>
- Security: see [security policy](https://github.com/invest4talent/.github/blob/main/SECURITY.md)
```

- [ ] **Step 3: Commit**

```bash
git add README.md profile/README.md
git commit -m "docs: replace README stub and add org profile landing page"
```

- [ ] **Step 4: Verify**

Push. Visit `https://github.com/invest4talent` in an incognito browser tab — the profile README should render as the org's landing page. Visit `https://github.com/invest4talent/.github` — the new README should render.

---

### Task 5: Self-CI workflow + enable branch protection on this repo

**Files:**
- Create: `.github/workflows/self-ci.yml`
- (Repo settings change; no file)

**Interfaces:**
- Consumes: nothing.
- Produces: an `actionlint` + `yamllint` gate on every PR to this repo. Blocks broken YAML from reaching `main`. **This is the first workflow this repo runs, and every reusable workflow shipped in Tasks 6–10 must pass it before merging.**

- [ ] **Step 1: Look up SHAs for pinned actions**

Resolve the current stable release SHA for each action used below. Use the recipe from Prerequisites. Record the version+SHA pairs — you will reuse `actions/checkout` in every later workflow, so save them.

Actions needed here:
- `actions/checkout`
- `rhysd/actionlint@<one of its official releases>` (or install the binary via a step)
- `ibiqlik/action-yamllint`

- [ ] **Step 2: Write `self-ci.yml`**

Create `.github/workflows/self-ci.yml`:
```yaml
name: self-ci

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  actionlint:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@<SHA>  # <tag>
      - name: Run actionlint
        uses: rhysd/actionlint@<SHA>  # <tag>

  yamllint:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@<SHA>  # <tag>
      - name: Run yamllint
        uses: ibiqlik/action-yamllint@<SHA>  # <tag>
        with:
          config_data: |
            extends: default
            rules:
              line-length:
                max: 140
              document-start: disable
              truthy:
                check-keys: false   # GitHub Actions uses `on:` which yamllint mis-flags
```

Replace every `<SHA>` and `<tag>` from Step 1.

- [ ] **Step 3: Test locally that the YAML is valid**

```bash
# quick syntax sanity — either of these works if installed:
yamllint .github/workflows/self-ci.yml
actionlint .github/workflows/self-ci.yml
```

- [ ] **Step 4: Commit and push**

```bash
git add .github/workflows/self-ci.yml
git commit -m "ci: add self-ci workflow with actionlint and yamllint"
git push origin main
```

- [ ] **Step 5: Verify the workflow ran and passed**

```bash
gh run list --workflow=self-ci.yml --limit 1
```
Expected: one recent run with status `success`.

- [ ] **Step 6: Prove the gate by breaking YAML in a PR**

```bash
git checkout -b test/verify-self-ci
# intentionally break the file (e.g. remove a colon)
sed -i.bak 's/timeout-minutes: 5/timeout-minutes 5/' .github/workflows/self-ci.yml
git add .github/workflows/self-ci.yml
git commit -m "ci: intentionally break for verification"
git push -u origin test/verify-self-ci
gh pr create --title "chore: verify self-ci gate" --body "temporary verification"
```
Wait for CI, then verify `gh pr checks` shows a failure on `yamllint` (or `actionlint`).

Clean up:
```bash
git checkout main
git branch -D test/verify-self-ci
git push origin --delete test/verify-self-ci
gh pr close --delete-branch <pr-number>  # if PR still open
```

- [ ] **Step 7: Enable branch protection on `main`**

**Operational — do this in the GitHub UI:**

Settings → Branches → **Add branch ruleset** targeting `main`:
- ☑ Require a pull request before merging (Require approvals: 1)
- ☑ Require status checks to pass — add `actionlint` and `yamllint`
- ☑ Require branches to be up to date before merging
- ☑ Require conversation resolution before merging
- ☑ Do not allow bypassing (or restrict to admins)
- ☑ Restrict force pushes on `main`
- ☑ Restrict deletions of `main`

Save the ruleset.

- [ ] **Step 8: Verify branch protection is live**

```bash
gh api repos/invest4talent/.github/rules/branches/main
```
Expected: JSON output listing the rules configured above.

---

### Task 5.5: Bootstrap `invest4talent/ci-pilot`

**Files (in a *separate* repository, not this one):**
- Create GitHub repo: `invest4talent/ci-pilot` (private)
- Create in that repo: `pyproject.toml`, `uv.lock`, `src/ci_pilot/__init__.py`, `src/ci_pilot/main.py`, `tests/test_main.py`, `infra/main.tf`, `infra/main.tftest.hcl`, `.gitignore`, `README.md`

**Interfaces:**
- Consumes: nothing.
- Produces: a purpose-built consumer repository containing minimal-but-realistic Python and OpenTofu content. Tasks 6–10 wire their newly-shipped reusable workflows into this repo and use it to prove each workflow catches its intended failure modes.

- [ ] **Step 1: Create the repo on GitHub**

```bash
gh repo create invest4talent/ci-pilot --private --description "Pilot repo for testing invest4talent/.github reusable workflows"
```

- [ ] **Step 2: Clone locally and set up initial structure**

```bash
cd "$HOME"   # or wherever you keep repos
gh repo clone invest4talent/ci-pilot
cd ci-pilot
```

- [ ] **Step 3: Add `.gitignore`**

```bash
cat > .gitignore <<'EOF'
# Python
__pycache__/
*.py[cod]
.venv/
.pytest_cache/
.ruff_cache/
.mypy_cache/

# OpenTofu
.terraform/
*.tfstate
*.tfstate.*
crash.log
crash.*.log

# OS
.DS_Store
Thumbs.db
EOF
```

- [ ] **Step 4: Add `pyproject.toml`** with the ruff `S` rules per `CONTRIBUTING.md` §5

```bash
cat > pyproject.toml <<'EOF'
[project]
name = "ci-pilot"
version = "0.1.0"
description = "Pilot for testing invest4talent/.github reusable workflows"
requires-python = ">=3.12"

[dependency-groups]
dev = [
  "mypy>=1.10",
  "pytest>=8.0",
  "ruff>=0.6",
  "pip-audit>=2.7",
]

[tool.ruff.lint]
select = ["E", "F", "I", "S"]

[tool.ruff.lint.per-file-ignores]
"tests/**/*.py" = ["S101"]  # pytest uses `assert`

[tool.mypy]
strict = true
files = ["src", "tests"]
EOF
```

- [ ] **Step 5: Add trivial source + test**

```bash
mkdir -p src/ci_pilot tests
cat > src/ci_pilot/__init__.py <<'EOF'
EOF
cat > src/ci_pilot/main.py <<'EOF'
def greet(name: str) -> str:
    return f"Hello, {name}!"
EOF
cat > tests/test_main.py <<'EOF'
from ci_pilot.main import greet


def test_greet_uses_name() -> None:
    assert greet("world") == "Hello, world!"
EOF
```

- [ ] **Step 6: Generate `uv.lock`**

```bash
uv sync --dev
git status  # confirm uv.lock was generated
```

- [ ] **Step 7: Add a minimal OpenTofu module + smoke test**

```bash
mkdir -p infra
cat > infra/main.tf <<'EOF'
terraform {
  required_version = ">= 1.9.0"
  required_providers {
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
  }
}

variable "prefix" {
  type        = string
  description = "Prefix for generated names"
  default     = "ci-pilot"
}

resource "random_pet" "example" {
  prefix = var.prefix
}

output "pet_name" {
  value = random_pet.example.id
}
EOF

cat > infra/main.tftest.hcl <<'EOF'
run "generates_pet_name_with_prefix" {
  command = plan

  assert {
    condition     = startswith(random_pet.example.prefix, "ci-pilot") || var.prefix == "ci-pilot"
    error_message = "Default prefix should be ci-pilot"
  }
}
EOF
```

- [ ] **Step 8: Add a minimal `README.md`**

```bash
cat > README.md <<'EOF'
# ci-pilot

Pilot repository for verifying the reusable workflows in `invest4talent/.github`.

Contains minimal Python (`src/ci_pilot`) and OpenTofu (`infra/`) content so every
reusable workflow can be exercised on a real consumer without setting up a
production project.

Do not deploy this repo. It is CI-only.
EOF
```

- [ ] **Step 9: Sanity-check locally before pushing**

Run each check to confirm the pilot is green *before* CI ever runs:
```bash
uv run ruff check .
uv run ruff format --check .
uv run mypy
uv run pytest -q
uv run pip-audit

# Requires tofu and tflint installed locally; skip if not available
cd infra
tofu fmt -check -recursive
tofu init -backend=false
tofu validate
tofu test
cd ..
```
All commands should exit 0. Fix any failures before proceeding.

- [ ] **Step 10: Initial commit and push**

```bash
git add .
git commit -m "chore: initial pilot content — python module and tofu smoke test"
git push origin main
```

- [ ] **Step 11: Confirm the repo is ready for Task 6**

```bash
gh repo view invest4talent/ci-pilot
```
Expected: private repo, default branch `main`, one commit. The pilot is now the reference consumer for Tasks 6–10.

---

### Task 6: `secret-scan.yml` — reusable workflow

**Files:**
- Create: `.github/workflows/secret-scan.yml`

**Interfaces:**
- Consumes: nothing.
- Produces: reusable workflow named `secret-scan.yml` exposing:
  - Trigger: `on: workflow_call`
  - Inputs: `runs-on` (string, default `"ubuntu-latest"`)
  - Permissions: `contents: read`
  - Job: `scan` — check name from a caller is `<caller-job-name> / scan`

- [ ] **Step 1: Write `secret-scan.yml`**

Create `.github/workflows/secret-scan.yml`:
```yaml
name: secret-scan

on:
  workflow_call:
    inputs:
      runs-on:
        description: Runner label
        type: string
        default: ubuntu-latest

permissions:
  contents: read

jobs:
  scan:
    runs-on: ${{ inputs.runs-on }}
    timeout-minutes: 5
    steps:
      - name: Checkout with full history
        uses: actions/checkout@<SHA>  # <tag>
        with:
          fetch-depth: 0

      - name: Install gitleaks
        run: |
          set -euo pipefail
          GITLEAKS_VERSION="<TODO: latest gitleaks release, e.g. 8.20.1>"
          TARBALL="gitleaks_${GITLEAKS_VERSION}_linux_x64.tar.gz"
          curl -sSL "https://github.com/gitleaks/gitleaks/releases/download/v${GITLEAKS_VERSION}/${TARBALL}" \
            -o /tmp/gitleaks.tar.gz
          tar -xzf /tmp/gitleaks.tar.gz -C /tmp
          sudo mv /tmp/gitleaks /usr/local/bin/gitleaks
          gitleaks version

      - name: Run gitleaks
        run: gitleaks detect --source . --redact --exit-code 2
```

Resolve the `<TODO>` gitleaks version at write time (`gh api repos/gitleaks/gitleaks/releases/latest --jq '.tag_name'` — strip the leading `v`).

- [ ] **Step 2: Commit on a branch and open a PR**

```bash
git checkout -b ci/secret-scan-workflow
git add .github/workflows/secret-scan.yml
git commit -m "ci: add secret-scan reusable workflow"
git push -u origin ci/secret-scan-workflow
gh pr create --title "ci: add secret-scan reusable workflow" --body "Reusable workflow scanning full history via gitleaks CLI. Verified against the pilot repo."
```

- [ ] **Step 3: Wait for self-ci to pass, then merge**

```bash
gh pr checks --watch
gh pr merge --squash --delete-branch
git checkout main && git pull
```

- [ ] **Step 4: Wire the workflow into the pilot repo pinned to `@main`**

In the pilot repo, add `.github/workflows/ci.yml` (or edit if it exists):
```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]
jobs:
  secrets:
    uses: invest4talent/.github/.github/workflows/secret-scan.yml@main
```
Commit + push in the pilot repo on a branch, open a PR there.

- [ ] **Step 5: Verify pass on clean input**

In the pilot repo PR, `gh pr checks` should show `secrets / scan` = success.

- [ ] **Step 6: Verify fail on planted secret**

In the pilot repo test branch, add a fake AWS-style secret to some file:
```bash
echo 'AWS_SECRET_ACCESS_KEY = "AKIAIOSFODNN7EXAMPLE"' > test-secret.txt
git add test-secret.txt
git commit -m "test: plant a fake secret for verification"
git push
```
Wait for CI. `gh pr checks` should show `secrets / scan` = **failure**. Confirm the failing step's log names the file (redacted).

Clean up:
```bash
git revert HEAD
git push
# then close the PR, delete the branch
```

---

### Task 7: `pr-title-lint.yml` — reusable workflow

**Files:**
- Create: `.github/workflows/pr-title-lint.yml`

**Interfaces:**
- Consumes: nothing.
- Produces: reusable workflow `pr-title-lint.yml`:
  - Trigger: `on: workflow_call`
  - Permissions: `pull-requests: read`
  - Job: `lint` — check name pattern `<caller-job-name> / lint`
  - **Event-context caveat:** the underlying action needs `pull_request` event context. Callers must invoke this from `on: pull_request` (or gate with `if: github.event_name == 'pull_request'`).

- [ ] **Step 1: Write `pr-title-lint.yml`**

Create `.github/workflows/pr-title-lint.yml`:
```yaml
name: pr-title-lint

on:
  workflow_call:

permissions:
  pull-requests: read

jobs:
  lint:
    runs-on: ubuntu-latest
    timeout-minutes: 2
    steps:
      - name: Enforce Conventional Commits on PR title
        uses: amannn/action-semantic-pull-request@<SHA>  # <tag>
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          types: |
            feat
            fix
            docs
            style
            refactor
            test
            chore
            ci
            revert
          requireScope: false
          subjectPattern: ^[a-z].+[^.]$
          subjectPatternError: |
            The subject "{subject}" found in the pull request title "{title}"
            doesn't match the configured pattern. Subjects must start with a lowercase
            letter and must not end with a period.
```

- [ ] **Step 2: Commit on a branch, open a PR, wait for self-ci, merge**

```bash
git checkout -b ci/pr-title-lint-workflow
git add .github/workflows/pr-title-lint.yml
git commit -m "ci: add pr-title-lint reusable workflow"
git push -u origin ci/pr-title-lint-workflow
gh pr create --title "ci: add pr-title-lint reusable workflow" --body "Enforces Conventional Commits format on PR titles. Pilot verified."
gh pr checks --watch
gh pr merge --squash --delete-branch
git checkout main && git pull
```

- [ ] **Step 3: Add the job to the pilot repo `ci.yml`**

Append to pilot's `.github/workflows/ci.yml`:
```yaml
  pr-title:
    if: github.event_name == 'pull_request'
    uses: invest4talent/.github/.github/workflows/pr-title-lint.yml@main
```
Push on a branch. Open a PR with a **conventional** title in the pilot (e.g. `chore: verify pr-title lint`).

- [ ] **Step 4: Verify pass on conventional title**

`gh pr checks` should show `pr-title / lint` = success. **This is the event-context propagation check called out in the spec's Phase 3.** If this passes on a real PR, the caveat is confirmed handled.

- [ ] **Step 5: Verify fail on non-conventional title**

Edit the PR title to `bad title without type`:
```bash
gh pr edit --title "bad title without type"
```
Re-run the check (`gh pr checks --watch`). Expected: `pr-title / lint` = failure with a message about the missing type.

Restore the title, close/merge/cleanup as appropriate for the pilot.

---

### Task 8: `commit-lint.yml` — reusable workflow

**Files:**
- Create: `.github/workflows/commit-lint.yml`

**Interfaces:**
- Consumes: nothing.
- Produces: reusable workflow `commit-lint.yml`:
  - Trigger: `on: workflow_call`
  - Permissions: `contents: read`
  - Job: `lint` — check name pattern `<caller-job-name> / lint`
  - **Event-context caveat:** same as `pr-title-lint.yml` — call from `on: pull_request`.

- [ ] **Step 1: Write `commit-lint.yml`**

Create `.github/workflows/commit-lint.yml`:
```yaml
name: commit-lint

on:
  workflow_call:

permissions:
  contents: read

jobs:
  lint:
    runs-on: ubuntu-latest
    timeout-minutes: 3
    steps:
      - uses: actions/checkout@<SHA>  # <tag>
        with:
          fetch-depth: 0
      - name: Lint commit messages
        uses: wagoid/commitlint-github-action@<SHA>  # <tag>
        with:
          configFile: .github/commitlint.config.js
          failOnWarnings: true
```

- [ ] **Step 2: Ship the shared commitlint config**

The action reads `.github/commitlint.config.js` **from the calling repository** (that is where the source is checked out). To avoid every consumer copying a config, embed the config inline instead. Rewrite Step 1's workflow to skip `configFile` and pass rules via `commitlint`'s CLI env:

Replace the `Lint commit messages` step with:
```yaml
      - name: Lint commit messages
        uses: wagoid/commitlint-github-action@<SHA>  # <tag>
        with:
          # Rules embedded here so consumers do not need a config file
          configFile: ""
        env:
          COMMITLINT_CONFIG: |
            {
              "extends": ["@commitlint/config-conventional"],
              "rules": {
                "type-enum": [2, "always",
                  ["feat","fix","docs","style","refactor","test","chore","ci","revert"]],
                "subject-case": [2, "always", "lower-case"],
                "subject-full-stop": [2, "never", "."]
              }
            }
```
**Verify at Step 4 that `wagoid/commitlint-github-action` supports `COMMITLINT_CONFIG` env at the pinned version — check the action's README.** If it does not, fall back to shipping `.github/commitlint.config.js` as a required file in the *shared* repo and pointing `configFile:` at an absolute path relative to `${{ github.workspace }}` — or accept that every consumer ships their own config, and document that in `CONTRIBUTING.md` §4.

- [ ] **Step 3: Commit, PR, wait for self-ci, merge**

```bash
git checkout -b ci/commit-lint-workflow
git add .github/workflows/commit-lint.yml
git commit -m "ci: add commit-lint reusable workflow"
git push -u origin ci/commit-lint-workflow
gh pr create --title "ci: add commit-lint reusable workflow" --body "Enforces Conventional Commits on every commit in a PR. Pilot verified."
gh pr checks --watch
gh pr merge --squash --delete-branch
git checkout main && git pull
```

- [ ] **Step 4: Wire into pilot and verify**

Append to pilot's `ci.yml`:
```yaml
  commits:
    if: github.event_name == 'pull_request'
    uses: invest4talent/.github/.github/workflows/commit-lint.yml@main
```
Open a PR in the pilot with a **conforming** commit — `gh pr checks` shows `commits / lint` = success.

Then push a commit with a bad message (`git commit -m "bad message without type" --allow-empty`) — `commits / lint` should fail.

If the action does not honour `COMMITLINT_CONFIG` env at the pinned version, iterate on Step 2's fallback.

---

### Task 9: `python-ci.yml` — reusable workflow

**Files:**
- Create: `.github/workflows/python-ci.yml`

**Interfaces:**
- Consumes: nothing.
- Produces: reusable workflow `python-ci.yml`:
  - Trigger: `on: workflow_call`
  - Inputs: `python-version` (string, default `"3.12"`), `working-directory` (string, default `.`), `runs-on` (string, default `"ubuntu-latest"`)
  - Permissions: `contents: read`
  - Jobs: `lint-python`, `typecheck-python` (needs lint), `test-python` (needs typecheck), `deps-audit` (parallel to the chain). Check names: `<caller-job-name> / <these job names>`.

- [ ] **Step 1: Write `python-ci.yml`**

Create `.github/workflows/python-ci.yml`:
```yaml
name: python-ci

on:
  workflow_call:
    inputs:
      python-version:
        description: Python version to install
        type: string
        default: "3.12"
      working-directory:
        description: Project root (where pyproject.toml lives)
        type: string
        default: "."
      runs-on:
        description: Runner label
        type: string
        default: ubuntu-latest

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint-python:
    runs-on: ${{ inputs.runs-on }}
    timeout-minutes: 5
    defaults:
      run:
        working-directory: ${{ inputs.working-directory }}
    steps:
      - uses: actions/checkout@<SHA>  # <tag>
      - uses: astral-sh/setup-uv@<SHA>  # <tag>
        with:
          enable-cache: true
      - run: uv python install ${{ inputs.python-version }}
      - run: uv sync --locked --dev
      - run: uv run ruff check .
      - run: uv run ruff format --check .

  typecheck-python:
    needs: lint-python
    runs-on: ${{ inputs.runs-on }}
    timeout-minutes: 5
    defaults:
      run:
        working-directory: ${{ inputs.working-directory }}
    steps:
      - uses: actions/checkout@<SHA>  # <tag>
      - uses: astral-sh/setup-uv@<SHA>  # <tag>
        with:
          enable-cache: true
      - run: uv python install ${{ inputs.python-version }}
      - run: uv sync --locked --dev
      - run: uv run mypy .

  test-python:
    needs: typecheck-python
    runs-on: ${{ inputs.runs-on }}
    timeout-minutes: 15
    defaults:
      run:
        working-directory: ${{ inputs.working-directory }}
    steps:
      - uses: actions/checkout@<SHA>  # <tag>
      - uses: astral-sh/setup-uv@<SHA>  # <tag>
        with:
          enable-cache: true
      - run: uv python install ${{ inputs.python-version }}
      - run: uv sync --locked --dev
      - run: uv run pytest -q

  deps-audit:
    runs-on: ${{ inputs.runs-on }}
    timeout-minutes: 5
    defaults:
      run:
        working-directory: ${{ inputs.working-directory }}
    steps:
      - uses: actions/checkout@<SHA>  # <tag>
      - uses: astral-sh/setup-uv@<SHA>  # <tag>
        with:
          enable-cache: true
      - run: uv python install ${{ inputs.python-version }}
      - run: uv sync --locked --dev
      - run: uv run pip-audit
```

- [ ] **Step 2: Commit, PR, wait for self-ci, merge**

```bash
git checkout -b ci/python-ci-workflow
git add .github/workflows/python-ci.yml
git commit -m "ci: add python-ci reusable workflow with lint, typecheck, test, deps-audit"
git push -u origin ci/python-ci-workflow
gh pr create --title "ci: add python-ci reusable workflow" --body "Multi-job Python CI: lint-python → typecheck-python → test-python, plus parallel deps-audit."
gh pr checks --watch
gh pr merge --squash --delete-branch
git checkout main && git pull
```

- [ ] **Step 3: Wire the workflow into the pilot Python repo**

Append to pilot's `ci.yml`:
```yaml
  python:
    uses: invest4talent/.github/.github/workflows/python-ci.yml@main
```

Ensure pilot's `pyproject.toml` has `S` in ruff select (per `CONTRIBUTING.md` §5) and `uv.lock` is committed.

- [ ] **Step 4: Verify all four jobs pass on green code**

Open a PR in the pilot. `gh pr checks` should show all four:
- `python / lint-python` = success
- `python / typecheck-python` = success
- `python / test-python` = success
- `python / deps-audit` = success

- [ ] **Step 5: Verify each job fails on its own failure mode**

In the pilot branch, introduce four separate failure scenarios (one at a time; revert between) and confirm the right job fails:
1. **Lint failure:** add an unused import → `lint-python` fails
2. **Type error:** add `x: int = "string"` → `typecheck-python` fails
3. **Test failure:** add `def test_fail(): assert False` → `test-python` fails
4. **Stale lock:** manually edit `pyproject.toml` to add a new dep without updating `uv.lock` → any job's `uv sync --locked --dev` fails at that step

Revert or drop the PR after verification.

---

### Task 10: `tofu-module-ci.yml` — reusable workflow

**Files:**
- Create: `.github/workflows/tofu-module-ci.yml`

**Interfaces:**
- Consumes: nothing.
- Produces: reusable workflow `tofu-module-ci.yml`:
  - Trigger: `on: workflow_call`
  - Inputs: `tofu_version` (string, default `"1.9.1"`), `working-directory` (string, default `.`), `runs-on` (string, default `"ubuntu-latest"`)
  - Permissions: `contents: read`
  - Job: `validate` — check name `<caller-job-name> / validate`

- [ ] **Step 1: Write `tofu-module-ci.yml`**

Create `.github/workflows/tofu-module-ci.yml`:
```yaml
name: tofu-module-ci

on:
  workflow_call:
    inputs:
      tofu_version:
        description: OpenTofu version to install
        type: string
        default: "1.9.1"
      working-directory:
        description: Module root
        type: string
        default: "."
      runs-on:
        description: Runner label
        type: string
        default: ubuntu-latest

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  validate:
    runs-on: ${{ inputs.runs-on }}
    timeout-minutes: 10
    defaults:
      run:
        working-directory: ${{ inputs.working-directory }}
    steps:
      - uses: actions/checkout@<SHA>  # <tag>

      - uses: opentofu/setup-opentofu@<SHA>  # <tag>
        with:
          tofu_version: ${{ inputs.tofu_version }}

      - name: Setup TFLint
        uses: terraform-linters/setup-tflint@<SHA>  # <tag>

      - name: Initialize TFLint
        run: tflint --init
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Run TFLint
        run: tflint --recursive

      - name: Check formatting
        run: tofu fmt -check -recursive

      - name: Initialize (no backend)
        run: tofu init -backend=false

      - name: Validate
        run: tofu validate

      - name: Run OpenTofu tests (if any)
        run: |
          if find . -name '*.tftest.hcl' -print -quit | grep -q .; then
            tofu test
          else
            echo "No .tftest.hcl files found; skipping tofu test."
          fi

      - name: Trivy misconfiguration scan
        uses: aquasecurity/trivy-action@<SHA>  # <tag>
        with:
          scan-type: config
          scan-ref: ${{ inputs.working-directory }}
          severity: HIGH,CRITICAL
          exit-code: "1"
```

- [ ] **Step 2: Commit, PR, wait for self-ci, merge**

```bash
git checkout -b ci/tofu-module-ci-workflow
git add .github/workflows/tofu-module-ci.yml
git commit -m "ci: add tofu-module-ci reusable workflow"
git push -u origin ci/tofu-module-ci-workflow
gh pr create --title "ci: add tofu-module-ci reusable workflow" --body "Fmt, tflint, validate, conditional tofu test, trivy config scan."
gh pr checks --watch
gh pr merge --squash --delete-branch
git checkout main && git pull
```

- [ ] **Step 3: Wire the workflow into the pilot Tofu repo**

Append to pilot's `ci.yml`:
```yaml
  tofu:
    uses: invest4talent/.github/.github/workflows/tofu-module-ci.yml@main
    with:
      working-directory: "."   # or the module path if not at root
```

- [ ] **Step 4: Verify pass on green module**

Open a PR in the pilot. `gh pr checks` shows `tofu / validate` = success.

- [ ] **Step 5: Verify each step fails on its own failure mode**

Introduce failures one at a time and confirm the right step fails:
1. **Formatting:** leave a `.tf` file with bad indent → the `Check formatting` step fails
2. **Validation:** introduce `resource "azurerm_x" "y" { unknown_arg = 1 }` → `Validate` fails
3. **TFLint:** unused variable → `Run TFLint` fails
4. **Trivy config:** create a resource with a HIGH-severity misconfig (e.g. an Azure Storage Account with `allow_blob_public_access = true`) → `Trivy misconfiguration scan` fails

Revert or close the pilot PR after verification.

---

### Task 11: Manual `v1.0.0` release (bootstrap)

**Files:**
- (No files — tags + GitHub Release only)

**Interfaces:**
- Consumes: everything on `main` at the moment of cutting the tag.
- Produces: immutable tag `v1.0.0` and floating major tag `v1`, both pointing at the same commit. A GitHub Release drafted from `v1.0.0`. Consumers can now pin `@v1`.

- [ ] **Step 1: Confirm `main` is green**

```bash
git checkout main && git pull
gh run list --branch main --limit 5
```
Expected: the latest self-ci run on main shows `success`.

- [ ] **Step 2: Create the immutable tag**

```bash
git tag -a v1.0.0 -m "release v1.0.0"
git push origin v1.0.0
```

- [ ] **Step 3: Move (create) the floating major tag**

```bash
git tag -f v1 v1.0.0
git push -f origin v1
```

- [ ] **Step 4: Draft the GitHub Release**

```bash
gh release create v1.0.0 \
  --title "v1.0.0" \
  --notes "Initial release of the Invest4Talent shared CI workflows.

Reusables:
- secret-scan.yml
- pr-title-lint.yml
- commit-lint.yml
- python-ci.yml
- tofu-module-ci.yml

Consumers pin \`@v1\` for floating-major updates. See CONTRIBUTING.md."
```

- [ ] **Step 5: Update the pilot repo(s) from `@main` to `@v1` and confirm still green**

In each pilot repo, in `.github/workflows/ci.yml`, replace every `@main` with `@v1` on the `invest4talent/.github/...` uses lines. Open a PR, wait for checks, verify all green.

---

### Task 12: `release.yml` + tag protection ruleset (Phase 6)

**Files:**
- Create: `.github/workflows/release.yml`
- (Repo settings change: tag protection ruleset)

**Interfaces:**
- Consumes: `main` being green and `v1.0.0` already existing (Task 11).
- Produces: automation that, on `workflow_dispatch` with a `vX.Y.Z` version input, creates the immutable tag, moves the matching major tag, and drafts a GitHub Release with auto-generated notes.

- [ ] **Step 1: Write `release.yml`**

Create `.github/workflows/release.yml`:
```yaml
name: release

on:
  workflow_dispatch:
    inputs:
      version:
        description: "Version to release (e.g. v1.2.0)"
        required: true
        type: string

permissions:
  contents: write

jobs:
  release:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@<SHA>  # <tag>
        with:
          fetch-depth: 0

      - name: Validate version format
        run: |
          if ! [[ "${{ inputs.version }}" =~ ^v[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
            echo "Version must match vX.Y.Z (got: ${{ inputs.version }})"
            exit 1
          fi

      - name: Verify version does not already exist
        run: |
          if git rev-parse "${{ inputs.version }}" >/dev/null 2>&1; then
            echo "Tag ${{ inputs.version }} already exists"
            exit 1
          fi

      - name: Configure git identity
        run: |
          git config user.name  "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"

      - name: Create immutable tag
        run: |
          git tag -a "${{ inputs.version }}" -m "release ${{ inputs.version }}"
          git push origin "${{ inputs.version }}"

      - name: Move major tag
        run: |
          VERSION="${{ inputs.version }}"
          MAJOR="${VERSION%%.*}"      # v1.2.0 -> v1
          git tag -f "$MAJOR" "$VERSION"
          git push -f origin "$MAJOR"

      - name: Create GitHub Release
        uses: softprops/action-gh-release@<SHA>  # <tag>
        with:
          tag_name: ${{ inputs.version }}
          generate_release_notes: true
```

- [ ] **Step 2: Commit, PR, wait for self-ci, merge**

```bash
git checkout -b ci/release-workflow
git add .github/workflows/release.yml
git commit -m "ci: add release.yml for automated tagging and github releases"
git push -u origin ci/release-workflow
gh pr create --title "ci: add release.yml" --body "workflow_dispatch entrypoint: validates version, creates immutable tag, moves major tag, drafts GitHub Release."
gh pr checks --watch
gh pr merge --squash --delete-branch
git checkout main && git pull
```

- [ ] **Step 3: Configure the tag protection ruleset**

**Operational — do this in the GitHub UI.** Requirement: `release.yml` must be able to force-move `v1`, `v2`, etc. Manual deletion of any `v*` tag must be blocked for non-admins.

Settings → Rules → Rulesets → **New ruleset** → Target: **tags**:
- Ruleset name: `Protect version tags`
- Target pattern: `v*` (fnmatch)
- Enforcement: Active
- Rules:
  - ☑ Restrict tag deletion
  - ☑ Restrict tag updates
- Bypass list — add whichever of these are available in the UI, in order of preference:
  1. Add **Repository admin** (bypass mode: Always) — this is the simplest path if the workflow can be dispatched by an admin, since the `contents: write` token inherits the caller's context for ruleset bypass purposes in most configurations.
  2. If admin bypass alone does not allow the workflow to force-move the tag (verified in Task 13), fall back to adding a dedicated Personal Access Token (fine-grained, `contents: write`) stored as `RELEASE_TOKEN` secret, and update `release.yml` to authenticate git with that token instead of `GITHUB_TOKEN`.

Save the ruleset. Then run Task 13 — if the `Move major tag` step in the workflow fails with a ruleset violation, apply the fallback above and retry.

- [ ] **Step 4: Verify the ruleset is live**

```bash
gh api repos/invest4talent/.github/rulesets --jq '.[] | select(.name=="Protect version tags")'
```
Expected: JSON showing the ruleset with target=tags, pattern `v*`.

---

### Task 13: Cut `v1.1.0` via `release.yml` (proves automation)

**Files:**
- (No files — tags + GitHub Release only, via the workflow)

**Interfaces:**
- Consumes: `release.yml` shipped in Task 12; tag protection ruleset configured.
- Produces: immutable tag `v1.1.0`, `v1` moved to point at it, a GitHub Release drafted with auto-generated notes. Confirms the release automation works end-to-end.

- [ ] **Step 1: Ensure at least one PR has merged to `main` since `v1.0.0`**

This is needed so `generate_release_notes: true` has something to summarise. If nothing has changed since `v1.0.0`, land a trivial fix or docs PR first (e.g. a typo in CONTRIBUTING.md).

- [ ] **Step 2: Dispatch the release workflow**

```bash
gh workflow run release.yml -f version=v1.1.0
```

- [ ] **Step 3: Watch the run**

```bash
gh run watch $(gh run list --workflow=release.yml --limit 1 --json databaseId --jq '.[0].databaseId')
```
Expected: green run with steps `Validate version format`, `Verify version does not already exist`, `Create immutable tag`, `Move major tag`, `Create GitHub Release` all passing.

- [ ] **Step 4: Verify tags exist and point at the right commit**

```bash
git fetch --tags --force
git rev-parse v1.1.0    # SHA
git rev-parse v1        # should be the same SHA
```

- [ ] **Step 5: Verify the GitHub Release exists with generated notes**

```bash
gh release view v1.1.0
```
Expected: release page with auto-generated notes derived from PR titles between `v1.0.0` and `v1.1.0`.

- [ ] **Step 6: Verify consumers on `@v1` see the update**

Trigger a re-run of the pilot repo's `ci.yml` — since `v1` now points at the new SHA, the reusable workflow contents come from `v1.1.0`. `gh run list` in the pilot should show a fresh green run.

---

### Task 14: Team adoption handoff (operational)

**Files:**
- (No files — human/process work)

**Interfaces:**
- Consumes: `v1` tag live, `release.yml` proven, `CONTRIBUTING.md` published.
- Produces: at least one non-pilot consumer repo migrated to `@v1` with the per-repo branch protection ruleset configured. This is the acceptance evidence for the v1 done criteria in the spec.

- [ ] **Step 1: Announce internally**

Post in whatever channel the team uses (Slack, Teams, email). Content:
- Link to `github.com/invest4talent/.github/blob/main/CONTRIBUTING.md`
- Short summary: "Every Invest4Talent repo now has one place to get CI. Copy the `ci.yml` template, pin `@v1`, configure branch protection per the doc. Ping <TODO: supply real value — owner name/handle> for help wiring in your repo."

- [ ] **Step 2: Pair with one non-pilot repo owner**

Walk through `CONTRIBUTING.md` §§4–9 with them:
1. Add `.pre-commit-config.yaml`, run `pre-commit autoupdate`
2. Add `.github/workflows/ci.yml` with the baseline + relevant conditional blocks
3. Add `.github/dependabot.yml`
4. For Python: confirm `S` in ruff select
5. Open a PR — verify all expected check names appear
6. Configure the branch protection ruleset with those check names

- [ ] **Step 3: Verify acceptance criteria are met**

- [ ] At least one non-pilot consumer repo is running on `@v1`
- [ ] That repo's branch protection ruleset lists the required check names from `CONTRIBUTING.md` §9
- [ ] All `<TODO: supply real value>` markers across the `.github` repo have been resolved (spec §10 — profile README, SECURITY contact, CODE_OF_CONDUCT contact, issue template config)

If any are unmet, v1 is not done. Land the fixes and re-check.

---

## Self-Review

**Spec coverage:**

| Spec section | Task(s) |
| :--- | :--- |
| §1 Purpose | Encoded in `README.md` (Task 4), `CLAUDE.md` (already shipped) |
| §2 Scope | Enumerated across all tasks; non-goals in `CLAUDE.md` |
| §3 Repo layout | Tasks 1–4 (foundation), 5–10 (workflows), 12 (release.yml) |
| §4.1 tofu-module-ci | Task 10 |
| §4.2 python-ci | Task 9 |
| §4.3 secret-scan | Task 6 |
| §4.4 pr-title-lint | Task 7 |
| §4.5 commit-lint | Task 8 |
| §5.1 profile/README.md | Task 4 |
| §5.2 SECURITY.md | Task 2 |
| §5.3 CODE_OF_CONDUCT.md | Task 2 |
| §5.4 CONTRIBUTING.md | Task 3 |
| §5.5 PR template | Task 1 |
| §5.6 issue templates | Task 1 |
| §5.7 dependabot.yml | Task 1 |
| §6.1–6.3 versioning + release | Task 11 (bootstrap), Task 12 (automation), Task 13 (proof) |
| §6.4 branch + tag protection on this repo | Task 5 (branch), Task 12 (tag ruleset) |
| §7 consumer integration pattern | Task 3 (documented), Tasks 6–10 (each tested against pilot) |
| §8 skill alignment | Encoded in workflow contents (Tasks 5–10) and CONTRIBUTING.md (Task 3) |
| §9 rollout phases | Tasks map onto phases (Phase 1 → Tasks 1–4, Phase 2 → Task 5, pilot bootstrap → Task 5.5, Phase 3 → Tasks 6–10, Phase 4 → Task 11, Phase 5 → Task 14, Phase 6 → Tasks 12–13) |
| §10 placeholders | Explicitly checked in Task 14 acceptance criteria |

Every spec requirement maps to at least one task.

**Placeholder scan:** All `<TODO>` markers in the plan are either (a) intentional user-supplied content (SECURITY contact, profile copy — flagged in Task 14 acceptance), or (b) intentional lookup-at-write-time values (action SHAs and versions — recipe in Prerequisites). No "TBD", "implement later", or vague-instruction placeholders.

**Type consistency:**
- `working-directory` input: same name in `python-ci.yml` (Task 9) and `tofu-module-ci.yml` (Task 10) — consistent.
- `runs-on` input: same name across all reusable workflows.
- `tofu_version` input: underscore, matches the spec table and the `opentofu/setup-opentofu` action's expected name.
- Job names: `scan`, `lint`, `lint`, `lint-python` / `typecheck-python` / `test-python` / `deps-audit`, `validate` — consistent with §7.2's branch protection list.
- Check-name pattern `<caller-job-name> / <reusable-job-name>` — stated in Global Constraints and reflected everywhere it matters.

---

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-07-01-shared-ci-github-repo.md`. Two execution options:

**1. Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration.

**2. Inline Execution** — Execute tasks in this session using executing-plans, batch execution with checkpoints.

Which approach?
