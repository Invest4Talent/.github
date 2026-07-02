---
title: Invest4Talent shared CI (.github) repo — design
date: 2026-07-01
status: draft
---

# Invest4Talent shared CI (`.github`) repo — design

## 1. Purpose

Provide a single, versioned home for CI logic and community-health defaults that every Invest4Talent repository consumes. Consumers wire in one thin `ci.yml` that calls the reusable workflows here; changes to CI logic ship centrally and propagate on the next run without touching each repo.

The organisation runs on a **free GitHub plan with private repos**. That constraint shapes several decisions: no GitHub Advanced Security (no CodeQL, no native secret scanning on private repos), no org-level branch rulesets. This spec compensates by putting equivalent controls into workflows and per-repo setup docs.

## 2. Scope

**In v1:**
- Reusable workflows for OpenTofu module CI and Python CI
- Reusable workflows for baseline PR gates: secret scanning, PR title lint, commit message lint
- Foundation files: org-default community health (`SECURITY.md`, `CODE_OF_CONDUCT.md`, `CONTRIBUTING.md`), PR template, issue templates, org landing page (`profile/README.md`), Dependabot for this repo
- Self-CI for this repo (`self-ci.yml`) — `actionlint` + `yamllint`
- Versioning contract: floating major tag (`v1`) alongside immutable semver tags (`v1.0.0`, …)
- Manual first release, then automated release workflow (`release.yml`)
- Consumer integration docs in `CONTRIBUTING.md`: single `ci.yml` pattern, canonical `.pre-commit-config.yaml`, per-repo branch protection ruleset

**Non-goals in v1 (explicit):**
- Reusable deploy workflow (Azure Container Apps OIDC pattern — future work per `github-actions-cicd` skill)
- Template repos (`python-service-template`, `tofu-module-template` — separate future projects)
- `release-please` or other release-PR tooling (bespoke `workflow_dispatch` is enough)
- TypeScript CI (not currently used at Invest4Talent)

## 3. Repo layout

```
invest4talent/.github/                       (repository)
├── .github/
│   ├── workflows/
│   │   ├── tofu-module-ci.yml               reusable (workflow_call)
│   │   ├── python-ci.yml                    reusable
│   │   ├── secret-scan.yml                  reusable
│   │   ├── pr-title-lint.yml                reusable
│   │   ├── commit-lint.yml                  reusable
│   │   ├── self-ci.yml                      this repo's own CI (actionlint, yamllint)
│   │   └── release.yml                      this repo's release automation
│   ├── dependabot.yml                       bumps this repo's own action pins
│   ├── pull_request_template.md             org default (also applies here)
│   └── ISSUE_TEMPLATE/
│       ├── bug_report.yml
│       ├── feature_request.yml
│       └── config.yml
├── profile/
│   └── README.md                            org landing page at github.com/invest4talent
├── SECURITY.md                              org default security policy
├── CODE_OF_CONDUCT.md                       org default
├── CONTRIBUTING.md                          org default — contributor + consumer guide
├── CLAUDE.md                                instructions for Claude sessions in this repo
├── README.md                                what this repo is, how to consume it
└── docs/
    └── superpowers/
        └── specs/
            └── 2026-07-01-shared-ci-github-repo-design.md
```

**Placement rules that aren't obvious:**
- Reusable workflows must live in `.github/workflows/` (GitHub's fixed convention). The consumer path is `invest4talent/.github/.github/workflows/xxx.yml@v1` — the doubled `.github` is the repo name followed by the folder.
- Org-default community-health files (`SECURITY.md`, `CODE_OF_CONDUCT.md`, `CONTRIBUTING.md`) can sit at repo root or under `.github/`. Placed at root here for discoverability — they are the org's front-facing policy docs.
- PR template and issue templates go under `.github/` as per GitHub convention.

## 4. Reusable workflow contracts

All actions are **SHA-pinned** with a `# vX.Y.Z` comment. Dependabot bumps them weekly. Every workflow declares an explicit `permissions:` block (least privilege). All workflows target GitHub-hosted `ubuntu-latest` by default and expose `runs-on` as an input so a consumer can point at a self-hosted runner later. The `working-directory` input on each language workflow is applied via `defaults.run.working-directory` at the job level (per the `github-actions-cicd` skill's pattern), so every `run:` step in the job executes there without repeating the value.

### 4.1 `tofu-module-ci.yml`

**Inputs:**
| Name | Type | Default | Purpose |
| --- | --- | --- | --- |
| `tofu_version` | string | `"1.9.1"` | OpenTofu version to install |
| `working-directory` | string | `.` | Module root |
| `runs-on` | string | `ubuntu-latest` | Runner label |

**Permissions:** `contents: read`
**Concurrency:** cancel-in-progress per PR
**Timeout:** 10 min

**Job — `validate`:**
1. `actions/checkout@<sha>`
2. `opentofu/setup-opentofu@<sha>` with `tofu_version`
3. `terraform-linters/setup-tflint@<sha>` → `tflint --init` → `tflint --recursive`
4. `tofu fmt -check -recursive`
5. `tofu init -backend=false`
6. `tofu validate`
7. `tofu test` — guarded by a bash conditional on `find . -name '*.tftest.hcl' -print -quit` so modules without tests are not failed
8. `aquasecurity/trivy-action@<sha>` — `scan-type: config`, `severity: HIGH,CRITICAL`, `exit-code: 1`

### 4.2 `python-ci.yml`

**Inputs:**
| Name | Type | Default | Purpose |
| --- | --- | --- | --- |
| `python-version` | string | `"3.12"` | Python version to install |
| `working-directory` | string | `.` | Project root (path to `pyproject.toml`) |
| `runs-on` | string | `ubuntu-latest` | Runner label |

**Permissions:** `contents: read`
**Concurrency:** cancel-in-progress per PR
**Timeouts:** 5 min per lint/typecheck job, 15 min test job

Structure follows the `github-actions-cicd` skill: separate jobs with a `needs:` chain give distinct PR status checks and fail-fast readability.

**Jobs:**

| Job | Depends on | Steps |
| --- | --- | --- |
| `lint-python` | — | checkout → `astral-sh/setup-uv@<sha>` (`enable-cache: true`) → `uv python install ${{ inputs.python-version }}` → `uv sync --locked --dev` → `uv run ruff check .` → `uv run ruff format --check .` |
| `typecheck-python` | `lint-python` | checkout → setup-uv → `uv sync --locked --dev` → `uv run mypy .` |
| `test-python` | `typecheck-python` | checkout → setup-uv → `uv sync --locked --dev` → `uv run pytest -q` |
| `deps-audit` | — (parallel) | checkout → setup-uv → `uv sync --locked --dev` → `uv run pip-audit` |

**Security scanning inside `lint-python`:** ruff's `S` (flake8-bandit) rules cover static analysis. No separate `bandit` step. Consumer `pyproject.toml` must include `S` in its ruff `select` — enforced by convention in `CONTRIBUTING.md`. This aligns with the `security` skill's principle of folding security linting into the tools already running.

**`uv sync --locked --dev`** (not `--frozen`): `--locked` fails the job if `uv.lock` is out of date rather than silently using it.

### 4.3 `secret-scan.yml`

**Inputs:**
| Name | Type | Default | Purpose |
| --- | --- | --- | --- |
| `runs-on` | string | `ubuntu-latest` | Runner label |

**Permissions:** `contents: read`
**Timeout:** 5 min

**Job — `scan`:**
1. `actions/checkout@<sha>` with `fetch-depth: 0` (needed for history scan)
2. Install `gitleaks` CLI directly from the GitHub Release archive — **not** the `gitleaks/gitleaks-action`, which requires a `GITLEAKS_LICENSE` secret for organization accounts on private repos. The CLI is the same tool, no licensing gate.
3. `gitleaks detect --source . --redact --exit-code 2` — scans the full checked-out history
4. Honours a `.gitleaks.toml` at the consumer repo root for allowlist entries (same file the local `scan-secrets.sh` hook uses — symmetry between dev-time and CI-time)

### 4.4 `pr-title-lint.yml`

**Trigger:** `workflow_call` — consumer calls from `on: pull_request`
**Permissions:** `pull-requests: read`
**Timeout:** 2 min

**Job — `lint`:** `amannn/action-semantic-pull-request@<sha>` configured with the allowed types from the `git-conventional-commits` skill: `feat | fix | docs | style | refactor | test | chore | ci | revert`.

**Caveat verified in Phase 3 pilot:** this action needs the `pull_request` event context. Reusable workflows inherit the caller's event, so a consumer calling from `on: pull_request` propagates it. Pilot must exercise a real PR to confirm.

### 4.5 `commit-lint.yml`

**Trigger:** `workflow_call`
**Permissions:** `contents: read`
**Timeout:** 3 min

**Job — `lint`:** `actions/checkout@<sha>` with `fetch-depth: 0` → `wagoid/commitlint-github-action@<sha>` with an inline config matching the allowed types.

## 5. Foundation files

### 5.1 `profile/README.md`

Renders at `github.com/invest4talent`. Skeleton in the spec with `<TODO: supply real value>` markers for org copy — mission statement, product links, careers link, external contact. Content owner supplies.

### 5.2 `SECURITY.md`

Structure:
- **Reporting a vulnerability** — email address (`<TODO: supply real value>` — e.g. `security@invest4talent.com`)
- **Scope** — all public and private Invest4Talent repositories
- **What to expect** — acknowledgment within 3 business days; no bug-bounty language
- **Do not** report via public GitHub issues

### 5.3 `CODE_OF_CONDUCT.md`

Contributor Covenant v2.1 verbatim, with the enforcement contact replaced (`<TODO: supply real value>`).

### 5.4 `CONTRIBUTING.md`

The human-readable ops guide. Sections:
- **Branch naming** — `type/short-description`, allowed types (from the git skill)
- **Commit messages** — Conventional Commits format + allowed types table
- **PR title + description** — points to the auto-populated PR template
- **Local dev setup** — `pre-commit install` one-liner + the canonical `.pre-commit-config.yaml` snippet (§7.3)
- **Consumer integration runbook** — the single `ci.yml` pattern (§7.1)
- **Consumer branch protection ruleset** — verbatim step-by-step (§7.2)
- **Consumer `pyproject.toml` requirement** — `S` must be in ruff `select` (§7.4)
- **Deliberate divergence from `github-actions-cicd` skill** — one paragraph explaining that consumer repos may pin actions by tag (per skill) but this org-shared repo pins to SHA because the blast radius is org-wide

### 5.5 `.github/pull_request_template.md`

Straight from the git skill:

```markdown
## What

## Why

## How tested

## Notes
```

### 5.6 `.github/ISSUE_TEMPLATE/`

- `bug_report.yml` — form-based; fields: what happened, expected, repro steps, environment, severity
- `feature_request.yml` — problem, proposed solution, alternatives considered
- `config.yml` — `blank_issues_enabled: false`, `contact_links: <TODO: supply real value>` (e.g. security link → `SECURITY.md`)

### 5.7 `.github/dependabot.yml` (this repo only)

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 5
```

Consumers ship their own `dependabot.yml` in their own repos — documented in `CONTRIBUTING.md`.

## 6. Versioning and release

### 6.1 SemVer scheme

Immutable tags `v1.0.0`, `v1.1.0`, `v1.2.0` for every release. A **floating major-version tag `v1`** always points at the newest `v1.x.x`. Consumers use `@v1` and get non-breaking updates automatically. Breaking changes cut `v2.0.0` with a new `v2` floating tag; `v1` stays put so nothing breaks silently.

### 6.2 Breaking-change definition (published contract)

**Breaking:**
- Removing or renaming an input on a reusable workflow
- Changing an input's type or removing a default
- Removing or renaming a job (branch protection references job names)
- Increasing the minimum required consumer permissions
- Dropping support for a language/tool version that was previously advertised
- Tightening scans in a way that fails previously-passing repos

**Not breaking:**
- Adding a new optional input
- Adding a new step to an existing job
- Bumping SHA-pinned action versions when behaviour is unchanged
- Documentation-only changes

### 6.3 Release process

**v1.0.0 — manual bootstrap:**
1. All work lands on `main` via PR; self-CI must be green.
2. Cut the immutable tag: `git tag -a v1.0.0 -m "release v1.0.0" && git push origin v1.0.0`
3. Move the floating major tag: `git tag -f v1 v1.0.0 && git push -f origin v1`
4. Create a GitHub Release from `v1.0.0` with a manual changelog.

**v1.1.0 onwards — automated via `release.yml`:**

`workflow_dispatch` with a `version` input. Cuts the immutable tag, moves the major tag, drafts the release with `generate_release_notes: true` (which reads PR titles between tags — Conventional Commits produces clean notes for free).

Sketch (final file lands in Phase 6):

```yaml
name: Release
on:
  workflow_dispatch:
    inputs:
      version:
        description: "Version to release (e.g. v1.2.0)"
        required: true
        type: string

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@<sha>
        with: { fetch-depth: 0 }
      - name: Validate version format
        run: |
          [[ "${{ inputs.version }}" =~ ^v[0-9]+\.[0-9]+\.[0-9]+$ ]] || {
            echo "Version must be vX.Y.Z"; exit 1;
          }
      - name: Create immutable tag
        run: |
          git tag -a "${{ inputs.version }}" -m "release ${{ inputs.version }}"
          git push origin "${{ inputs.version }}"
      - name: Move major tag
        run: |
          MAJOR="${{ inputs.version }}"
          MAJOR="${MAJOR%%.*}"            # v1.2.0 -> v1
          git tag -f "$MAJOR" "${{ inputs.version }}"
          git push -f origin "$MAJOR"
      - name: Create GitHub Release
        uses: softprops/action-gh-release@<sha>
        with:
          tag_name: ${{ inputs.version }}
          generate_release_notes: true
```

### 6.4 Branch and tag protection on this repo

**Branch protection on `main`** (this repo):
- Require PR before merging
- Require self-CI status checks to pass
- Require branches to be up to date before merging
- Disallow force pushes to `main`
- Disallow deletion of `main`

**Tag protection ruleset** (separate from branch protection; Settings → Rules → Rulesets → target tags):
- Target pattern `v*`
- **Allow tag force-updates for the `Actions` bot** (needed by `release.yml` to move the `v1` floating tag) — restrict to admins + the workflow
- Disallow tag deletion by non-admins

Without the tag ruleset allowance for force-updates, `release.yml` will fail when it tries to move `v1`.

## 7. Consumer integration pattern

### 7.1 Single `ci.yml` with baseline + conditional blocks

One file per repo, sections deleted per stack. Matches the `github-actions-cicd` skill's "keep it flat" philosophy.

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

**Every repo keeps the BASELINE block.** Python and OpenTofu blocks are conditional. Multi-stack repos keep both language blocks — they run in parallel.

### 7.2 Consumer branch protection ruleset

Free plan cannot push rulesets centrally. Each repo owner runs this once. **Important free-plan caveat:** branch rulesets and classic branch protection are only available on **public** repositories on the free plan. Private consumer repos on the free plan cannot mark checks as required for merge — the CI still runs on every PR, but merge blocking is discipline-based until the org upgrades to Team (or the repo is made public). This is why `Invest4Talent/.github` itself is public; consumer repos that hold shared/module templates should typically be public too, while private app repos should either upgrade the plan or lean on PR review discipline.

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

Optional (turn on if the team wants them):
 ☐ Require signed commits — adds friction; only if the team is set up for it
 ☐ Require linear history — pair with squash-merge for a clean main
```

**Note on check names:** when a reusable workflow is called, the check appears as `<job-name-in-caller> / <job-name-in-reusable>`. That is why the check names are doubled (e.g. `python / lint-python`). Not pretty, but stable and grep-able.

### 7.3 Canonical `.pre-commit-config.yaml`

Same baseline/conditional shape as `ci.yml`. Delete blocks per stack.

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

**Deliberately excluded:** `mypy`, `pytest`, `pip-audit`, `bandit` (via ruff `S`), `tofu validate`, `tofu test`, `tflint`, `trivy`. Rationale: too slow for every commit, or already covered by ruff, or too dependent on repo-specific config. All are covered by CI.

### 7.4 Consumer `pyproject.toml` — required for security coverage

Python projects must include `S` in ruff select. This is what replaces the standalone bandit step in `python-ci.yml`.

```toml
[tool.ruff.lint]
select = ["E", "F", "I", "S", ...]  # S = flake8-bandit rules

[tool.ruff.lint.per-file-ignores]
"tests/**/*.py" = ["S101"]  # pytest uses `assert`
```

### 7.5 Optional consumer files

- **`.gitleaks.toml`** at repo root — allowlist for false positives. Honoured by both the local `scan-secrets.sh` Claude hook and the reusable `secret-scan.yml` workflow. Symmetry across dev-time and CI-time.
- **Own `pull_request_template.md`** — overrides the org default if the repo needs a project-specific PR structure. Most repos should let the org default apply.

### 7.6 Consumer `dependabot.yml`

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
  # add python, npm, terraform ecosystems as appropriate for the project
```

### 7.7 First-time consumer setup runbook

Order matters — this exact sequence lives in `CONTRIBUTING.md`:
1. Copy the `ci.yml` template above into `.github/workflows/`
2. Copy the `dependabot.yml` template above into `.github/`
3. Copy the `.pre-commit-config.yaml` template and run `pre-commit autoupdate` to pin `rev:` fields
4. For Python repos: ensure `pyproject.toml` has `S` in ruff select
5. Each developer runs `pre-commit install` once per clone
6. Open a PR — this makes the check names visible in GitHub
7. Configure the branch protection ruleset per §7.2 with the checks that actually ran

## 8. Alignment with existing skills and hooks

**`github-actions-cicd` skill — adopted:**
- Job structure: split `lint-python` / `typecheck-python` / `test-python` with `needs:` chain
- `astral-sh/setup-uv` with `enable-cache: true`
- `uv sync --locked --dev` (fails on stale lock file)
- `pytest -q` for quieter CI logs
- Clear job names because they surface in PR status checks

**`github-actions-cicd` skill — deliberate divergence:**
- Actions SHA-pinned in this org-shared repo (skill uses tag pins). Rationale: blast radius is org-wide; supply-chain risk warrants stricter pinning. Consumer repos may follow the skill's tag pins. Called out in `CONTRIBUTING.md`.

**`security` skill — adopted:**
- Static analysis folded into ruff (`S` rules) rather than a separate bandit step
- `pip-audit` in CI for known-vuln dependencies
- Gitleaks in CI for history scanning
- Least-privilege `permissions:` block on every workflow

**`git-conventional-commits` skill — adopted:**
- Allowed types propagated verbatim into `pr-title-lint.yml`, `commit-lint.yml`, and `conventional-pre-commit`
- PR template from the skill's structure (What / Why / How tested / Notes)

**`scan-secrets.sh` local hook — composes with:**
- `.gitleaks.toml` at repo root is the shared allowlist config between the local hook and the CI workflow. Same tool, same config, dev-time and CI-time.

**`protect-files.sh` local hook — no interaction:**
- None of the blocked paths (`.env*`, lock files, migrations, prod compose) live in this repo.

## 9. Rollout (v1 phases)

Phases are ordered so each produces something usable and de-risks the next.

### Phase 1 — Foundation files
Ship first; independent, low-risk, immediate value.
- `README.md`
- `profile/README.md`
- `SECURITY.md`, `CODE_OF_CONDUCT.md`, `CONTRIBUTING.md`
- `.github/pull_request_template.md`
- `.github/ISSUE_TEMPLATE/{bug_report,feature_request,config}.yml`
- `.github/dependabot.yml`
- `CLAUDE.md`

**Verify:** push to GitHub; confirm `github.com/invest4talent` renders the profile page; open a test issue and PR in this repo to see templates fire; confirm all `<TODO: supply real value>` markers have been resolved.

### Phase 2 — Self-CI for this repo
- `.github/workflows/self-ci.yml` — `actionlint` + `yamllint` on PRs to this repo
- Enable branch protection on `main` (same shape as the consumer ruleset, minus language jobs), requiring self-CI to pass

**Verify:** intentionally break a workflow YAML in a PR; confirm the PR is blocked.

### Phase 3 — Reusable workflows (in order)
One PR per workflow, reviewable individually.
1. `secret-scan.yml` — smallest surface
2. `pr-title-lint.yml`
3. `commit-lint.yml`
4. `python-ci.yml` — multi-job pattern debuts here
5. `tofu-module-ci.yml`

**Verify each** by wiring it into a pilot consumer repo pinned to `@main` (not `@v1` — no tag exists yet). Confirm the check passes on green input and fails on the failure it is meant to catch:
- `secret-scan.yml` — plant a fake secret, confirm the scan fails
- `pr-title-lint.yml` — open a PR with a non-conventional title, confirm it fails; **explicitly exercise a real PR to confirm the event context propagates through `workflow_call`**
- `commit-lint.yml` — push a bad commit in a PR, confirm the check fails; **same event-context caveat as `pr-title-lint.yml` — confirm via a real PR**
- `python-ci.yml` — introduce a ruff violation, a type error, a failing test, an outdated `uv.lock`; confirm the right job fails in each case
- `tofu-module-ci.yml` — introduce unformatted HCL, an invalid config, a Trivy-flagged misconfiguration; confirm the right step fails

### Phase 4 — Cut `v1.0.0` (manual bootstrap)
Only after all five reusables have been exercised end-to-end via the pilot.
1. Ensure `main` is green.
2. `git tag -a v1.0.0 -m "release v1.0.0" && git push origin v1.0.0`
3. `git tag -f v1 v1.0.0 && git push -f origin v1`
4. Create a GitHub Release from `v1.0.0` with a manual changelog.
5. Update the pilot repo to `@v1`; confirm nothing breaks.

### Phase 5 — Team adoption
- Announce; link `CONTRIBUTING.md`
- Pair with one owner per repo for the first rollout (copy `ci.yml`, wire `dependabot.yml`, configure branch protection ruleset per §7.2)
- Track adoption informally

### Phase 6 — Automated release tooling
- Ship `.github/workflows/release.yml` per §6.3
- Configure the tag protection ruleset per §6.4 so `release.yml` can move the floating major tag
- **First real use:** cut `v1.1.0` via `workflow_dispatch`. Bootstrap manually with `v1.0.0` (done in Phase 4); every release after that is one click.

### v1 done when
- [ ] All Phase-1 files present; all `<TODO: supply real value>` markers resolved
- [ ] `main` has branch protection enabled and self-CI is required
- [ ] All five reusable workflows exist and are tested against a pilot repo
- [ ] `v1.0.0` and `v1` tags pushed; GitHub Release created
- [ ] At least one non-pilot consumer repo migrated to `@v1` with branch protection configured per §7.2
- [ ] `release.yml` shipped; tag protection ruleset configured; `v1.1.0` cut via `workflow_dispatch` to prove the automation works end-to-end

## 10. Open questions / placeholders

Content owners need to supply:
- `profile/README.md` — org mission, product links, careers, external contact
- `SECURITY.md` — vulnerability disclosure email
- `CODE_OF_CONDUCT.md` — enforcement contact
- `.github/ISSUE_TEMPLATE/config.yml` — external contact links (security policy link at minimum)

Team decisions to confirm during Phase 5:
- Merge strategy (squash-merge vs merge commit vs rebase) — affects whether `commit-lint.yml` is strictly required for consumers and whether "Require linear history" makes sense
- Whether to require signed commits (currently optional in §7.2)
