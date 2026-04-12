---
name: golang-github-actions
description: "GitHub Actions workflow patterns — Devbox + Taskfile CI, Docker deploy, multi-service builds, auto-release, path filtering, self-hosted runners. Use when creating or modifying GitHub Actions workflows."
metadata:
  author: openserbia
  version: "1.0.0"
---

# GitHub Actions Workflows

Patterns for CI/CD workflows using Devbox for reproducible tooling and Taskfile for task execution.

## Core Principles

- **Devbox for all tooling** — never install tools manually in CI, use `devbox run -- task <name>`
- **Taskfile for all commands** — workflows call tasks, not raw commands
- **Concurrency control** — always cancel in-progress runs on new push
- **Self-hosted for deploy** — use `self-hosted` runner for Docker build/push, `ubuntu-latest` for lint/test
- **SHA-tagged images** — tag Docker images with `${{ github.sha }}`

## Always Include

Every workflow MUST have concurrency control:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

## Workflow Templates

### Simple Deploy — Single Service

For services with one Dockerfile (bot, data-provider, tg-antispam):

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build-and-deploy:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v6

      - name: Build and push Docker image
        run: devbox run -- task docker:build IMAGE_TAG=${{ github.sha }}
```

### Matrix Deploy — Multiple Services

For monorepos with multiple services (tg-statistic: bot, svc, web):

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build-and-deploy:
    runs-on: self-hosted
    strategy:
      matrix:
        service: [bot, svc, web]
    steps:
      - uses: actions/checkout@v6

      - name: Build and push ${{ matrix.service }} Docker image
        run: devbox run -- task -d ${{ matrix.service }} docker:build IMAGE_TAG=${{ github.sha }}
```

`task -d <dir>` runs the task from within the service directory.

### Path-Filtered Deploy — Build Only What Changed

For monorepos where you only want to build changed services:

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  changes:
    runs-on: self-hosted
    outputs:
      backend: ${{ steps.filter.outputs.backend }}
      frontend: ${{ steps.filter.outputs.frontend }}
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 2

      - uses: dorny/paths-filter@v4.0.0
        id: filter
        with:
          filters: |
            backend:
              - 'svc/**'
              - 'Taskfile.yml'
              - 'devbox.json'
              - '.github/workflows/**'
            frontend:
              - 'web/**'
              - 'Taskfile.yml'
              - 'devbox.json'
              - '.github/workflows/**'

  backend:
    needs: changes
    if: needs.changes.outputs.backend == 'true'
    runs-on: self-hosted
    environment: production
    steps:
      - uses: actions/checkout@v6

      - name: Build and push Docker image
        run: devbox run -- task svc:docker:build IMAGE_TAG=${{ github.sha }}

  frontend:
    needs: changes
    if: needs.changes.outputs.frontend == 'true'
    runs-on: self-hosted
    environment: production
    steps:
      - uses: actions/checkout@v6

      - name: Build and push Docker image
        run: devbox run -- task web:docker:build IMAGE_TAG=${{ github.sha }}
```

Always include `Taskfile.yml`, `devbox.json`, and `.github/workflows/**` in every filter — changes to tooling or CI should trigger all builds.

### Lint & Test — PR Checks

For running linters and tests on push and PR (uses `ubuntu-latest` with Devbox install action):

```yaml
name: Lint & Test

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - name: Install Devbox
        uses: jetify-com/devbox-install-action@v0.12.0

      - name: Lint
        run: devbox run -- task lint

      - name: Test
        run: devbox run -- task test
```

### Auto-Release with Version Increment

Creates a GitHub release with auto-incremented patch version on push to main:

```yaml
name: Release

on:
  push:
    branches: [main]
    paths:
      - "src/**"
      - "Taskfile.yml"
      - "devbox.json"
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0

      - name: Determine next version
        id: version
        run: |
          LATEST=$(git tag --sort=-v:refname | head -1)
          if [ -z "$LATEST" ]; then
            echo "tag=v1.0.0" >> "$GITHUB_OUTPUT"
          else
            MAJOR=$(echo "$LATEST" | sed 's/v//' | cut -d. -f1)
            MINOR=$(echo "$LATEST" | sed 's/v//' | cut -d. -f2)
            PATCH=$(echo "$LATEST" | sed 's/v//' | cut -d. -f3)
            PATCH=$((PATCH + 1))
            echo "tag=v${MAJOR}.${MINOR}.${PATCH}" >> "$GITHUB_OUTPUT"
          fi
          echo "prev=$LATEST" >> "$GITHUB_OUTPUT"

      - name: Create tag and release
        run: |
          git tag ${{ steps.version.outputs.tag }}
          git push origin ${{ steps.version.outputs.tag }}

      - uses: softprops/action-gh-release@v2
        with:
          tag_name: ${{ steps.version.outputs.tag }}
          generate_release_notes: true
```

For minor/major bumps, create the tag manually — auto-increment continues from there.

### Cloudflare Workers Deploy

For frontend/dashboard deployed to Cloudflare:

```yaml
deploy:
  runs-on: self-hosted
  environment: production
  env:
    HOME: ${{ github.workspace }}
    CI: "true"
  steps:
    - uses: actions/checkout@v6

    - name: Build
      run: devbox run -- task deps build

    - name: Deploy to Cloudflare Workers
      env:
        CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
        CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
      run: devbox run -- task deploy
```

Set `HOME: ${{ github.workspace }}` when Cloudflare wrangler needs a writable home directory.

### SSH for Private Dependencies

When Docker build needs access to private repos:

```yaml
steps:
  - name: Setup SSH agent
    run: |
      eval $(ssh-agent -s)
      ssh-add /var/lib/github-runner/ssh-key
      echo "SSH_AUTH_SOCK=$SSH_AUTH_SOCK" >> $GITHUB_ENV
      echo "SSH_AGENT_PID=$SSH_AGENT_PID" >> $GITHUB_ENV

  - uses: actions/checkout@v6

  - name: Build with SSH
    run: devbox run -- task docker:build DOCKER_SSH="--ssh default" IMAGE_TAG=${{ github.sha }}
```

## Runner Selection

| Use case            | Runner          | Why                                            |
| ------------------- | --------------- | ---------------------------------------------- |
| Docker build/push   | `self-hosted`   | Needs Docker daemon, registry access, SSH keys |
| Lint, test, release | `ubuntu-latest` | Stateless, no special access needed            |
| Cloudflare deploy   | `self-hosted`   | May need secrets, network access               |

## Devbox in CI

Devbox is pre-installed on the self-hosted runner. Just call `devbox run` directly — no install action needed.

```yaml
- run: devbox run -- task build
```

Only use the install action if running on GitHub-hosted runners (`ubuntu-latest`):

```yaml
- uses: jetify-com/devbox-install-action@v0.12.0
- run: devbox run -- task build
```

## Common Patterns

### Passing build metadata

```yaml
- run: devbox run -- task docker:build IMAGE_TAG=${{ github.sha }}
```

The Taskfile receives `IMAGE_TAG` as a variable and uses it for Docker tagging.

### Job dependencies

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    # ...

  deploy:
    needs: lint
    runs-on: self-hosted
    # ...
```

### Permissions

Only request what you need:

```yaml
permissions:
  contents: write # for creating releases/tags
```

### Path triggers

```yaml
on:
  push:
    branches: [main]
    paths:
      - "src/**"
      - "Taskfile.yml"
      - "devbox.json"
      - ".github/workflows/**"
```

Always include tooling and workflow files in path triggers.

## Anti-Patterns

| Bad                                                 | Good                                        |
| --------------------------------------------------- | ------------------------------------------- |
| Installing tools with `apt-get` or `npm install -g` | Use Devbox — reproducible, cached           |
| Running raw `go build`, `golangci-lint` in workflow | Use `devbox run -- task build`              |
| Missing concurrency control                         | Always add `cancel-in-progress: true`       |
| Using `ubuntu-latest` for Docker builds             | Use `self-hosted` with Docker pre-installed |
| Hardcoding image tags                               | Use `${{ github.sha }}`                     |
| Building all services on every push                 | Use path filters or `dorny/paths-filter`    |
| Missing `workflow_dispatch` on deploy workflows     | Always allow manual triggers                |
| `fetch-depth: 1` when needing git history           | Use `fetch-depth: 0` for tags/release notes |
