# deploy-action

Minimal GitHub composite action that deploys a service via an
OIDC-authenticated webhook (no static secrets, no SSH keys).

## Usage

```yaml
steps:
  - uses: rsxdalv/deploy-action@v1
    with:
      webhook-url: ${{ secrets.DEPLOY_WEBHOOK_URL }}  # required
      service: my-service                              # optional, defaults to repo name
      version: ${{ steps.build.outputs.version }}     # optional, defaults to 'latest'
```

The calling job must have:

```yaml
permissions:
  id-token: write   # mint the OIDC JWT
  contents: read
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `webhook-url` | yes | — | Base URL of the deployment webhook |
| `service` | no | repo name | Service name as registered on the server |
| `version` | no | `latest` | Version string to deploy |

## Outputs

| Output | Description |
|--------|-------------|
| `response` | JSON response body from the webhook |

## How it works

1. Requests a short-lived OIDC JWT from GitHub (`id-token: write` permission)
2. `POST`s to `{webhook-url}/deploy/{service}` with `Authorization: Bearer <jwt>`
3. The server verifies the JWT, checks the repo is in its scope map, and runs the playbook
