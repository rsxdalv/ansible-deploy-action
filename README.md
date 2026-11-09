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

## Reusable workflow: quadlet-apply (repo-defined Podman services on the host)

`.github/workflows/quadlet-apply.yml` builds the app image with the repo's `deploy/` folder (Quadlet unit
files) embedded as the `dev.ttswebui.deploy` label, pushes it to zot and deploys it through the ansible-hooks
`quadlet-apply` service. No secrets and no entry in ansible-hooks: the deploy waits for your approval in the
dashboard, and the unit files come from the repo.

```yaml
# .github/workflows/deploy.yml in the app repo
name: deploy
on:
  push: {branches: [main]}
  pull_request: {branches: [main]}
  workflow_dispatch:
jobs:
  deploy:
    uses: rsxdalv/ansible-deploy-action/.github/workflows/quadlet-apply.yml@v3
    permissions: {contents: read, id-token: write}
```

`deploy/` holds `service.yml` (`unit: <main unit>`, optional `env_files`, `health_retries`, `secret_prefix`),
`<unit>.container` (or `.container.j2`; `{{ image_ref }}` becomes the deployed image) and any extra
`.network` / `.volume` / `.pod` / `.timer` files, plus an optional SOPS-encrypted `secrets.sops.yaml`.
A repo with several images runs one job per app with `app: <name>` and `deploy-dir: <its folder>`.

## Reusable workflow: k8s-apply (repo-defined apps on k3s)

`.github/workflows/k8s-apply.yml` builds the app image with the repo's `deploy/` folder embedded as the
`dev.ttswebui.deploy` image label, pushes it to zot (OIDC) and deploys it through the ansible-hooks
`k8s-apply` service into the app's own namespace. No secrets and no per-repo registration: the first
deploy of a repo waits for an approval in the ansible.ttswebui.com dashboard.

```yaml
# .github/workflows/k8s-apply.yml in the app repo
name: k8s-apply
on:
  push: {branches: [main]}
  pull_request: {branches: [main]}
  workflow_dispatch:
jobs:
  deploy:
    uses: rsxdalv/ansible-deploy-action/.github/workflows/k8s-apply.yml@v2
    permissions: {contents: read, id-token: write}
```

`deploy/values.yaml` holds values for the shared `charts/app` chart (port, probes, routes on
`<name>.dev.ttswebui.com`, resources, persistence, `scaleToZero`) plus `postgres: {enabled: true}` for
an automatically provisioned database. Encrypt secrets to `deploy/secrets.sops.yaml` with the host age
recipient `age10acwlz77803rce9jlq4dx89ugywqzxxlsjlh0a5d9rayhtqucpqscakjf7` (flat `ENV_NAME: value` map).
