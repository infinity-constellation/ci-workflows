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
- [`terragrunt-plan.yml`](#terragrunt-planyml--terragrunt-plan-on-pr) — Plan a Terragrunt stack on a PR; post sticky comment, upload artifact, optional Conftest gate.
- [`terragrunt-apply.yml`](#terragrunt-applyyml--terragrunt-apply-from-saved-plan) — Apply a saved Terragrunt plan after merge to `main`.

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

### `terragrunt-plan.yml` — Terragrunt plan on PR

Runs `terragrunt plan` against a single stack, uploads the binary plan as a
workflow artifact, and posts (or updates) a sticky PR comment with the
human-readable plan. Optionally runs a Conftest/OPA policy gate against the
JSON-rendered plan.

The companion `terragrunt-apply.yml` consumes the artifact this workflow
uploads.

This workflow exists because Terragrunt apply/destroy from developer laptops
is the documented failure mode behind the 2026-05-12 `gravity-{web,api,agent}`
incident; see
[ADR-0009](https://github.com/infinity-constellation/gravity-docs/blob/main/decisions/ADR-0009-iac-orchestration.md)
in `gravity-docs` (local-only) and the destroy-guardrails section in
`devshop-infra/AGENTS.md`.

**Usage — typical caller in `devshop-infra`**

```yaml
plan-stack:
  uses: infinity-constellation/ci-workflows/.github/workflows/terragrunt-plan.yml@main
  with:
    working-directory: live/infinity/aws/sso
    policy-dir: policies/conftest
  permissions:
    contents: read
    id-token: write
    pull-requests: write
```

**Usage — destroy plan**

```yaml
plan-destroy-stack:
  if: contains(github.event.pull_request.labels.*.name, 'tf:destroy')
  uses: infinity-constellation/ci-workflows/.github/workflows/terragrunt-plan.yml@main
  with:
    working-directory: live/infinity/aws/staging/us-east-2/some-old-db
    destroy: true
    policy-dir: policies/conftest
```

**Inputs**

| Input | Required | Default | Description |
|---|---|---|---|
| `working-directory` | yes | — | Terragrunt stack directory relative to repo root. |
| `stack-id` | no | derived | Stable identifier for artifacts and PR comments. Defaults to `working-directory` with `/` replaced by `-`. |
| `terraform-version` | no | `1.14.8` | Terraform version. |
| `terragrunt-version` | no | `0.99.5` | Terragrunt version. |
| `aws-role-to-assume` | no | repo var `AWS_TF_ROLE_TO_ASSUME` | OIDC role ARN. |
| `aws-region` | no | repo var `AWS_REGION`, then `us-east-2` | AWS region. |
| `policy-dir` | no | _(empty)_ | Conftest/OPA policy directory. Empty disables the gate. |
| `conftest-version` | no | `0.59.0` | Conftest version when policy-dir is set. |
| `destroy` | no | `false` | Run `terragrunt plan -destroy`. |
| `runs-on` | no | `ubuntu-latest` | Runner label. |

**Outputs**

| Output | Description |
|---|---|
| `plan-artifact` | Name of the uploaded artifact, suitable for `terragrunt-apply.yml` `plan-artifact-name`. |
| `plan-summary-path` | Path within the artifact to `plan.txt`. |
| `stack-id` | Resolved stack identifier. |
| `has-changes` | `"true"` if the plan reports any resource changes. |

**Required repo configuration**

- Repository variable `AWS_TF_ROLE_TO_ASSUME` (or per-call input) — ARN of an
  IAM role trusted by the GitHub OIDC provider for read-mostly Terraform
  operations. The role's trust policy MUST include both `pull_request` and
  `push:main` refs.
- AWS S3 + DynamoDB Terraform state backend reachable from the role's
  permissions. The role needs at minimum: `s3:GetObject`/`PutObject`/`ListBucket`
  on the state bucket, `dynamodb:GetItem`/`PutItem`/`DeleteItem` on the lock
  table, plus read-only access to the AWS APIs the stack queries.

### `terragrunt-apply.yml` — Terragrunt apply from saved plan

Applies a previously-generated `terragrunt plan` artifact against a single
stack. Designed to be invoked on a `push: branches: [main]` event after a PR
that ran `terragrunt-plan.yml` is merged.

Refuses to apply unless:

- The plan artifact is downloadable.
- When `destroy: true`, `destroy-confirmed: true` is also set (wire this to a
  PR-label check or environment approval in the caller).
- When `destroy: true`, the saved plan contains only delete actions.

**Usage — typical caller in `devshop-infra`**

```yaml
apply-stack:
  uses: infinity-constellation/ci-workflows/.github/workflows/terragrunt-apply.yml@main
  with:
    working-directory: live/infinity/aws/sso
    plan-artifact-name: ${{ needs.plan.outputs.plan-artifact }}
    plan-run-id: ${{ needs.plan.outputs.plan-run-id }}
  permissions:
    contents: read
    id-token: write
```

**Inputs**

| Input | Required | Default | Description |
|---|---|---|---|
| `working-directory` | yes | — | Terragrunt stack directory; must match the plan. |
| `stack-id` | no | derived | Stable identifier. |
| `plan-artifact-name` | yes | — | Name of the artifact from `terragrunt-plan.yml`. |
| `plan-run-id` | no | _(current run)_ | Run ID of the workflow that produced the artifact. Required when downloading from a prior run (e.g. the merged PR's plan). |
| `terraform-version` | no | `1.14.8` | Should match plan workflow. |
| `terragrunt-version` | no | `0.99.5` | Should match plan workflow. |
| `aws-role-to-assume` | no | repo var `AWS_TF_APPLY_ROLE_TO_ASSUME` then `AWS_TF_ROLE_TO_ASSUME` | OIDC role ARN. Apply prefers a dedicated apply role. |
| `aws-region` | no | repo var `AWS_REGION`, then `us-east-2` | AWS region. |
| `destroy` | no | `false` | The saved plan must be a `-destroy` plan. |
| `destroy-confirmed` | no | `false` | Must be `true` to allow a destroy apply. |
| `runs-on` | no | `ubuntu-latest` | Runner label. |

**Required repo configuration**

- Repository variable `AWS_TF_APPLY_ROLE_TO_ASSUME` — distinct ARN for apply,
  trusted only on `ref:refs/heads/main` (or your protected branch). PR refs
  MUST NOT have apply trust.
- Branch protection on `main` requiring the plan check to pass and at least
  one approving review.
