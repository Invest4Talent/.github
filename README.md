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

## Verifying the CI still works

See [`TESTING.md`](TESTING.md) — runbook for pilot verifications, release dry-runs, and ruleset drift checks. Not needed on the happy path; kept for post-refactor or post-incident sanity.

## Security

See [`SECURITY.md`](SECURITY.md) — do not open public issues for vulnerabilities.
