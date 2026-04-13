# Load Balancers

This directory contains Podman configurations for three industry-standard load balancing solutions: **HAProxy**, **Envoy**, and a discussion of **LVS**.

## Table of Contents

- [Overview](#overview)
- [Services](#services)
  - [HAProxy](#haproxy)
  - [Envoy Proxy](#envoy-proxy)
  - [LVS (Linux Virtual Server)](#lvs-linux-virtual-server)
- [Quick Start](#quick-start)
- [Configuration Details](#configuration-details)
- [Ports](#ports)
- [Use Cases](#use-cases)

---

## Overview

| Tool    | Type              | Layer | Best For                              |
|---------|-------------------|-------|---------------------------------------|
| HAProxy | Software LB/Proxy | L4/L7 | HTTP/TCP load balancing, ACLs, SSL termination |
| Envoy   | Service Proxy     | L7    | Microservices, service mesh, observability |
| LVS     | Kernel-level LB   | L4    | Ultra-high throughput, kernel bypass   |

---

## Services

### HAProxy

[HAProxy](https://www.haproxy.org/) is a reliable, high-performance TCP/HTTP load balancer and proxy. It is widely used in production for its flexibility, rich ACL system, and excellent observability.

**Configuration file**: `haproxy.cfg`

Key features configured:
- Round-robin load balancing across `backend1` and `backend2`
- Health checks every 2 seconds (`check inter 2s`)
- Stats dashboard at `http://localhost:8404/stats` (admin/admin)
- Request forwarding headers (`X-Forwarded-For`)
- Connection timeouts

**Access the stats page:**
```bash
open http://localhost:8404/stats
# Login: admin / admin
```

**Test load balancing:**
```bash
for i in $(seq 1 10); do curl -s http://localhost:8080/ | grep -o "backend[12]"; done
```

---

### Envoy Proxy

[Envoy](https://www.envoyproxy.io/) is a high-performance, cloud-native proxy originally built at Lyft and now part of the CNCF. It powers Istio service mesh and many large-scale systems.

**Configuration file**: `envoy.yaml`

Key features configured:
- HTTP listener on port 10000 with round-robin upstream cluster
- Path-based routing (`/service1` → `backend1`, `/` → both backends)
- Built-in health check filter (`/healthz`)
- Admin UI at `http://localhost:9901`
- Access logging to stdout

**Access admin UI:**
```bash
open http://localhost:9901
```

**View cluster stats:**
```bash
curl http://localhost:9901/stats | grep service_backend
```

---

### LVS (Linux Virtual Server)

LVS operates at **kernel level (L4)** using the netfilter framework. It cannot run meaningfully inside a container because it requires:
- Direct access to kernel modules (`ip_vs`, `ip_vs_rr`, etc.)
- Host-level network configuration (Virtual IP assignment)
- Typically used with `keepalived` for health checking and VIP failover

**To use LVS on a Linux host:**
```bash
# Install ipvsadm
sudo apt-get install -y ipvsadm keepalived

# Load kernel modules
sudo modprobe ip_vs ip_vs_rr ip_vs_wrr ip_vs_lc

# Add a virtual service (example)
sudo ipvsadm -A -t 192.168.1.100:80 -s rr
sudo ipvsadm -a -t 192.168.1.100:80 -r 192.168.1.10:80 -m
sudo ipvsadm -a -t 192.168.1.100:80 -r 192.168.1.11:80 -m

# View current rules
sudo ipvsadm -L -n
```

For container-friendly high-availability, use HAProxy or Envoy with `keepalived` in a separate container.

---

## Quick Start

```bash
# Start all services
podman-compose up -d

# Check status
podman-compose ps

# View HAProxy logs
podman-compose logs -f haproxy

# View Envoy logs
podman-compose logs -f envoy

# Stop all services
podman-compose down
```

---

## Configuration Details

### HAProxy (`haproxy.cfg`)

| Section    | Purpose                              |
|------------|--------------------------------------|
| `global`   | Process-level settings (threads, log)|
| `defaults` | Default timeouts and modes           |
| `frontend` | Listener definitions with ACLs       |
| `backend`  | Server pool with health checks       |

Modify the backend servers by editing the `server` lines in `haproxy.cfg`:
```
server myserver 10.0.0.5:80 check inter 2s
```

### Envoy (`envoy.yaml`)

| Section              | Purpose                                  |
|----------------------|------------------------------------------|
| `admin`              | Admin API and UI                         |
| `static_resources`   | Listeners (inbound) and clusters (upstream) |
| `listeners`          | Port bindings and filter chains           |
| `clusters`           | Upstream service definitions              |

---

## Ports

| Service    | Port  | Protocol | Description           |
|------------|-------|----------|-----------------------|
| HAProxy    | 8080  | HTTP     | Load-balanced traffic |
| HAProxy    | 8404  | HTTP     | Stats dashboard       |
| Envoy      | 10000 | HTTP     | Proxy listener        |
| Envoy      | 9901  | HTTP     | Admin UI              |
| backend1   | (internal) | HTTP | Nginx backend 1  |
| backend2   | (internal) | HTTP | Nginx backend 2  |

---

## Use Cases

### When to use HAProxy
- Traditional web/API load balancing
- SSL/TLS termination
- Advanced ACL-based routing
- High-performance TCP proxying

### When to use Envoy
- Microservices and service mesh (Istio sidecar)
- gRPC load balancing
- Advanced observability (metrics, tracing)
- Dynamic configuration via xDS APIs

### When to use LVS
- Extreme throughput requirements (millions of connections)
- Kernel-level packet forwarding without userspace overhead
- Combined with keepalived for VIP failover
