# Shared Workflows

Reusable GitHub Actions workflows that build Docker images and deploy them with
docker compose on the Shelly NAS, and the CI that checks every push. The build
and deploy workflows run on the self-hosted runner `shelly`; CI runs on GitHub's
runners.

| Workflow | What it does |
| -------- | ------------ |
| [`build-push-image.yml`](.github/workflows/build-push-image.yml) | Builds **one** image from a directory's `Dockerfile` and pushes it to the registry |
| [`deploy-production.yml`](.github/workflows/deploy-production.yml) | Copies a compose file and a `.env` to a directory on the NAS, pulls the images and starts the stack |
| [`deploy-acceptance.yml`](.github/workflows/deploy-acceptance.yml) | Deploys a branch to the app's acceptance stack, with a fresh database of schema, seed and demo data |
| [`teardown-acceptance.yml`](.github/workflows/teardown-acceptance.yml) | Removes the acceptance stack when the pull request on it is merged or closed |
| [`ci-node-postgres.yml`](.github/workflows/ci-node-postgres.yml) | CI for an app with a Node server: typecheck, tests, and the schema, seed and migration checks against Postgres |

Every deploy, acc and production, ends by removing what the NAS no longer
uses: images no container references, dangling anonymous volumes and dangling
build cache. See [Cleanup](#cleanup).

The conventions around these workflows (release-please, compose layout, Traefik
labels) live in the `release-deploy` and `docker-traefik` skills of
[shelly-nas/shelly-fundamentals](https://github.com/shelly-nas/shelly-fundamentals).
`shelly-nas/FinanceApp` is the reference implementation.

## Quick start

One `build-push` job per image, then one `deploy` job that waits for all of them:

```yaml
jobs:
  build-server:
    uses: shelly-nas/shared-workflows/.github/workflows/build-push-image.yml@main
    with:
      registry: ${{ vars.REGISTRY_URL }}
      # Comma-separated: the version is what production pins to, latest is a pointer.
      image_tag: ${{ vars.REGISTRY_URL }}/my-app-server:1.2.3,${{ vars.REGISTRY_URL }}/my-app-server:latest
      context: ./server
    secrets:
      registry_password: ${{ secrets.REGISTRY_PASSWORD }}

  deploy:
    needs: [build-server]
    uses: shelly-nas/shared-workflows/.github/workflows/deploy-production.yml@main
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

## `build-push-image.yml`

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

## `deploy-production.yml`

| Input | Required | Default | Description |
| ----- | -------- | ------- | ----------- |
| `registry` | yes | | Registry host; also written to `.env` as `REGISTRY` |
| `deploy_directory` | yes | | Directory on the NAS, by convention `/volume1/docker/<app>` |
| `compose_file` | no | `docker-compose.prod.yaml` | Compose file in the repo; copied to the deploy directory as `docker-compose.yaml`. Note the `.yaml` default - pass it explicitly if yours is `.yml` |
| `image_tag` | no | `latest` | Written to `.env` as `IMAGE_TAG`. Pass a version, not `latest` |
| `runner` | no | `shelly` | Runner label |
| `health_check_delay` | no | `15` | Seconds to wait after `up` before checking the containers |
| `prune_images` | no | `true` | Run the [cleanup](#cleanup) after the deploy |
| `prune_age` | no | *(empty)* | Only prune images older than this, e.g. `168h`. Empty prunes every unused image |
| `acc_directory` | no | *(empty)* | The app's acc directory (`/volume1/docker/<app>-acc`). When set, acc is torn down once its change is in production; see below. The fallback for [`teardown-acceptance.yml`](#teardown-acceptanceyml) |

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
6. Run the [cleanup](#cleanup), also when the deploy failed.
7. With `acc_directory` set: tear down acc if its change is now in production.

**Tearing down acc.** Acc images are tagged `acc-<commit>`, so the running acc
containers tell which commit is on acc. It is read from the image each
container was created from, which survives the tag being removed later. A job
on `ubuntu-latest` then decides whether that change is in production: either the commit itself is in the
deployed history (merge commit), or a merged pull request containing it is
(squash or rebase merge, looked up with the GitHub API). Only then are the acc
containers, networks, compose volumes and the whole `acc_directory` removed,
followed by another cleanup. A pull request that is still under review stays
on acc, and so does acc when no acc containers exist or the lookup fails. The
directory must be absolute and end in `-acc`; anything else is refused. The
next acc deploy recreates it.

Normally acc is already gone by then: [`teardown-acceptance.yml`](#teardown-acceptanceyml)
removes it as soon as its pull request closes. This teardown catches what that
misses, such as a branch put on acc by hand.

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

## `deploy-acceptance.yml`

Deploys a branch (normally an open pull request) to the app's **acceptance**
stack: `<subdomain>-acc.shelly-nas.nl`, in its own directory with its own
containers and database. There is one acc per app; the last deploy wins.
Production is not touched.

| Input | Required | Default | Description |
| ----- | -------- | ------- | ----------- |
| `registry` | yes | | Registry host; also written to `.env` as `REGISTRY` |
| `deploy_directory` | yes | | Acc directory on the NAS. **Must end in `-acc`**, by convention `/volume1/docker/<app>-acc`; anything else is refused |
| `compose_file` | no | `docker-compose.acc.yml` | Acc compose file in the repo; copied as `docker-compose.yaml` |
| `image_tag` | yes | | Written to `.env` as `IMAGE_TAG` |
| `db_service` | no | `db` | Database service name in the acc compose file |
| `demo_seed_file` | no | `database/seed-demo.sql` | SQL loaded after the schema; empty to skip |
| `runner` | no | `shelly` | Runner label |
| `health_check_delay` | no | `15` | Seconds to wait after `up` before checking the containers |
| `prune_images` | no | `true` | Run the [cleanup](#cleanup) after the deploy |
| `prune_age` | no | *(empty)* | Only prune images older than this. Empty prunes every unused image |

Secrets are the same as for `deploy-production.yml`.

Steps:

1. Refuse a `deploy_directory` that does not end in `-acc`.
2. Copy the compose file, write the `.env`, log in, pull, `docker compose down`.
3. **Empty the acc data directory** and start only the database. Its image runs
   `init.sql` and `seed.sql` on the empty directory.
4. Wait until Postgres answers over TCP (it only does once the init scripts are
   done), then load `demo_seed_file` from the checked-out branch.
5. Start the rest of the stack; the server applies its pending migrations.
6. Check the containers like `deploy-production.yml`, log out and delete the `.env`.
7. Run the [cleanup](#cleanup), which removes the previous `acc-<commit>` images.

Because the database is rebuilt on every deploy, data entered on acc does not
survive the next deploy.

```yaml
on:
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened, closed]

concurrency:
  group: acc-deploy
  # A teardown waits for a running deploy instead of cancelling it.
  cancel-in-progress: ${{ github.event.action != 'closed' }}

jobs:
  # build-push jobs tagging the images acc-${{ github.event.pull_request.head.sha }}

  deploy-acc:
    if: github.event.action != 'closed'
    needs: [build-server, build-client, build-db]
    uses: shelly-nas/shared-workflows/.github/workflows/deploy-acceptance.yml@main
    with:
      registry: ${{ vars.REGISTRY_URL }}
      deploy_directory: /volume1/docker/my-app-acc
      image_tag: acc-${{ github.event.pull_request.head.sha }}
    secrets:
      registry_username: ${{ vars.REGISTRY_USERNAME }}
      registry_password: ${{ secrets.REGISTRY_PASSWORD }}
      env_file_content: |
        DB_USER=${{ vars.DB_USER }}
        DB_PASSWORD=${{ secrets.DB_PASSWORD }}
```

## `teardown-acceptance.yml`

Removes the app's acc stack when the pull request on it is merged or closed,
so acc does not use the NAS while nothing is under review. Call it from the
acc workflow on `pull_request: closed`.

| Input | Required | Default | Description |
| ----- | -------- | ------- | ----------- |
| `acc_directory` | yes | | Acc directory on the NAS. **Must end in `-acc`**; anything else is refused |
| `commit` | yes | | Head commit of the closed pull request |
| `runner` | no | `shelly` | Runner label |
| `prune_images` | no | `true` | Run the [cleanup](#cleanup) after the teardown |
| `prune_age` | no | *(empty)* | Only prune images older than this. Empty prunes every unused image |

Acc is only torn down while it **still runs `commit`**: the running containers'
`acc-<commit>` image tag is compared with it. If another pull request or a
branch put there by hand has been deployed since, that one is under review and
acc stays. With no acc containers there is nothing to do. Otherwise the acc
containers, networks, compose volumes and the whole `acc_directory` are
removed, followed by the cleanup. The next acc deploy recreates everything.

Add it to the acc workflow next to the deploy (see the example above for the
`closed` trigger and the concurrency):

```yaml
  teardown-acc:
    if: >-
      github.event.action == 'closed' &&
      !startsWith(github.head_ref, 'release-please--') &&
      github.event.pull_request.head.repo.full_name == github.repository
    uses: shelly-nas/shared-workflows/.github/workflows/teardown-acceptance.yml@main
    with:
      acc_directory: /volume1/docker/my-app-acc
      commit: ${{ github.event.pull_request.head.sha }}
```

## `ci-node-postgres.yml`

The CI from the `release-deploy` standard, for an app with a Node server and
React client. Call it from `.github/workflows/ci.yml` and keep `on:` and
`concurrency:` in the app:

```yaml
on:
  push:
  pull_request:
  workflow_dispatch:

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  ci:
    uses: shelly-nas/shared-workflows/.github/workflows/ci-node-postgres.yml@main
    with:
      db_user: my_user
      db_name: my_db
      seed_check_table: categories
```

| Input | Required | Default | Description |
| ----- | -------- | ------- | ----------- |
| `db_user` | yes | | Database user for the CI Postgres |
| `db_name` | yes | | Database name for the CI Postgres |
| `node_version` | no | `22` | Node.js version |
| `typecheck_projects` | no | `["server", "client"]` | JSON array of directories to run `tsc --noEmit` in |
| `client_directory` | no | `client` | Directory whose `npm test` runs without a database; empty to skip |
| `server_directory` | no | `server` | Built, migrated and tested against Postgres |
| `init_sql` | no | `database/init.sql` | Schema applied to the fresh database |
| `seed_sql` | no | `database/seed.sql` | Seed applied after the schema and once more; empty to skip |
| `seed_check_table` | no | *(empty)* | Table whose row count must not change on the second seed. Empty only checks that it does not fail |
| `migrations_directory` | no | `server/migrations` | Where the `NNN_*.sql` migrations live; checked for duplicate numbers |
| `migrate_command` | no | `node -e "require('./dist/context/migrations').runMigrations()…"` | Run twice in `server_directory` after the build; the second run must be a no-op. Empty to skip |

Jobs: `Typecheck <dir>` per directory, `Client unit tests`, and `Schema and
server tests` (build, duplicate migration numbers, schema and seed, seed again,
migrations twice, `npm test`). The server tests get `DB_HOST`, `DB_PORT`,
`DB_USER`, `DB_PASSWORD`, `DB_NAME` and `DATABASE_URL` in their environment.

A Python backend is not covered yet; keep the CI from the `release-deploy`
skill in the app for now.

## Cleanup

`deploy-production.yml` and `deploy-acceptance.yml` end with the same step (unless `prune_images`
is `false`), and it also runs after a failed deploy. `teardown-acceptance.yml`
runs it after a teardown:

```bash
docker image prune -af                       # images no container uses
docker volume ls -q --filter dangling=true \
  | grep -E '^[0-9a-f]{64}$' | xargs -r docker volume rm   # anonymous volumes only
docker builder prune -f                      # dangling build cache
```

- An image of a running **or stopped** container is kept, so nothing that can
  still start is affected. Everything removed can be pulled again from the
  registry, which is also how a rollback works.
- Only **anonymous** volumes are removed, recognised by their 64-character hex
  name. Named volumes are never pruned, even of stopped stacks and on Docker
  versions where `docker volume prune` would remove them. Acc's own named
  volumes are removed only by an acc teardown.
- The databases live in bind mounts (`./data/postgres`), which no prune touches.
