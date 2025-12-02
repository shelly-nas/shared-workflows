# Shared GitHub Actions

Reusable GitHub Actions and Workflows for Docker-based CI/CD pipelines.

## Quick Start

```yaml
name: Build & Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    uses: shelly-nas/shared-actions/.github/workflows/build-push.yml@main
    with:
      registry: ${{ vars.REGISTRY_URL }}
      images: '[{"name": "my-app", "context": "."}]'
    secrets:
      registry-username: ${{ vars.REGISTRY_USERNAME }}
      registry-password: ${{ secrets.REGISTRY_PASSWORD }}

  deploy:
    needs: build
    uses: shelly-nas/shared-actions/.github/workflows/deploy.yml@main
    with:
      registry: ${{ vars.REGISTRY_URL }}
      deploy-directory: /opt/my-app
      env-variables: |
        DB_HOST=localhost
        API_KEY=${{ secrets.API_KEY }}
    secrets:
      registry-username: ${{ vars.REGISTRY_USERNAME }}
      registry-password: ${{ secrets.REGISTRY_PASSWORD }}
```

## Workflows

| Workflow | Description |
| -------- | ----------- |
| `build-push.yml` | Build and push Docker images in parallel |
| `deploy.yml` | Deploy with docker-compose, health checks, and cleanup |
