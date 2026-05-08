# ci-workflows

Shared GitHub Actions reusable workflows for the Infinity Constellation GitHub org.

## Workflows

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

## Flux deploy notifications

Flux Notification Controller resources live in
[`infinity-k8s-fleet`](https://github.com/infinity-constellation/infinity-k8s-fleet)
under `infrastructure/notifications/`. Those resources watch the `apps-infinity-factory`
Kustomization and fire Slack alerts on reconcile success and failure.
