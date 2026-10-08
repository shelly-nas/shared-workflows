# Shared Workflows

Reusable GitHub Actions workflows that build Docker images and deploy them with
docker compose on the Shelly NAS. Both run on the self-hosted runner `shelly`.

| Workflow | What it does |
| -------- | ------------ |
| [`build-push.yml`](.github/workflows/build-push.yml) | Builds **one** image from a directory's `Dockerfile` and pushes it to the registry |
| [`deploy.yml`](.github/workflows/deploy.yml) | Copies a compose file and a `.env` to a directory on the NAS, pulls the images and starts the stack |

The conventions around these workflows (release-please, compose layout, Traefik
labels) live in the `release-deploy` and `docker-traefik` skills of
[shelly-nas/shelly-fundamentals](https://github.com/shelly-nas/shelly-fundamentals).
`shelly-nas/FinanceApp` is the reference implementation.

## Quick start

One `build-push` job per image, then one `deploy` job that waits for all of them:

```yaml
jobs:
  build-server:
    uses: shelly-nas/shared-workflows/.github/workflows/build-push.yml@main
    with:
      registry: ${{ vars.REGISTRY_URL }}
      # Comma-separated: the version is what production pins to, latest is a pointer.
      image_tag: ${{ vars.REGISTRY_URL }}/my-app-server:1.2.3,${{ vars.REGISTRY_URL }}/my-app-server:latest
      context: ./server
    secrets:
      registry_password: ${{ secrets.REGISTRY_PASSWORD }}

  deploy:
    needs: [build-server]
    uses: shelly-nas/shared-workflows/.github/workflows/deploy.yml@main
    with:
      registry: ${{ vars.REGISTRY_URL }}
      deploy_directory: /volume1/docker/my-app
      compose_file: docker-compose.prod.yml
      image_tag: 1.2.3
    secrets:
      registry_username: ${{ vars.REGISTRY_USERNAME }}
      registry_password: ${{ secrets.REGISTRY_PASSWORD }}
      env_file_content: |
        DB_USER=${{ vars.DB_USER }}
        DB_PASSWORD=${{ secrets.DB_PASSWORD }}
```

In practice the version comes from release-please and the jobs only run on the
release merge; see FinanceApp's
[`deploy.yml`](https://github.com/shelly-nas/FinanceApp/blob/main/.github/workflows/deploy.yml)
for the complete pipeline.

## `build-push.yml`

| Input | Required | Default | Description |
| ----- | -------- | ------- | ----------- |
| `registry` | yes | | Registry host to log in to |
| `image_tag` | yes | | Full image reference(s), `registry/name:tag`. Several tags separated by commas |
| `context` | yes | | Directory containing the `Dockerfile`; also the build context |
| `runner` | no | `shelly` | Runner label |

| Secret | Required | Description |
| ------ | -------- | ----------- |
| `registry_password` | yes | Registry password |

The registry **username** is read from the calling repository's
`vars.REGISTRY_USERNAME`, not passed as an input, so that variable must exist
in every repo that uses this workflow.

Steps: log in, `docker image prune -af`, build, push, remove the local image,
log out.

## `deploy.yml`

| Input | Required | Default | Description |
| ----- | -------- | ------- | ----------- |
| `registry` | yes | | Registry host; also written to `.env` as `REGISTRY` |
| `deploy_directory` | yes | | Directory on the NAS, by convention `/volume1/docker/<app>` |
| `compose_file` | no | `docker-compose.prod.yaml` | Compose file in the repo; copied to the deploy directory as `docker-compose.yaml`. Note the `.yaml` default - pass it explicitly if yours is `.yml` |
| `image_tag` | no | `latest` | Written to `.env` as `IMAGE_TAG`. Pass a version, not `latest` |
| `runner` | no | `shelly` | Runner label |
| `health_check_delay` | no | `15` | Seconds to wait after `up` before checking the containers |
| `prune_images` | no | `true` | Prune old images after a successful deploy |
| `prune_age` | no | `168h` | Age filter for that prune |

| Secret | Required | Description |
| ------ | -------- | ----------- |
| `registry_username` | yes | Registry username |
| `registry_password` | yes | Registry password |
| `env_file_content` | no | Extra `KEY=VALUE` lines appended to the `.env` |

Steps:

1. Copy the compose file to `deploy_directory/docker-compose.yaml`.
2. Write `.env` with `REGISTRY`, `IMAGE_TAG` and `env_file_content`.
3. Log in, `docker compose down`, `docker compose pull`, `docker compose up -d`.
4. Wait `health_check_delay` seconds, then fail if any service is not running
   (with the last 100 log lines of each service on failure).
5. Log out and **delete the `.env`**.
6. Prune images older than `prune_age`.

Things to know:

- **The `.env` does not survive the deploy.** Running containers keep their
  environment and `restart: always` brings them back after a reboot, but a manual
  `docker compose up` in the deploy directory fails on the required variables. To
  roll back or restart by hand, recreate the `.env` first, or rerun the deploy
  with the earlier version.
- **The stack is stopped before the new images are pulled**, so there is a short
  outage on every deploy, longer if the pull is slow.
- **The check means "running", not "healthy".** Give services a `healthcheck` in
  the compose file and use `depends_on: condition: service_healthy` so a broken
  start shows up as a restarting container.
- Only the compose file and the `.env` reach the NAS. Anything else a container
  needs (schema, config) must be baked into its image.
