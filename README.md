# ci-workflows

Shared GitHub Actions reusable workflows for the Infinity Constellation GitHub org.

## Workflows

- [`python-uv-lint.yml`](#python-uv-lintyml--lint-a-python-uv-project) — Lint a Python `uv` project (ruff, optionally black/mypy).
- [`python-uv-build.yml`](#python-uv-buildyml--build-and-test-a-python-uv-project) — Install dependencies, run tests, and optionally build a Python `uv` project.
- [`temporal-replay-test.yml`](#temporal-replay-testyml--temporal-workflow-replay-tests) — Run Temporal workflow replay tests for a Python `uv` project.
- [`notify-release.yml`](#notify-releaseyml--slack-release-notification) — Slack release notification.

### `python-uv-lint.yml` — Lint a Python `uv` project

Runs `ruff check` (and optionally `ruff format --check`, `black --check`, and
`mypy`) against a Python project managed with [`uv`](https://docs.astral.sh/uv/).

**Usage**

```yaml
lint:
  uses: infinity-constellation/ci-workflows/.github/workflows/python-uv-lint.yml@main
  with:
    python-version: "3.12"
    black-check: true
```

**Inputs**

| Input | Required | Default | Description |
|---|---|---|---|
| `python-version` | no | `3.12` | Python version to install via `uv python install`. |
| `uv-version` | no | `latest` | uv version passed to `astral-sh/setup-uv`. |
| `working-directory` | no | `.` | Directory containing `pyproject.toml`. |
| `install-command` | no | `uv sync --all-extras --dev` | Command used to install dependencies. |
| `ruff-check` | no | `true` | Run `uv run ruff check`. |
| `ruff-check-paths` | no | `.` | Paths passed to `ruff check`. |
| `ruff-format-check` | no | `false` | Run `uv run ruff format --check`. |
| `ruff-format-paths` | no | `.` | Paths passed to `ruff format --check`. |
| `black-check` | no | `false` | Run `uv run black --check`. |
| `black-paths` | no | `.` | Paths passed to `black --check`. |
| `mypy-check` | no | `false` | Run `uv run mypy`. |
| `mypy-paths` | no | `.` | Paths passed to `mypy`. |
| `runs-on` | no | `ubuntu-latest` | Runner label. |

### `python-uv-build.yml` — Build and test a Python `uv` project

Installs dependencies, runs tests, and optionally runs a build step (e.g.
`uv build`).

**Usage**

```yaml
build:
  uses: infinity-constellation/ci-workflows/.github/workflows/python-uv-build.yml@main
  with:
    python-version: "3.12"
    test-command: "uv run python -m pytest -v"
```

**Inputs**

| Input | Required | Default | Description |
|---|---|---|---|
| `python-version` | no | `3.12` | Python version to install via `uv python install`. |
| `uv-version` | no | `latest` | uv version passed to `astral-sh/setup-uv`. |
| `working-directory` | no | `.` | Directory containing `pyproject.toml`. |
| `install-command` | no | `uv sync --all-extras --dev` | Command used to install dependencies. |
| `test-command` | no | `uv run python -m pytest -v` | Command used to run tests. |
| `build-command` | no | _(empty)_ | Optional command run after tests (e.g. `uv build`). |
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
| `install-command` | no | `uv sync --all-extras --dev` | Command used to install dependencies. |
| `replay-command` | no | `uv run python -m pytest tests/replay -v` | Command used to run replay tests. |
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
