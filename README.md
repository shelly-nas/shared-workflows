# Shared GitHub Actions

A collection of reusable GitHub Actions for Docker-based CI/CD workflows.

## Available Actions

### Docker Build and Push

Build and push Docker images to a container registry.

**Location:** `docker-build-push`

#### Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `registry` | Container registry URL (e.g., ghcr.io, docker.io) | Yes | - |
| `username` | Registry username | Yes | - |
| `password` | Registry password or token | Yes | - |
| `image-name` | Docker image name (without registry prefix) | Yes | - |
| `tag` | Docker image tag | No | `latest` |
| `context` | Build context path | No | `.` |
| `dockerfile` | Path to Dockerfile | No | `Dockerfile` |
| `build-args` | Build arguments (one per line, KEY=VALUE format) | No | - |
| `push` | Push the image to registry | No | `true` |

#### Outputs

| Output | Description |
|--------|-------------|
| `image` | Full image reference (registry/image:tag) |
| `digest` | Image digest |

#### Example Usage

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build and Push Docker Image
        uses: ShellyNAS/shared-actions/docker-build-push@main
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
          image-name: ${{ github.repository }}
          tag: ${{ github.sha }}
```

---

### Docker Compose Deploy

Deploy applications from a container registry using docker compose.

**Location:** `docker-compose-deploy`

#### Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `registry` | Container registry URL (e.g., ghcr.io, docker.io) | Yes | - |
| `username` | Registry username | Yes | - |
| `password` | Registry password or token | Yes | - |
| `compose-file` | Path to docker-compose file | No | `docker-compose.yml` |
| `project-name` | Docker Compose project name | No | - |
| `env-file` | Path to environment file for docker compose | No | - |
| `services` | Specific services to deploy (space-separated) | No | all services |
| `pull` | Pull images before starting containers | No | `true` |
| `remove-orphans` | Remove containers for services not defined in the compose file | No | `true` |
| `force-recreate` | Force recreation of containers | No | `false` |
| `build` | Build images before starting containers | No | `false` |

#### Outputs

| Output | Description |
|--------|-------------|
| `deployed-services` | List of deployed services |

#### Example Usage

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy with Docker Compose
        uses: ShellyNAS/shared-actions/docker-compose-deploy@main
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
          compose-file: docker-compose.yml
          project-name: my-app
```

---

## Complete CI/CD Example

Here's an example of using both actions together in a workflow:

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image: ${{ steps.build.outputs.image }}
    steps:
      - uses: actions/checkout@v4

      - name: Build and Push
        id: build
        uses: ShellyNAS/shared-actions/docker-build-push@main
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
          image-name: ${{ github.repository }}
          tag: ${{ github.sha }}

  deploy:
    runs-on: self-hosted
    needs: build
    steps:
      - uses: actions/checkout@v4

      - name: Deploy
        uses: ShellyNAS/shared-actions/docker-compose-deploy@main
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
          compose-file: docker-compose.prod.yml
          project-name: my-application
```

## License

MIT