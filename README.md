# tools-ai-learn

A hands-on reference repository for **infrastructure tooling**, organized by category. Each category has its own `podman-compose.yml` to spin up all services locally, plus a README with configuration details, usage examples, and best practices.

All services run via **Podman** and **podman-compose** — a daemonless, rootless alternative to Docker.

---

## Table of Contents

- [Categories](#categories)
- [Folder Structure](#folder-structure)
- [What is Podman?](#what-is-podman)
- [Installation](#installation)
  - [Linux](#linux)
  - [macOS](#macos)
  - [Windows](#windows)
- [Installing podman-compose](#installing-podman-compose)
- [Quick Start (Any Category)](#quick-start-any-category)
- [Category Details](#category-details)
  - [Load Balancers](#load-balancers)
  - [Monitoring & Logging](#monitoring--logging)
  - [CI/CD](#cicd)
  - [Databases](#databases)
  - [Caching](#caching)
  - [Message Brokers](#message-brokers)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)

---

## Categories

| Folder                | Tools                                                               |
|-----------------------|---------------------------------------------------------------------|
| `load-balancers/`     | HAProxy, Envoy, LVS (notes)                                        |
| `monitoring-logging/` | Grafana, Graylog, ELK (Elasticsearch + Logstash + Kibana), Jaeger  |
| `ci-cd/`              | GitHub Actions (workflow examples), GitLab CI/CD, GitLab Runner    |
| `databases/`          | MySQL, PostgreSQL, Percona XtraDB Cluster, Elasticsearch, Vitess   |
| `caching/`            | Redis, Memcached                                                   |
| `message-brokers/`    | RabbitMQ, Kafka, Google PubSub emulator                            |

---

## Folder Structure

```
tools-ai-learn/
│
├── README.md                        ← You are here
│
├── load-balancers/
│   ├── podman-compose.yml           # HAProxy + Envoy + backend services
│   ├── haproxy.cfg                  # HAProxy configuration
│   ├── envoy.yaml                   # Envoy proxy configuration
│   └── README.md
│
├── monitoring-logging/
│   ├── podman-compose.yml           # Grafana + ELK + Graylog + Jaeger
│   ├── logstash/
│   │   └── pipeline/
│   │       └── logstash.conf        # Logstash pipeline definition
│   └── README.md
│
├── ci-cd/
│   ├── podman-compose.yml           # GitLab Runner
│   ├── .github/
│   │   └── workflows/
│   │       └── example.yml         # GitHub Actions workflow example
│   ├── gitlab-ci.yml               # GitLab CI/CD pipeline example
│   └── README.md
│
├── databases/
│   ├── podman-compose.yml           # MySQL + PostgreSQL + PXC + ES + Vitess
│   └── README.md
│
├── caching/
│   ├── podman-compose.yml           # Redis + Redis Sentinel + Memcached
│   └── README.md
│
└── message-brokers/
    ├── podman-compose.yml           # RabbitMQ + Kafka + PubSub emulator
    └── README.md
```

---

## What is Podman?

[Podman](https://podman.io/) is an OCI-compliant container engine developed by Red Hat. Key differences from Docker:

| Feature                  | Podman                          | Docker                          |
|--------------------------|---------------------------------|---------------------------------|
| **Daemon**               | Daemonless                      | Requires `dockerd` daemon       |
| **Root access**          | Rootless by default             | Requires root (or `docker` group) |
| **CLI compatibility**    | Drop-in Docker CLI replacement  | Docker CLI                      |
| **Compose support**      | `podman-compose` (or Podman v4 `docker-compose` compatible) | `docker-compose` / Docker Compose v2 |
| **Kubernetes output**    | `podman generate kube`          | Not built-in                    |
| **Security**             | Each container in its own user namespace | Shared daemon socket (escalation risk) |

Podman images are stored in the same OCI format as Docker — you can use any image from Docker Hub, Quay.io, or GHCR.

---

## Installation

### Linux

#### Fedora / RHEL / CentOS Stream
```bash
sudo dnf install -y podman
```

#### Ubuntu / Debian
```bash
sudo apt-get update
sudo apt-get install -y podman
```

#### Arch Linux
```bash
sudo pacman -S podman
```

#### Enable rootless containers (Linux)
```bash
# Ensure subuid/subgid are configured
echo "$USER:100000:65536" | sudo tee -a /etc/subuid
echo "$USER:100000:65536" | sudo tee -a /etc/subgid

# Configure the user namespace
podman system migrate
```

---

### macOS

Podman on macOS runs containers inside a lightweight VM (similar to Docker Desktop).

```bash
# Install via Homebrew
brew install podman

# Initialize and start the Podman machine (VM)
podman machine init
podman machine start

# Verify installation
podman info
podman run --rm hello-world
```

**Set memory for the VM** (useful for Elasticsearch, etc.):
```bash
podman machine stop
podman machine set --memory 8192   # 8 GB
podman machine start
```

---

### Windows

```powershell
# Option 1: winget
winget install RedHat.Podman

# Option 2: Chocolatey
choco install podman-desktop

# Initialize Podman machine (WSL2-based)
podman machine init
podman machine start

# Verify
podman run --rm hello-world
```

> **Note**: Windows requires WSL2. Install it with `wsl --install` if not already available.

---

## Installing podman-compose

`podman-compose` is a Python script that implements the `docker-compose` spec for Podman.

```bash
# Using pip (recommended)
pip3 install podman-compose

# Or install from source (latest)
pip3 install https://github.com/containers/podman-compose/archive/main.tar.gz

# Verify
podman-compose --version
```

> **Alternative**: Podman v4.x also supports the `--compatibility` flag and can use Docker Compose v2 files directly via the `podman compose` subcommand (requires `docker-compose` binary in PATH).

---

## Quick Start (Any Category)

Every category folder follows the same workflow:

```bash
# 1. Navigate to a category
cd load-balancers/      # or any other folder

# 2. Start all services in the background
podman-compose up -d

# 3. Check running containers and their health
podman-compose ps

# 4. Follow logs for a specific service
podman-compose logs -f haproxy

# 5. Execute a command inside a container
podman-compose exec redis redis-cli ping

# 6. Stop all services (data in volumes is preserved)
podman-compose down

# 7. Stop AND remove all data volumes
podman-compose down -v
```

---

## Category Details

### Load Balancers

**Location**: `load-balancers/`

| Tool    | Image                                    | Port(s)          | Purpose                    |
|---------|------------------------------------------|------------------|----------------------------|
| HAProxy | `docker.io/library/haproxy:2.9`          | 8080, 8404       | HTTP LB + stats UI         |
| Envoy   | `docker.io/envoyproxy/envoy:v1.29-latest`| 10000, 9901      | Service proxy + admin UI   |
| LVS     | (kernel-level — see README)              | N/A              | Layer-4 kernel LB          |

```bash
cd load-balancers && podman-compose up -d

# Test HAProxy load balancing
curl http://localhost:8080/

# HAProxy stats
open http://localhost:8404/stats   # admin/admin

# Envoy admin
open http://localhost:9901
```

→ See [load-balancers/README.md](load-balancers/README.md) for full details.

---

### Monitoring & Logging

**Location**: `monitoring-logging/`

| Tool          | Image                                        | Port   | Purpose              |
|---------------|----------------------------------------------|--------|----------------------|
| Grafana       | `docker.io/grafana/grafana:10.2.0`           | 3000   | Metrics dashboards   |
| Elasticsearch | `docker.io/elastic/elasticsearch:8.11.0`     | 9200   | Log/data storage     |
| Logstash      | `docker.io/elastic/logstash:8.11.0`          | 5044   | Log pipeline         |
| Kibana        | `docker.io/elastic/kibana:8.11.0`            | 5601   | Log exploration      |
| Graylog       | `docker.io/graylog/graylog:5.2`              | 9000   | Log management       |
| MongoDB       | `docker.io/library/mongo:6.0`                | —      | Graylog store        |
| Jaeger        | `docker.io/jaegertracing/all-in-one:1.52`    | 16686  | Distributed tracing  |

```bash
cd monitoring-logging && podman-compose up -d

# Grafana UI
open http://localhost:3000       # admin/admin

# Kibana
open http://localhost:5601

# Graylog
open http://localhost:9000       # admin/admin

# Jaeger UI
open http://localhost:16686

# Send a test log to Logstash
echo '{"message":"hello","level":"info"}' | nc localhost 5000
```

> ⚠️ Requires at least **4–6 GB RAM**. Elasticsearch is memory-intensive.

→ See [monitoring-logging/README.md](monitoring-logging/README.md) for full details.

---

### CI/CD

**Location**: `ci-cd/`

| Tool           | Image                                    | Purpose                          |
|----------------|------------------------------------------|----------------------------------|
| GitLab Runner  | `docker.io/gitlab/gitlab-runner:latest`  | Executes GitLab CI/CD jobs       |

```bash
cd ci-cd && podman-compose up -d

# Register with your GitLab instance
podman-compose exec gitlab-runner \
  gitlab-runner register \
    --url https://gitlab.com \
    --registration-token <TOKEN> \
    --executor docker \
    --docker-image docker:latest
```

Example pipeline files:
- **GitHub Actions**: `ci-cd/.github/workflows/example.yml`
- **GitLab CI/CD**: `ci-cd/gitlab-ci.yml`

→ See [ci-cd/README.md](ci-cd/README.md) for full details.

---

### Databases

**Location**: `databases/`

| Database    | Image                                             | Port  |
|-------------|---------------------------------------------------|-------|
| MySQL       | `docker.io/library/mysql:8.2`                     | 3306  |
| PostgreSQL  | `docker.io/library/postgres:16`                   | 5432  |
| PXC Node 1  | `docker.io/percona/percona-xtradb-cluster:8.0`    | 3307  |
| PXC Node 2  | same                                              | 3308  |
| PXC Node 3  | same                                              | 3309  |
| Elasticsearch| `docker.io/elastic/elasticsearch:8.11.0`         | 9200  |
| Vitess (vtgate)| `docker.io/vitess/lite:latest`                 | 15306 |

```bash
cd databases && podman-compose up -d

# Connect to MySQL
mysql -h 127.0.0.1 -u appuser -papppassword appdb

# Connect to PostgreSQL
psql -h 127.0.0.1 -U pguser appdb

# Connect to Vitess
mysql -h 127.0.0.1 -P 15306 -u root

# Start only specific services
podman-compose up -d mysql postgres
```

→ See [databases/README.md](databases/README.md) for full details.

---

### Caching

**Location**: `caching/`

| Service          | Image                              | Port  |
|------------------|------------------------------------|-------|
| Redis            | `docker.io/library/redis:7.2`      | 6379  |
| Redis Sentinel   | `docker.io/library/redis:7.2`      | 26379 |
| Memcached        | `docker.io/library/memcached:1.6`  | 11211 |

```bash
cd caching && podman-compose up -d

# Redis CLI
redis-cli -h localhost -a redispassword

# Test Memcached
echo "stats" | nc localhost 11211
```

→ See [caching/README.md](caching/README.md) for full details.

---

### Message Brokers

**Location**: `message-brokers/`

| Service          | Image                                            | Port  |
|------------------|--------------------------------------------------|-------|
| RabbitMQ         | `docker.io/library/rabbitmq:3.12-management`     | 5672, 15672 |
| Zookeeper        | `docker.io/confluentinc/cp-zookeeper:7.5.0`      | 2181  |
| Kafka            | `docker.io/confluentinc/cp-kafka:7.5.0`          | 9092  |
| PubSub Emulator  | `docker.io/google/cloud-sdk:latest`              | 8085  |

```bash
cd message-brokers && podman-compose up -d

# RabbitMQ Management UI
open http://localhost:15672    # admin/adminpassword

# Create a Kafka topic
podman exec kafka kafka-topics \
  --bootstrap-server localhost:9092 --create --topic my-topic

# Connect to PubSub emulator
export PUBSUB_EMULATOR_HOST=localhost:8085
```

→ See [message-brokers/README.md](message-brokers/README.md) for full details.

---

## Best Practices

### Security

1. **Never use default credentials in production** — change all passwords before deploying
2. **Run rootless** — Podman's rootless mode prevents container escapes from affecting the host
3. **Use read-only mounts** where possible: `volumes: - ./config.yml:/etc/app/config.yml:ro`
4. **Scan images for vulnerabilities**:
   ```bash
   podman run --rm -v /var/run/podman/podman.sock:/var/run/docker.sock \
     aquasec/trivy image docker.io/library/redis:7.2
   ```
5. **Keep images updated** — pin to specific versions, not `latest`, in production

### Performance

6. **Set resource limits** to prevent one container from starving others:
   ```yaml
   deploy:
     resources:
       limits:
         memory: 512M
         cpus: "1.0"
   ```
7. **Use named volumes** (not bind mounts) for database data — better I/O performance
8. **Set JVM heap** explicitly for Java services (Elasticsearch, Logstash):
   ```yaml
   environment:
     - ES_JAVA_OPTS=-Xms1g -Xmx1g
   ```

### Operations

9. **Always add health checks** — `depends_on` with `condition: service_healthy` ensures correct startup order
10. **Use `restart: unless-stopped`** for services that must survive container restarts
11. **Externalize configuration** — use environment variables or config file mounts; never bake secrets into images
12. **Clean up unused resources** periodically:
    ```bash
    podman system prune -a
    podman volume prune
    ```

### Development Workflow

13. **One category at a time** — the full stack requires significant RAM; start only what you need
14. **Use `podman-compose logs -f`** to debug startup issues
15. **Generate Kubernetes manifests** from your compose file:
    ```bash
    podman generate kube > k8s-manifests.yaml
    ```

---

## Troubleshooting

### Containers exit immediately
```bash
# Check logs
podman-compose logs <service-name>

# Check exit code
podman ps -a
```

### Elasticsearch won't start
```bash
# On Linux — increase vm.max_map_count
sudo sysctl -w vm.max_map_count=262144

# Make it permanent
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

### Port already in use
```bash
# Find what's using the port
sudo ss -tlnp | grep :6379

# Or
sudo lsof -i :6379
```

### Rootless Podman: "permission denied" on volume
```bash
# Fix ownership for volumes
podman unshare chown -R 1000:1000 /path/to/data
```

### podman-compose not found
```bash
pip3 install --user podman-compose
export PATH="$HOME/.local/bin:$PATH"
```

### Reset everything for a category
```bash
cd <category>
podman-compose down -v    # stops containers AND removes volumes
podman-compose up -d      # fresh start
```
