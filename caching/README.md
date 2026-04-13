# Caching

This directory contains Podman configurations for two industry-standard caching solutions: **Redis** and **Memcached**.

## Table of Contents

- [Overview](#overview)
- [Services](#services)
  - [Redis 7.2](#redis-72)
  - [Redis Sentinel](#redis-sentinel)
  - [Memcached 1.6](#memcached-16)
- [Quick Start](#quick-start)
- [Common Patterns](#common-patterns)
- [Choosing Between Redis and Memcached](#choosing-between-redis-and-memcached)
- [Ports Reference](#ports-reference)

---

## Overview

| Tool        | Type               | Persistence | Data Structures | Clustering |
|-------------|--------------------|-------------|-----------------|------------|
| Redis       | Cache + Data Store | ✅ AOF + RDB | Rich (strings, lists, sets, hashes, streams, etc.) | Redis Cluster / Sentinel |
| Memcached   | Pure Cache         | ❌ None      | Strings only    | Client-side sharding |

---

## Services

### Redis 7.2

[Redis](https://redis.io/) is an in-memory data structure store used as a cache, message broker, and database.

**Configuration highlights:**
- Password authentication: `redispassword`
- Max memory: `512 MB` with `allkeys-lru` eviction
- **Persistence**: AOF (`appendfsync everysec`) + RDB snapshots
- Logs to stdout

**Connect:**
```bash
# CLI
redis-cli -h localhost -p 6379 -a redispassword

# Using podman exec
podman exec -it redis redis-cli -a redispassword
```

**Common commands:**
```bash
# Set/Get
SET user:1 '{"name":"Alice","email":"alice@example.com"}'
GET user:1

# Set with TTL (expire in 300 seconds)
SET session:abc123 token_value EX 300

# Increment counter
INCR page_views:homepage

# Lists (queue)
LPUSH jobs '{"type":"email","to":"user@example.com"}'
RPOP jobs

# Pub/Sub
SUBSCRIBE notifications
PUBLISH notifications '{"event":"new_order","id":42}'

# Hash (store object fields)
HSET product:99 name "Widget" price "9.99" stock "100"
HGETALL product:99

# Check memory usage
INFO memory
```

**Persistence files** are stored in the `redis-data` volume at `/data`.

---

### Redis Sentinel

Redis Sentinel provides **high availability** for Redis: it monitors the primary, detects failures, and promotes a replica automatically.

**Configuration:**
- Monitors the `redis` primary instance
- 1 sentinel quorum (suitable for single-sentinel local dev)
- Failover timeout: 10 seconds

**Connect to Sentinel:**
```bash
redis-cli -p 26379 sentinel masters
redis-cli -p 26379 sentinel slaves mymaster
```

**Application connection with Sentinel (Python example):**
```python
from redis.sentinel import Sentinel

sentinel = Sentinel([("localhost", 26379)], socket_timeout=0.1)
master = sentinel.master_for("mymaster", password="redispassword")
slave = sentinel.slave_for("mymaster", password="redispassword")

master.set("key", "value")
print(slave.get("key"))
```

---

### Memcached 1.6

[Memcached](https://memcached.org/) is a simple, high-performance distributed memory cache. It is stateless and scales horizontally via client-side consistent hashing.

**Configuration:**
- Max memory: `256 MB`
- Max connections: `1024`
- Threads: `4`
- Max item size: `5 MB`

**Connect:**
```bash
# Using telnet
telnet localhost 11211

# Using nc
echo "stats" | nc localhost 11211

# Using Python
pip install pymemcache
```

**Common operations (via telnet):**
```
set mykey 0 300 5
hello
STORED

get mykey
VALUE mykey 0 5
hello
END

stats
delete mykey
```

**Python example:**
```python
from pymemcache.client.base import Client

client = Client(("localhost", 11211))
client.set("user:1", b'{"name":"Alice"}', expire=300)
value = client.get("user:1")
print(value)
```

---

## Quick Start

```bash
# Start Redis + Memcached
podman-compose up -d

# Check status
podman-compose ps

# View Redis logs
podman-compose logs -f redis

# Stop
podman-compose down
```

---

## Common Patterns

### Cache-Aside (Lazy Loading)
```python
def get_user(user_id):
    cached = redis.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)

    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    redis.setex(f"user:{user_id}", 3600, json.dumps(user))
    return user
```

### Write-Through
```python
def update_user(user_id, data):
    db.execute("UPDATE users SET ... WHERE id = ?", user_id)
    redis.setex(f"user:{user_id}", 3600, json.dumps(data))
```

### Rate Limiting
```python
def is_rate_limited(user_id, limit=100, window=60):
    key = f"ratelimit:{user_id}"
    count = redis.incr(key)
    if count == 1:
        redis.expire(key, window)
    return count > limit
```

### Distributed Lock
```python
import uuid
lock_key = "lock:resource"
lock_id = str(uuid.uuid4())

# Acquire
acquired = redis.set(lock_key, lock_id, nx=True, ex=30)

# Release (only if we own the lock)
if redis.get(lock_key) == lock_id:
    redis.delete(lock_key)
```

---

## Choosing Between Redis and Memcached

| Feature              | Redis            | Memcached       |
|----------------------|------------------|-----------------|
| Data structures      | Rich (20+ types) | Strings only    |
| Persistence          | AOF + RDB        | None            |
| Pub/Sub              | ✅               | ❌              |
| Scripting (Lua)      | ✅               | ❌              |
| Transactions         | ✅ (MULTI/EXEC)  | ❌              |
| High availability    | Sentinel/Cluster | Client-side     |
| Memory efficiency    | Moderate         | High            |
| Multi-threading      | Limited (6.0+)   | Full            |
| **Best for**         | Sessions, queues, leaderboards, pub/sub | Simple key-value cache, read-heavy workloads |

---

## Ports Reference

| Service          | Port  | Protocol | Description                   |
|------------------|-------|----------|-------------------------------|
| Redis            | 6379  | TCP      | Redis protocol                |
| Redis Sentinel   | 26379 | TCP      | Sentinel protocol             |
| Memcached        | 11211 | TCP/UDP  | Memcached protocol            |
