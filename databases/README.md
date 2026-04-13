# Databases

This directory contains Podman configurations for five production-grade database systems: **MySQL**, **PostgreSQL**, **Percona XtraDB Cluster**, **Elasticsearch**, and **Vitess**.

## Table of Contents

- [Overview](#overview)
- [Services](#services)
  - [MySQL 8.2](#mysql-82)
  - [PostgreSQL 16](#postgresql-16)
  - [Percona XtraDB Cluster](#percona-xtradb-cluster)
  - [Elasticsearch](#elasticsearch)
  - [Vitess](#vitess)
- [Quick Start](#quick-start)
- [Connection Details](#connection-details)
- [Ports Reference](#ports-reference)

---

## Overview

| Database              | Type              | Best For                                    |
|-----------------------|-------------------|---------------------------------------------|
| MySQL 8.2             | Relational        | General-purpose OLTP, web applications      |
| PostgreSQL 16         | Relational        | Complex queries, JSONB, extensions, GIS     |
| Percona XtraDB Cluster| Relational (HA)   | MySQL with synchronous multi-master replication |
| Elasticsearch 8.11    | Document/Search   | Full-text search, analytics, log storage    |
| Vitess                | MySQL-compatible  | Horizontal sharding of MySQL at scale       |

---

## Services

### MySQL 8.2

[MySQL](https://www.mysql.com/) is the world's most popular open-source relational database.

**Configuration highlights:**
- `utf8mb4` charset and collation
- `innodb_buffer_pool_size=256M`
- `max_connections=200`

**Connect:**
```bash
# Using mysql CLI
mysql -h 127.0.0.1 -P 3306 -u appuser -papppassword appdb

# Using podman exec
podman exec -it mysql mysql -u root -prootpassword
```

**Create a table:**
```sql
USE appdb;
CREATE TABLE users (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

### PostgreSQL 16

[PostgreSQL](https://www.postgresql.org/) is a powerful, standards-compliant relational database with advanced features.

**Configuration highlights:**
- `shared_buffers=256MB`
- `max_connections=200`
- WAL level set to `replica` (replication-ready)
- Slow query logging > 500ms

**Connect:**
```bash
# Using psql
psql -h 127.0.0.1 -p 5432 -U pguser -d appdb

# Using podman exec
podman exec -it postgres psql -U pguser appdb
```

**Create a table:**
```sql
CREATE TABLE products (
  id BIGSERIAL PRIMARY KEY,
  name TEXT NOT NULL,
  price NUMERIC(10, 2),
  metadata JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Create an index on JSONB
CREATE INDEX idx_products_metadata ON products USING GIN (metadata);
```

---

### Percona XtraDB Cluster

[Percona XtraDB Cluster (PXC)](https://www.percona.com/software/mysql-database/percona-xtradb-cluster) provides synchronous multi-master replication for MySQL using the Galera library.

**Cluster topology (3 nodes):**
- `pxc-node1` — bootstraps the cluster (port 3307)
- `pxc-node2` — joins pxc-node1 (port 3308)
- `pxc-node3` — joins pxc-node1 (port 3309)

> ⚠️ **Startup order matters**: pxc-node1 must be healthy before nodes 2 and 3 start.

**Check cluster status:**
```bash
podman exec pxc-node1 mysql -u root -prootpassword \
  -e "SHOW STATUS LIKE 'wsrep_%';" | grep -E 'wsrep_cluster_size|wsrep_local_state_comment'
```

**Connect to any node:**
```bash
mysql -h 127.0.0.1 -P 3307 -u pxcuser -ppxcpassword pxcdb
```

---

### Elasticsearch

[Elasticsearch](https://www.elastic.co/elasticsearch/) is a distributed, RESTful search and analytics engine.

**Configuration:**
- Single-node mode (`discovery.type=single-node`)
- Security disabled for local development
- 1 GB JVM heap

**Basic operations:**
```bash
# Index a document
curl -X POST "localhost:9200/products/_doc/1" \
  -H 'Content-Type: application/json' \
  -d '{"name": "Widget", "price": 9.99, "tags": ["sale", "new"]}'

# Search
curl "localhost:9200/products/_search?q=Widget&pretty"

# Check cluster health
curl "localhost:9200/_cluster/health?pretty"
```

---

### Vitess

[Vitess](https://vitess.io/) is a database clustering system for MySQL designed to handle internet-scale deployments with horizontal sharding.

**Architecture:**
```
Application → vtgate (MySQL port 15306)
                ↓
           vttablet (tablet manager)
                ↓
            MySQL (internal)
                ↑
            etcd (topology store)
            vtctld (control plane)
```

**Connect to Vitess via vtgate (MySQL protocol):**
```bash
mysql -h 127.0.0.1 -P 15306 -u root
```

**Access vtctld UI:**
```
http://localhost:15000
```

**Create a keyspace and table:**
```bash
# Via vtgate
mysql -h 127.0.0.1 -P 15306 -e "CREATE DATABASE commerce;"
mysql -h 127.0.0.1 -P 15306 commerce -e "
  CREATE TABLE orders (
    order_id BIGINT NOT NULL AUTO_INCREMENT,
    customer_id BIGINT NOT NULL,
    amount DECIMAL(10, 2),
    PRIMARY KEY (order_id)
  ) ENGINE=InnoDB;
"
```

> **Note**: Full Vitess sharding configuration requires `vtctl` commands to define `VSchema`. See the [Vitess docs](https://vitess.io/docs/get-started/) for production setup.

---

## Quick Start

```bash
# Start all databases (this will take 2-3 minutes)
podman-compose up -d

# Start only MySQL and PostgreSQL
podman-compose up -d mysql postgres

# Check all service health
podman-compose ps

# Stop and remove containers (data persisted in volumes)
podman-compose down

# Stop and remove ALL data
podman-compose down -v
```

---

## Connection Details

| Database     | Host      | Port  | User       | Password       | Database  |
|--------------|-----------|-------|------------|----------------|-----------|
| MySQL        | localhost | 3306  | appuser    | apppassword    | appdb     |
| PostgreSQL   | localhost | 5432  | pguser     | pgpassword     | appdb     |
| PXC Node 1   | localhost | 3307  | pxcuser    | pxcpassword    | pxcdb     |
| PXC Node 2   | localhost | 3308  | pxcuser    | pxcpassword    | pxcdb     |
| PXC Node 3   | localhost | 3309  | pxcuser    | pxcpassword    | pxcdb     |
| Elasticsearch| localhost | 9200  | (none)     | (none)         | (REST API)|
| Vitess/vtgate| localhost | 15306 | root       | (none)         | (MySQL)   |

---

## Ports Reference

| Service        | Port  | Protocol | Description                     |
|----------------|-------|----------|---------------------------------|
| MySQL          | 3306  | TCP      | MySQL protocol                  |
| PostgreSQL     | 5432  | TCP      | PostgreSQL protocol             |
| PXC Node 1     | 3307  | TCP      | MySQL protocol                  |
| PXC Node 2     | 3308  | TCP      | MySQL protocol                  |
| PXC Node 3     | 3309  | TCP      | MySQL protocol                  |
| Elasticsearch  | 9200  | HTTP     | REST API                        |
| Elasticsearch  | 9300  | TCP      | Transport protocol              |
| etcd           | 2379  | HTTP     | etcd client API                 |
| vtctld         | 15000 | HTTP     | Vitess control plane UI         |
| vtctld         | 15999 | gRPC     | Vitess control plane gRPC       |
| vttablet       | 15100 | HTTP     | Tablet API                      |
| vtgate         | 15001 | HTTP     | vtgate HTTP                     |
| vtgate         | 15306 | TCP      | MySQL protocol entry point      |
