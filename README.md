# ci-workflows

Shared GitHub Actions reusable workflows for the Infinity Constellation GitHub org.

## Compatibility notes

The Python `uv` workflows below are designed to be the org-wide default for
any Python project managed with [`uv`](https://docs.astral.sh/uv/). They were
audited against the existing CI of `infinity-factory-agent`,
`infinity-factory-api`, `gravity-contracts`, and `gravity-temporal`. To stay
flexible across these:

- Every command (install, test, build, lint, replay) is overridable via an
  input.
- `pre-*` / `post-*` hooks let callers run repo-specific commands (Django
  `manage.py` checks, `gravity-contracts validate`, etc.) inside the same
  job without forking the workflow.
- `extra-env` exposes a multi-line `KEY=VALUE` block for non-secret config
  (CI flags, target env names, fake/test-only keys). Real secrets stay in
  caller-side jobs that pass them via `secrets:` to the relevant tool.
- `working-directory` and `cache-dependency-glob` support monorepos and
  non-root project layouts.
- `actions/setup-uv@v4` and `uv python install <version>` mirror the
  existing pattern in the org (`infinity-factory-agent/ci.yml`).

If a repo's CI step doesn't fit any of these knobs, please open an issue or
PR on this repo rather than copy-pasting workflow YAML — the goal is one
canonical Python `uv` lint/build pipeline across the org.

## Workflows

- [`python-uv-lint.yml`](#python-uv-lintyml--lint-a-python-uv-project) — Lint a Python `uv` project (ruff, optionally black/mypy).
- [`python-uv-build.yml`](#python-uv-buildyml--build-and-test-a-python-uv-project) — Install dependencies, run tests, and optionally build a Python `uv` project.
- [`temporal-replay-test.yml`](#temporal-replay-testyml--temporal-workflow-replay-tests) — Run Temporal workflow replay tests for a Python `uv` project.
- [`docker-ecr-release.yml`](#docker-ecr-releaseyml--build-and-optionally-push-a-docker-image-to-ecr) — Build and optionally push a Docker image to ECR.
- [`notify-release.yml`](#notify-releaseyml--slack-release-notification) — Slack release notification.
- [`devops-agent-plan.yml`](#gravity-devops-agent-v0-iac-orchestration) / [`devops-agent-approve.yml`](#gravity-devops-agent-v0-iac-orchestration) / [`devops-agent-apply.yml`](#gravity-devops-agent-v0-iac-orchestration) / [`devops-agent-postmortem.yml`](#gravity-devops-agent-v0-iac-orchestration) — `gravity-devops-agent` v0: PR-driven plan, CODEOWNERS-gated auto-apply, auto-merge, structured postmortem.

### `python-uv-lint.yml` — Lint a Python `uv` project

Runs `ruff check` by default and optionally `ruff format --check`,
`black --check`, `mypy`, and `basedpyright` against a Python project managed
with [`uv`](https://docs.astral.sh/uv/). Each tool has its own toggle and
paths input; pre/post hooks let callers run repo-specific commands
(schema validation, codegen, etc.) inside the same job.

**Usage**

```yaml
lint:
  uses: infinity-constellation/ci-workflows/.github/workflows/python-uv-lint.yml@main
  with:
    python-version: "3.12"
    black-check: true
    mypy-check: true
```

**Inputs**

| Input | Required | Default | Description |
|---|---|---|---|
| `python-version` | no | `3.12` | Python version to install via `uv python install`. |
| `uv-version` | no | `latest` | uv version passed to `astral-sh/setup-uv`. |
| `working-directory` | no | `.` | Directory containing `pyproject.toml`. |
| `cache-dependency-glob` | no | `**/uv.lock` | Glob passed to `astral-sh/setup-uv` to scope its cache key. |
| `install-command` | no | `uv sync --all-extras --dev` | Command used to install dependencies. |
| `pre-lint-command` | no | _(empty)_ | Optional command run after install, before lint checks. |
| `post-lint-command` | no | _(empty)_ | Optional command run after lint checks. |
| `extra-env` | no | _(empty)_ | Multi-line `KEY=VALUE` pairs appended to `$GITHUB_ENV` before user commands. Non-secret config only. |
| `ruff-check` | no | `true` | Run `uv run ruff check`. |
| `ruff-check-paths` | no | `.` | Paths passed to `ruff check`. |
| `ruff-format-check` | no | `false` | Run `uv run ruff format --check`. |
| `ruff-format-paths` | no | `.` | Paths passed to `ruff format --check`. |
| `black-check` | no | `false` | Run `uv run black --check`. |
| `black-paths` | no | `.` | Paths passed to `black --check`. |
| `mypy-check` | no | `false` | Run `uv run mypy`. |
| `mypy-paths` | no | `.` | Paths passed to `mypy`. |
| `basedpyright-check` | no | `false` | Run `uv run basedpyright`. |
| `basedpyright-paths` | no | `.` | Paths passed to `basedpyright`. |
| `runs-on` | no | `ubuntu-latest` | Runner label. |

### `python-uv-build.yml` — Build and test a Python `uv` project

Installs dependencies, runs tests, and optionally runs additional steps
(pre-test setup, post-test deploy checks, packaging, artifact upload).

**Usage — basic**

```yaml
build:
  uses: infinity-constellation/ci-workflows/.github/workflows/python-uv-build.yml@main
  with:
    python-version: "3.12"
    test-command: "uv run python -m pytest -v"
```

**Usage — Django (with migration check + deploy check)**

```yaml
build:
  uses: infinity-constellation/ci-workflows/.github/workflows/python-uv-build.yml@main
  with:
    pre-test-command: "uv run python manage.py makemigrations --check --dry-run"
    post-test-command: "uv run python manage.py check --deploy --fail-level WARNING"
    extra-env: |
      SECRET_DJANGO_SECRET_KEY=ci-testing-only-not-a-real-secret
```

**Usage — package + upload**

```yaml
build:
  uses: infinity-constellation/ci-workflows/.github/workflows/python-uv-build.yml@main
  with:
    test-command: "uv run gravity-contracts validate && uv run pytest"
    build-command: "uv run gravity-contracts bundle --output dist/bundle.tar.gz"
    upload-artifact-name: "contracts-bundle-${{ github.sha }}"
    upload-artifact-path: "dist/bundle.tar.gz"
```

**Inputs**

| Input | Required | Default | Description |
|---|---|---|---|
| `python-version` | no | `3.12` | Python version to install via `uv python install`. |
| `uv-version` | no | `latest` | uv version passed to `astral-sh/setup-uv`. |
| `working-directory` | no | `.` | Directory containing `pyproject.toml`. |
| `cache-dependency-glob` | no | `**/uv.lock` | Glob passed to `astral-sh/setup-uv` to scope its cache key. |
| `install-command` | no | `uv sync --all-extras --dev` | Command used to install dependencies. |
| `pre-test-command` | no | _(empty)_ | Optional command run after install, before tests. |
| `test-command` | no | `uv run python -m pytest -v` | Command used to run tests. |
| `post-test-command` | no | _(empty)_ | Optional command run after tests. |
| `build-command` | no | _(empty)_ | Optional command run after `post-test-command` (e.g. `uv build`). |
| `upload-artifact-name` | no | _(empty)_ | If set, upload `upload-artifact-path` as a workflow artifact. |
| `upload-artifact-path` | no | _(empty)_ | Path uploaded when `upload-artifact-name` is set. |
| `extra-env` | no | _(empty)_ | Multi-line `KEY=VALUE` pairs appended to `$GITHUB_ENV` before user commands. Non-secret config only. |
| `runs-on` | no | `ubuntu-latest` | Runner label. |

### `temporal-replay-test.yml` — Temporal workflow replay tests

Runs Temporal workflow replay tests for a Python `uv` project. Replay tests
use [`temporalio.worker.Replayer`](https://python.temporal.io/temporalio.worker.Replayer.html)
to verify that the current workflow code can replay previously recorded
workflow histories without raising `NondeterminismError` — the standard
guardrail against breaking changes to deployed Temporal workflows.

**Usage**

```yaml
replay:
  uses: infinity-constellation/ci-workflows/.github/workflows/temporal-replay-test.yml@main
  with:
    python-version: "3.12"
```

To capture a history for replay, run on a workstation authenticated to your
Temporal namespace:

```bash
temporal workflow show \
  --workflow-id <id> \
  --output json > tests/replay/histories/<name>.json
```

Then commit the JSON file. The default replay command is
`uv run python -m pytest tests/replay -v`; override with `replay-command` to
point at a different test entrypoint.

**Inputs**

| Input | Required | Default | Description |
|---|---|---|---|
| `python-version` | no | `3.12` | Python version to install via `uv python install`. |
| `uv-version` | no | `latest` | uv version passed to `astral-sh/setup-uv`. |
| `working-directory` | no | `.` | Directory containing `pyproject.toml`. |
| `cache-dependency-glob` | no | `**/uv.lock` | Glob passed to `astral-sh/setup-uv` to scope its cache key. |
| `install-command` | no | `uv sync --all-extras --dev` | Command used to install dependencies. |
| `pre-replay-command` | no | _(empty)_ | Optional command run after install, before replay tests. |
| `replay-command` | no | `uv run python -m pytest tests/replay -v` | Command used to run replay tests. |
| `post-replay-command` | no | _(empty)_ | Optional command run after replay tests. |
| `extra-env` | no | _(empty)_ | Multi-line `KEY=VALUE` pairs appended to `$GITHUB_ENV` before user commands. Non-secret config only. |
| `runs-on` | no | `ubuntu-latest` | Runner label. |

### `notify-release.yml` — Slack release notification

### `docker-ecr-release.yml` — Build and optionally push a Docker image to ECR

Builds a Docker image and optionally pushes it to ECR. This workflow is meant
to be called after semantic-release determines whether a release was published.
On PRs, call it with `push: false` as a Docker build check. On `staging` and
`main`, call it with `push: true` and `version` set to the semantic-release
version.

The workflow resolves AWS/ECR settings from inputs first, then repository
variables:

- `AWS_REGION`
- `AWS_ROLE_TO_ASSUME`
- `ECR_REPOSITORY`
- `ECR_ACCOUNT_ID` (optional, defaults to `979686420760`)
- `ECR_REGISTRY` (optional, derived from account/region)

**Usage — PR build check**

```yaml
docker-build:
  uses: infinity-constellation/ci-workflows/.github/workflows/docker-ecr-release.yml@main
  with:
    service: gravity-temporal
    dockerfile: Dockerfile.production
    push: false
```

**Usage — release push**

```yaml
docker-release:
  needs: release
  if: needs.release.outputs.new_release_published == 'true'
  uses: infinity-constellation/ci-workflows/.github/workflows/docker-ecr-release.yml@main
  with:
    service: gravity-temporal
    version: ${{ needs.release.outputs.new_release_version }}
    dockerfile: Dockerfile.production
    push: true
  permissions:
    contents: read
    id-token: write
```

**Inputs**

| Input | Required | Default | Description |
|---|---|---|---|
| `service` | yes | — | Human-readable service name. |
| `dockerfile` | no | `Dockerfile.production` | Dockerfile path. |
| `context` | no | `.` | Build context. |
| `ecr-account-id` | no | repo var or `979686420760` | ECR account id. |
| `ecr-registry` | no | derived | ECR registry host. |
| `ecr-repository` | no | repo var `ECR_REPOSITORY` | ECR repository name. |
| `aws-region` | no | repo var `AWS_REGION` | AWS region. |
| `aws-role-to-assume` | no | repo var `AWS_ROLE_TO_ASSUME` | OIDC role ARN. |
| `push` | no | `true` | Push image to ECR. |
| `version` | required when `push=true` | — | Semver version without `v`. |
| `extra-tags` | no | _(empty)_ | Comma-separated extra tags. |
| `latest-on-main` | no | `false` | Add `latest` on main. |

### `notify-release.yml` — Slack release notification

Posts a message to the `#engineering` Slack channel when a new Docker image is
published to ECR.

**Usage** — add a `notify` job to your service's `release.yml`:

```yaml
notify:
  needs: build-and-push
  if: needs.release.outputs.new_release_published == 'true'
  uses: infinity-constellation/ci-workflows/.github/workflows/notify-release.yml@main
  with:
    service: infinity-factory-api
    version: ${{ needs.release.outputs.new_release_version }}
    environment: ${{ github.ref_name == 'main' && 'production' || 'staging' }}
    ecr_repository: ${{ vars.ECR_REPOSITORY }}
  secrets:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

**Inputs**

| Input | Required | Description |
|---|---|---|
| `service` | yes | Human-readable service name (e.g. `infinity-factory-api`) |
| `version` | yes | Semver version without `v` prefix (e.g. `1.2.3`) |
| `environment` | yes | `staging` or `production` |
| `ecr_repository` | yes | ECR repository name (e.g. `infinity-factory-api`) |

**Secrets**

| Secret | Required | Description |
|---|---|---|
| `SLACK_WEBHOOK_URL` | yes | Incoming webhook URL for the Slack app |

**Required repo secret** — add `SLACK_WEBHOOK_URL` to each service repo:

```bash
gh secret set SLACK_WEBHOOK_URL \
  --repo infinity-constellation/<repo> \
  --body "https://hooks.slack.com/services/..."
```

## gravity-devops-agent (v0): IaC orchestration

Four reusable workflows + one composite action that together implement the
**`gravity-devops-agent` v0** — a PR-driven, CODEOWNERS-gated, auto-merging,
auto-postmortem-on-failure operator for Terraform / Terragrunt
infrastructure changes.

Background: see
[ADR-0009 in `gravity-docs`](https://github.com/infinity-constellation/gravity-docs/blob/main/decisions/ADR-0009-iac-orchestration.md)
(local-only repo today). Long-term, v1 of this agent moves into a
`bridges/devops/` subtree of `gravity-agent` per the BridgeBase ABC; the
external contract (PR comments, artifacts, lifecycle events) is preserved.

### What it does

1. **On `pull_request` open / synchronize / reopen / labeled** — the caller
   discovers affected stacks, then calls `devops-agent-plan.yml` per stack.
   The plan posts a sticky PR comment, uploads the binary plan as an
   artifact, and (optionally) gates on a Conftest/OPA policy bundle. Engine
   (Terragrunt vs. raw Terraform) is auto-detected.
2. **On `pull_request_review` with state=approved** — the caller calls
   `devops-agent-approve.yml`. The approver is checked against CODEOWNERS
   for every affected path via the composite action
   `devops-agent-resolve-approver`. If the approver covers every path,
   `devops-agent-apply.yml` is dispatched; otherwise an "approval not yet
   sufficient" comment lists the required owners.
3. **`devops-agent-apply.yml`** — re-checks out the PR head, walks
   `terragrunt graph-dependencies` for the affected stacks, applies each
   stack's saved plan in topological order in a single job. On success: a
   structured outcome comment is posted and the PR is auto-squash-merged
   with a trailer block recording approver + applied stacks + run URL. On
   failure: the postmortem workflow is dispatched.
4. **`devops-agent-postmortem.yml`** — dismisses every current PR
   approval, adds the `apply-failure` label, and posts a structured
   postmortem comment with per-stack outcome, the failing-stack log tail,
   error category from a static playbook, and a suggested next step. v0
   does not call an LLM; v0.5 will add an opt-in Bedrock-backed suggester.

### CODEOWNERS is the source of truth for approvers

The composite action reads `.github/CODEOWNERS` (or `CODEOWNERS` /
`docs/CODEOWNERS`) using GitHub's standard semantics:

- Last matching pattern wins per path.
- An approver is eligible iff they cover **every** affected path (the
  intersection across paths).
- Team owners (`@org/team-slug`) are resolved via the GitHub API; the
  workflow's token must have `read:org` scope.
- Paths with no CODEOWNERS rule fail closed: the agent comments asking
  for explicit ownership.

### Required repo configuration

| Repo variable | Purpose |
|---|---|
| `AWS_TF_ROLE_TO_ASSUME` | OIDC role for plan. Trust policy MUST include both `pull_request` and `push:main` refs. |
| `AWS_TF_APPLY_ROLE_TO_ASSUME` | OIDC role for apply. Trust policy SHOULD restrict to the head SHA pattern of approved PRs (or, for simplicity in v0, `repo:<org>/<repo>:*` plus a check-policy gate). |
| `AWS_REGION` | Default region (workflow falls back to `us-east-2`). |

Plus a `CODEOWNERS` file with explicit ownership for every path that may
be touched by a PR. Branch protection on `main` should require the
`devops-agent / plan` check and a CODEOWNERS-required review.

### Caller skeleton

A typical caller's `.github/workflows/infra.yml` (here using `devshop-infra`
as the example) wires these together. Discovery, path-walk, and matrix
fan-out live in the caller; the four reusables are pure plumbing.

```yaml
name: gravity-devops-agent

on:
  pull_request:
    paths:
      - "live/**"
      - "terraform/modules/**"
  pull_request_review:
    types: [submitted]
  workflow_dispatch:
    # devops-agent-apply.yml is dispatched by devops-agent-approve.yml; we
    # expose it here as a workflow_dispatch endpoint to let the approve
    # workflow target it.
    inputs:
      pr_number:        { type: string, required: true }
      head_sha:         { type: string, required: true }
      stacks:           { type: string, required: true }
      approver_login:   { type: string, required: true }
      auto_merge:       { type: boolean, required: false, default: true }
      destroy_confirmed:{ type: boolean, required: false, default: false }

jobs:
  discover:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    outputs:
      stacks: ${{ steps.discover.outputs.stacks }}
      affected-paths: ${{ steps.discover.outputs.paths }}
    steps:
      # ... walk changed files up to nearest terragrunt.hcl / *.tf ...
      # outputs.stacks is a JSON array of {directory, stack_id, engine, destroy}.

  plan:
    if: github.event_name == 'pull_request'
    needs: [discover]
    strategy:
      fail-fast: false
      matrix:
        stack: ${{ fromJson(needs.discover.outputs.stacks) }}
    uses: infinity-constellation/ci-workflows/.github/workflows/devops-agent-plan.yml@main
    with:
      working-directory: ${{ matrix.stack.directory }}
      stack-id: ${{ matrix.stack.stack_id }}
      destroy: ${{ matrix.stack.destroy }}

  approve:
    if: github.event_name == 'pull_request_review'
    # The discover step in the approve flow needs to recompute affected paths
    # from the PR head; in practice you call discover first and feed its
    # outputs into devops-agent-approve.yml.
    uses: infinity-constellation/ci-workflows/.github/workflows/devops-agent-approve.yml@main
    with:
      apply-workflow-ref: .github/workflows/infra.yml
      stacks: ${{ ... }}
      affected-paths: ${{ ... }}

  apply:
    if: github.event_name == 'workflow_dispatch'
    uses: infinity-constellation/ci-workflows/.github/workflows/devops-agent-apply.yml@main
    with:
      pr-number: ${{ fromJson(inputs.pr_number) }}
      head-sha: ${{ inputs.head_sha }}
      stacks: ${{ inputs.stacks }}
      approver-login: ${{ inputs.approver_login }}
      auto-merge: ${{ fromJson(inputs.auto_merge) }}
      destroy-confirmed: ${{ fromJson(inputs.destroy_confirmed) }}
```

A complete reference caller is in `infinity-constellation/devshop-infra`
under `.github/workflows/infra.yml`.

### `devops-agent-plan.yml` — inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `working-directory` | yes | — | Stack directory relative to repo root. |
| `stack-id` | no | derived | Stable identifier; defaults to `working-directory` with `/` → `-`. |
| `terraform-version` | no | `1.15.2` | |
| `terragrunt-version` | no | `0.99.5` | Used only when `terragrunt.hcl` is detected. |
| `aws-role-to-assume` | no | repo var `AWS_TF_ROLE_TO_ASSUME` | OIDC role ARN. |
| `aws-region` | no | repo var `AWS_REGION` then `us-east-2` | |
| `policy-dir` | no | _(empty)_ | Path to a Conftest/OPA policy directory; empty disables the gate. |
| `conftest-version` | no | `0.59.0` | |
| `destroy` | no | `false` | Run `plan -destroy`. |
| `runs-on` | no | `ubuntu-latest` | |

### `devops-agent-approve.yml` — inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `apply-workflow-ref` | yes | — | Workflow path of the apply workflow to dispatch (e.g. `.github/workflows/infra.yml`). |
| `stacks` | yes | — | JSON array of stack descriptors. |
| `affected-paths` | yes | — | Newline-separated repo-relative paths the PR touches. |
| `codeowners-path` | no | auto-detect | Override path to `CODEOWNERS`. |
| `auto-merge` | no | `true` | Forwarded to apply. |
| `destroy-confirmed` | no | `false` | Required when any stack has `destroy=true`. |

### `devops-agent-apply.yml` — inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `pr-number` | yes | — | PR being applied. |
| `head-sha` | yes | — | SHA of the PR head this apply was authorized against. The workflow refuses if the PR head has moved. |
| `stacks` | yes | — | JSON array of `{directory, stack_id, plan_artifact, plan_run_id, engine, destroy}` objects. |
| `approver-login` | yes | — | For audit trail and squash-commit trailer. |
| `terraform-version` | no | `1.15.2` | Should match plan. |
| `terragrunt-version` | no | `0.99.5` | Should match plan. |
| `aws-role-to-assume` | no | repo var `AWS_TF_APPLY_ROLE_TO_ASSUME` then `AWS_TF_ROLE_TO_ASSUME` | |
| `aws-region` | no | as above | |
| `destroy-confirmed` | no | `false` | Must be `true` if any stack has `destroy=true`. |
| `auto-merge` | no | `true` | Enable auto-squash-merge on the PR after a successful apply. |

### `devops-agent-postmortem.yml` — inputs

| Input | Required | Description |
|---|---|---|
| `pr-number` | yes | |
| `apply-run-id` | yes | GH Actions run ID of the failed apply (for log retrieval). |
| `applied-stacks` | no | JSON array of stack_ids that applied before the failure. |
| `failed-stack` | yes | stack_id of the failing stack. |
| `stacks` | yes | Full set of stacks the PR was applying. |
| `approver-login` | yes | Login of the approver (whose approval is being dismissed). |

### Composite action: `devops-agent-resolve-approver`

```yaml
- uses: infinity-constellation/ci-workflows/.github/actions/devops-agent-resolve-approver@main
  id: resolve
  with:
    approver-login: ${{ github.event.review.user.login }}
    affected-paths: |
      live/infinity/aws/sso
      live/infinity/github/gravity-api
```

Outputs: `eligible` (bool string), `reason`, `covering-owners` (JSON
array), `unowned-paths` (JSON array).

### v1 migration plan

When `gravity-agent#1`'s `BridgeBase` ABC merges, this v0 workflow set is
replaced by `bridges/devops/` in the `gravity-agent` process:

| v0 | v1 |
|---|---|
| `devops-agent-plan.yml` | `DevOpsBridge.handle_pull_request_event()` |
| `devops-agent-approve.yml` | `DevOpsBridge.handle_review_event()` (approver resolution via `gateway.check_access`) |
| `devops-agent-apply.yml` | `DevOpsApplyTracker.run()` |
| `devops-agent-postmortem.yml` | `DevOpsBridge.handle_apply_failure()` |
| Composite action | `bridges/devops/codeowners_resolver.py` (still reads CODEOWNERS, but resolves through OPA + `AccessSnapshot`) |
| Job-summary lifecycle events | Avro `agent.status` topic events |
