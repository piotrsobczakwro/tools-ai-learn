# CI/CD

This directory contains configurations for CI/CD platforms: **GitHub Actions** workflow examples and **GitLab CI/CD** pipeline examples, along with a self-hosted **GitLab Runner** via `podman-compose`.

## Table of Contents

- [Overview](#overview)
- [GitHub Actions](#github-actions)
- [GitLab CI/CD](#gitlab-cicd)
- [GitLab Runner (Self-hosted)](#gitlab-runner-self-hosted)
- [Quick Start](#quick-start)
- [Pipeline Features](#pipeline-features)
- [Best Practices](#best-practices)

---

## Overview

| Platform         | Type             | Where It Runs          |
|------------------|------------------|------------------------|
| GitHub Actions   | SaaS + self-hosted | GitHub-managed or your infrastructure |
| GitLab CI/CD     | SaaS + self-hosted | GitLab-managed or your infrastructure |
| GitLab Runner    | Self-hosted agent  | Your server / container |

---

## GitHub Actions

GitHub Actions is a SaaS CI/CD platform. There is no standalone "GitHub Actions server" — pipelines run on GitHub-managed runners or self-hosted runners that connect **back** to `github.com`.

### Workflow File

``.github/workflows/example.yml`` — a complete multi-job CI/CD pipeline including:

| Job              | Description                                       |
|------------------|---------------------------------------------------|
| `lint`           | ESLint + Prettier format check                    |
| `test`           | Unit + integration tests with Postgres and Redis  |
| `security`       | Trivy vulnerability scan → GitHub Security tab    |
| `build`          | Build and push Docker image to GHCR               |
| `deploy-staging` | SSH deploy to staging (main branch only)          |

### Self-Hosted GitHub Actions Runner

To run GitHub Actions on your own infrastructure:

```bash
# Download the runner
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.311.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz
tar xzf actions-runner-linux-x64-2.311.0.tar.gz

# Configure (get token from GitHub repo → Settings → Actions → Runners)
./config.sh --url https://github.com/YOUR_ORG/YOUR_REPO --token YOUR_TOKEN

# Run as a service
sudo ./svc.sh install && sudo ./svc.sh start
```

### Required Secrets (GitHub)

| Secret              | Description                        |
|---------------------|------------------------------------|
| `STAGING_HOST`      | Staging server hostname/IP         |
| `STAGING_USER`      | SSH username on staging            |
| `STAGING_SSH_KEY`   | Private SSH key (RSA/Ed25519)      |

---

## GitLab CI/CD

`gitlab-ci.yml` — a complete pipeline. Copy this to your GitLab repo root as `.gitlab-ci.yml`.

### Pipeline Stages

| Stage      | Jobs                                         |
|------------|----------------------------------------------|
| `validate` | `lint`, `typecheck`                          |
| `test`     | `unit-tests`, `integration-tests`            |
| `security` | `dependency-scanning`, `sast`, `secret-detection` |
| `build`    | `build-app`, `build-docker`                  |
| `deploy`   | `deploy-staging`, `deploy-production` (manual)|

### Required CI/CD Variables (GitLab)

Set these in **Settings → CI/CD → Variables**:

| Variable                 | Description                        |
|--------------------------|------------------------------------|
| `CI_REGISTRY`            | Auto-set by GitLab                 |
| `CI_REGISTRY_USER`       | Auto-set by GitLab                 |
| `CI_REGISTRY_PASSWORD`   | Auto-set by GitLab                 |
| `STAGING_SSH_PRIVATE_KEY`| Private key for staging deploy     |
| `STAGING_HOST_KEY`       | SSH host key of staging server     |
| `STAGING_USER`           | SSH username on staging            |
| `STAGING_HOST`           | Staging server hostname/IP         |
| `PROD_SSH_PRIVATE_KEY`   | Private key for production deploy  |
| `PROD_HOST_KEY`          | SSH host key of production server  |
| `PROD_USER`              | SSH username on production         |
| `PROD_HOST`              | Production server hostname/IP      |

---

## GitLab Runner (Self-hosted)

The `podman-compose.yml` in this directory runs a **GitLab Runner** container that can execute pipeline jobs on your local machine.

### Start the Runner

```bash
podman-compose up -d
```

### Register the Runner

```bash
podman-compose exec gitlab-runner \
  gitlab-runner register \
    --url https://gitlab.com \
    --registration-token <YOUR_PROJECT_TOKEN> \
    --executor docker \
    --docker-image docker:latest \
    --description "local-podman-runner" \
    --tag-list "docker,local" \
    --run-untagged true \
    --locked false
```

> Get your registration token from **GitLab → Project → Settings → CI/CD → Runners**

### Verify Registration

```bash
podman-compose exec gitlab-runner gitlab-runner list
```

---

## Quick Start

```bash
# Start GitLab Runner
podman-compose up -d

# Check runner status
podman-compose ps
podman-compose logs -f gitlab-runner

# Stop
podman-compose down
```

---

## Pipeline Features

### GitHub Actions Features
- ✅ Dependency caching (`actions/cache`)
- ✅ Matrix builds (test across multiple versions)
- ✅ Service containers (Postgres, Redis)
- ✅ OIDC-based cloud deployments (no long-lived secrets)
- ✅ Concurrency groups (cancel stale runs)
- ✅ GitHub Container Registry (GHCR) push

### GitLab CI/CD Features
- ✅ YAML anchors and templates (DRY configuration)
- ✅ Built-in artifact management
- ✅ Service containers for tests
- ✅ Coverage reporting
- ✅ JUnit test reporting in MR UI
- ✅ Manual approval for production deployments
- ✅ Environment tracking

---

## Best Practices

1. **Never commit secrets** — use encrypted CI/CD variables or OIDC
2. **Pin action versions** — use `@v4` not `@main` to avoid supply chain attacks
3. **Fail fast** — put lint/typecheck before slow test jobs
4. **Cache dependencies** — speeds up jobs significantly
5. **Use environments** — GitHub/GitLab environments add protection rules and tracking
6. **Test in parallel** — split unit and integration tests into separate jobs
7. **Scan for vulnerabilities** — add Trivy or Snyk to every pipeline
8. **Keep pipelines fast** — aim for < 10 min feedback on PRs
