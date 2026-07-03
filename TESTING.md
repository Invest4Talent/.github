# Testing — how to verify this repo's CI still works

Three self-contained runbooks:

1. **Pilot verifications** — is each reusable workflow still catching what it should?
2. **Release dry-runs** — sanity-check `release.yml` before cutting a real version
3. **Ruleset drift verification** — did anyone change the branch or tag ruleset via the GitHub UI?

You should not need any of this on the happy path — self-CI, Dependabot, and the pilot's standing PR keep the surface healthy on their own. This runbook is for when something has changed (a refactor, a new pinned action, an incident) and you want to prove nothing broke.

There is a **Fast health check** at the bottom that covers all three at a glance.

---

## 1. Pilot verifications

The pilot repo is `Invest4Talent/ci-pilot` — private, wired to `@v1`. It contains a minimal Python module (`src/ci_pilot/main.py` + `tests/test_main.py`) and a minimal OpenTofu module (`infra/main.tf` + `infra/main.tftest.hcl`). PR #3 (`ci/wire-baseline`) stays open with all baseline + Python + Tofu jobs green as the standing "everything works" baseline.

Every reusable follows the same verification shape:

1. Confirm the pass path is still green on the pilot as-is (PR #3's most recent run).
2. Plant one specific failure on a throwaway branch.
3. Confirm ONLY the intended check fails.
4. Clean up.

**Cleanup rule for all fail-plants:** close the throwaway PR without merging and delete the branch. `secret-scan.yml` scans full git history — a `git revert` on the same branch does not clear the check, because the plant commit is still reachable from HEAD.

### `secret-scan.yml` → check `secrets / scan`

**Pass path:** no secret matches anywhere in pilot history. Confirmed by every green run on PR #3.

**Fail plant:**
```bash
cd /path/to/ci-pilot
git checkout main && git pull
git checkout -b test/verify-secret-scan
cat > test-secret.txt <<'EOF'
# DO NOT COMMIT REAL SECRETS. Fake for gitleaks CI verification.
AWS_ACCESS_KEY_ID = "AKIAZXCVBNMLKJHGFDSA"  # gitleaks:allow
EOF
git add test-secret.txt
git commit -m "test: plant fake AWS-style secret"
git push -u origin test/verify-secret-scan
gh pr create --title "test: verify secret-scan" --body "verification" --repo Invest4Talent/ci-pilot
```

**Expected:** `secrets / scan` = **failure**, log shows `leaks found: 1`, exit code 2.

**Do NOT use `AKIAIOSFODNN7EXAMPLE`.** Gitleaks' shipped config allowlists any AWS key ending `EXAMPLE` (`.+EXAMPLE$`). The suggested `AKIAZXCVBNMLKJHGFDSA` uses only base32 characters (A–Z, 2–7) and does not end in `EXAMPLE`.

**Cleanup:** `gh pr close <#> --delete-branch --repo Invest4Talent/ci-pilot` (do not revert — see cleanup rule above).

### `pr-title-lint.yml` → check `pr-title / lint`

**Pass path:** any Conventional Commits–compliant PR title, e.g. `chore: verify pr-title lint`.

**Fail plant:**
```bash
gh pr edit <PR#> --title "bad title without type" --repo Invest4Talent/ci-pilot
```

**Expected:** `pr-title / lint` = **failure**, log lists the 9 allowed types (`feat, fix, docs, style, refactor, test, chore, ci, revert`) and complains that none was found.

**Cleanup:** restore the good title:
```bash
gh pr edit <PR#> --title "chore: verify pr-title lint" --repo Invest4Talent/ci-pilot
```

### `commit-lint.yml` → check `commits / lint`

**Pass path:** every commit in the PR matches Conventional Commits (allowed types + lowercase subject + no trailing period).

**Fail plants** (pick one; add an empty commit so no code changes):

| Rule | Message |
| :--- | :--- |
| `type-empty` | `bad message without type` |
| `subject-case` | `chore: Verify UPPERCASE subject` |
| `subject-full-stop` | `chore: message with a trailing period.` |

```bash
cd /path/to/ci-pilot
git checkout <existing-test-branch>
git commit --allow-empty -m "bad message without type"
git push
```

**Expected:** `commits / lint` = **failure**, log names the offending rule.

**Cleanup:** since the bad commit is in the branch history, either force-reset before it (`git reset --hard HEAD~1 && git push --force-with-lease`) or close and re-open the PR from a fresh branch.

### `python-ci.yml` → checks `python / lint-python`, `typecheck-python`, `test-python`, `deps-audit`

Four independent plants, one per job. Revert between plants (`git reset --hard HEAD~1 && git push --force-with-lease` if you can force-push, otherwise `git revert HEAD && git push`).

| Job | Plant | File |
| :--- | :--- | :--- |
| `lint-python` | Add unused import (`import os`) at the top | `src/ci_pilot/main.py` |
| `typecheck-python` | Change `greet` to `return 1` (mypy sees `int` where `str` is annotated) | `src/ci_pilot/main.py` |
| `test-python` | Add `def test_fail(): assert False` | `tests/test_main.py` |
| `deps-audit` + `lint-python` | Add a new dev dep without running `uv lock`; `--locked` fails | `pyproject.toml` |

The stale-lock plant fails **both** `deps-audit` and `lint-python` (both run `uv sync --locked --dev` at their first step). All other plants isolate to their intended job.

### `tofu-module-ci.yml` → check `tofu / validate`

Four internal steps to exercise, one plant per step.

| Step | Plant | Where |
| :--- | :--- | :--- |
| `Check formatting` | Bad indent or extra whitespace on any HCL block | `infra/main.tf` |
| `Run TFLint` | Add `variable "unused" {}` without any reference | `infra/main.tf` |
| `Validate` | Add `unknown_arg = 1` on `random_pet.example` | `infra/main.tf` |
| `Trivy misconfiguration scan` | Add a `Dockerfile` at the pilot root with `FROM ubuntu:latest\nUSER root\n` (triggers `DS-0002` HIGH) | `Dockerfile` at the pilot root |

Trivy's default scan targets include Dockerfiles (`DS-*` rules) even under `scan-type: config`. That is the approach the original Task 10 verification used.

`tofu test` runs conditionally on the presence of any `.tftest.hcl` file. The pilot ships `infra/main.tftest.hcl`, so the step runs on every clean run — look for `Executing test` in the log.

---

## 2. Release dry-runs

The regex on `release.yml` is strict: `^v[0-9]+\.[0-9]+\.[0-9]+$`. Pre-release suffixes like `-rc1` or `-dryrun` are rejected by design. Three dry-run strategies, in ascending cost:

### Option A — Local workflow lint (default)

Before dispatching, when you have edited `release.yml`:
```bash
actionlint .github/workflows/release.yml
yamllint  .github/workflows/release.yml
```
Both must exit 0. Self-CI catches this at merge time; running locally shortens the round-trip.

If you have [`zizmor`](https://woodruffw.github.io/zizmor/) installed:
```bash
zizmor .github/workflows/release.yml
```
It flags Actions-specific patterns including the template-injection surface that the v1 whole-branch review caught. Adding a `zizmor` step to `self-ci.yml` is a post-v1 recommendation.

**Use when:** you edited `release.yml` — always run this before dispatch.

### Option B — Canary version on this repo

Cut a large synthetic version (`v999.0.0`) from `main`, confirm all steps ran green, then clean up. Because the tag ruleset restricts deletion, admin bypass is required.

```bash
# 1. Cut the canary
gh workflow run release.yml -f version=v999.0.0

# 2. Wait for the run and verify
RUN_ID=$(gh run list --workflow=release.yml --limit 1 --json databaseId --jq '.[0].databaseId')
gh run watch "$RUN_ID" --exit-status

git fetch --tags --force
git rev-parse v999.0.0^{commit}
git rev-parse v999^{commit}
gh release view v999.0.0

# 3. Clean up (admin bypass required for tag deletion)
gh release delete v999.0.0 --yes
gh api --method DELETE repos/Invest4Talent/.github/git/refs/tags/v999.0.0
gh api --method DELETE repos/Invest4Talent/.github/git/refs/tags/v999
```

**Use when:** there's genuine doubt about the release mechanics — a refactor of `release.yml`, a new pinned action, changed permissions, first release after a compromise investigation. It leaves an audit trail (a deleted release, a deleted tag in the git reflog) so don't do it every release.

### Option C — Fork or scratch repo

Fork `Invest4Talent/.github` to a personal account (or a throwaway org repo) and disable the tag ruleset on the fork. Cut disposable tags there. Highest isolation, highest setup cost.

**Use when:** you are redesigning the release model — changing tag semantics, replacing the underlying release action, rebuilding the tag ruleset from scratch, or testing a compromised-token recovery drill.

---

## 3. Ruleset drift verification

Two rulesets protect this repo. Anyone with admin access can change them via the GitHub UI, and those changes do not surface as a commit. Dump them any time and compare against the expected shape below.

### Branch ruleset — `main` (id `18430699`)

```bash
gh api repos/Invest4Talent/.github/rulesets/18430699 \
  --jq '{name, target, enforcement, conditions, rules: [.rules[].type], bypass: [.bypass_actors[] | {actor_type, actor_id, bypass_mode}]}'
```

**Expected:**
```json
{
  "name": "Protect main",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "include": ["~DEFAULT_BRANCH"],
      "exclude": []
    }
  },
  "rules": ["pull_request", "required_status_checks", "non_fast_forward", "deletion"],
  "bypass": [
    { "actor_type": "RepositoryRole", "actor_id": 5, "bypass_mode": "always" }
  ]
}
```

(`~DEFAULT_BRANCH` and `refs/heads/main` are equivalent when the default branch is `main`. Either is accepted.)

**Required status checks specifically:**
```bash
gh api repos/Invest4Talent/.github/rulesets/18430699 \
  --jq '.rules[] | select(.type=="required_status_checks") | .parameters.required_status_checks[].context'
```
Expected (order not significant):
```
actionlint
yamllint
```

### Tag ruleset — `v*` (id `18459577`)

```bash
gh api repos/Invest4Talent/.github/rulesets/18459577 \
  --jq '{name, target, enforcement, conditions, rules: [.rules[].type], bypass: [.bypass_actors[] | {actor_type, actor_id, bypass_mode}]}'
```

**Expected:**
```json
{
  "name": "Protect version tags",
  "target": "tag",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "include": ["refs/tags/v*"],
      "exclude": []
    }
  },
  "rules": ["non_fast_forward", "deletion"],
  "bypass": [
    { "actor_type": "RepositoryRole", "actor_id": 5, "bypass_mode": "always" }
  ]
}
```

### If either ruleset has drifted

Restore via the UI (Settings → Rules → Rulesets → edit) or via API `PUT /repos/Invest4Talent/.github/rulesets/{id}` with the original payload. The originating payloads are in the SDD reports:

- Branch ruleset — `.superpowers/sdd/task-5-report.md`
- Tag ruleset — `.superpowers/sdd/task-12-report.md`

**Do not delete-and-recreate.** A new ruleset gets a new ID, and this document (plus any monitoring you wire in later) refers to the current IDs.

---

## Fast health check

For a monthly sweep or a post-incident sanity check, the whole runbook collapses to three commands:

```bash
# 1. Ruleset drift — both should show enforcement:active
gh api repos/Invest4Talent/.github/rulesets/18430699 --jq '{n:.name, e:.enforcement}'
gh api repos/Invest4Talent/.github/rulesets/18459577 --jq '{n:.name, e:.enforcement}'

# 2. Pilot standing check status — all 8 checks should be pass
gh pr checks 3 --repo Invest4Talent/ci-pilot

# 3. Last release ran clean — status should be success
gh run list --workflow=release.yml --limit 1
```

If (1) shows `enforcement:active` on both, (2) shows every check passing, and (3) shows the last release as `success`, everything is healthy without deeper verification. Any deviation → drop into the section above that matches the failing signal.
