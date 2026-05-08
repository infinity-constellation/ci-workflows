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
- [`notify-release.yml`](#notify-releaseyml--slack-release-notification) — Slack release notification.

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
