# Monitoring & Logging

This directory contains the full **observability stack** — metrics, logs, and traces — using Grafana, the ELK Stack, Graylog, and Jaeger.

## Table of Contents

- [Overview](#overview)
- [Services](#services)
  - [Grafana](#grafana)
  - [ELK Stack](#elk-stack-elasticsearch--logstash--kibana)
  - [Graylog](#graylog)
  - [Jaeger](#jaeger)
- [Quick Start](#quick-start)
- [Sending Logs](#sending-logs)
- [Ports Reference](#ports-reference)
- [Resource Requirements](#resource-requirements)

---

## Overview

| Tool              | Purpose                                           |
|-------------------|---------------------------------------------------|
| **Elasticsearch** | Distributed search and analytics engine (shared by ELK + Graylog) |
| **Logstash**      | Log ingestion, parsing, and enrichment pipeline   |
| **Kibana**        | ELK log exploration and visualization UI          |
| **Grafana**       | Metrics dashboards and alerting                   |
| **MongoDB**       | Graylog metadata and configuration store          |
| **Graylog**       | Centralized log management, SIEM, alerting        |
| **Jaeger**        | Distributed tracing (OpenTelemetry / Zipkin compatible) |

---

## Services

### Grafana

[Grafana](https://grafana.com/) is the standard for metrics visualization. Connect it to Prometheus, Elasticsearch, Loki, CloudWatch, and many more datasources.

- **URL**: http://localhost:3000
- **Login**: `admin` / `admin`

**Add Elasticsearch as a datasource:**
1. Go to **Configuration → Data Sources → Add data source**
2. Choose **Elasticsearch**
3. URL: `http://elasticsearch:9200`
4. Index: `logstash-*`

---

### ELK Stack (Elasticsearch + Logstash + Kibana)

The classic log management trio.

#### Elasticsearch
Central data store — indexes and queries logs. Available at http://localhost:9200.

```bash
# Check cluster health
curl http://localhost:9200/_cluster/health?pretty

# List all indices
curl http://localhost:9200/_cat/indices?v
```

#### Logstash
Parses and enriches incoming logs. Pipeline defined in `logstash/pipeline/logstash.conf`.

Supported inputs:
| Input     | Port  | Protocol | Format          |
|-----------|-------|----------|-----------------|
| Beats     | 5044  | TCP      | Beats protocol  |
| Syslog    | 5514  | TCP/UDP  | RFC 3164        |
| JSON TCP  | 5000  | TCP      | JSON lines      |
| JSON UDP  | 5001  | UDP      | JSON lines      |

**Send a test log:**
```bash
echo '{"message": "hello world", "level": "info", "service": "myapp"}' | nc localhost 5000
```

#### Kibana
Log exploration UI. Available at http://localhost:5601.

After startup:
1. Go to **Stack Management → Index Patterns**
2. Create pattern: `logstash-*`
3. Go to **Discover** to query logs

---

### Graylog

[Graylog](https://graylog.org/) is a powerful log management and SIEM platform with alerting, streams, and dashboards.

- **URL**: http://localhost:9000
- **Login**: `admin` / `admin`

**Send GELF log:**
```bash
echo '{"version":"1.1","host":"myhost","short_message":"test message","level":1}' | nc -u localhost 12201
```

**Configure an input (in the UI):**
1. Go to **System → Inputs**
2. Select **GELF UDP** and click **Launch new input**
3. Port: `12201`, Bind: `0.0.0.0`

---

### Jaeger

[Jaeger](https://www.jaegertracing.io/) is a distributed tracing system compatible with OpenTelemetry and Zipkin.

- **Jaeger UI**: http://localhost:16686
- **OTLP gRPC**: `localhost:4317`
- **OTLP HTTP**: `localhost:4318`

**Send a test trace (OpenTelemetry SDK):**
```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace.export import BatchSpanProcessor

provider = TracerProvider()
exporter = OTLPSpanExporter(endpoint="http://localhost:4317", insecure=True)
provider.add_span_processor(BatchSpanProcessor(exporter))
trace.set_tracer_provider(provider)

tracer = trace.get_tracer("my-service")
with tracer.start_as_current_span("my-operation"):
    print("Tracing!")
```

---

## Quick Start

```bash
# Start the full stack (may take 2-3 minutes for all services to become healthy)
podman-compose up -d

# Watch startup progress
podman-compose ps

# Follow logs
podman-compose logs -f

# Stop and remove volumes
podman-compose down -v
```

> **Note**: Elasticsearch requires at least **4 GB of RAM** on the host. If it fails to start, increase Docker/Podman memory limits.

---

## Sending Logs

### From a containerized application

Add your app to the same `monitoring-net` network and send to Logstash:

```yaml
# In your app's podman-compose.yml
services:
  myapp:
    networks:
      - monitoring-logging_monitoring-net
    logging:
      driver: gelf
      options:
        gelf-address: "udp://localhost:12201"
```

### Filebeat agent

```yaml
filebeat.inputs:
  - type: log
    paths:
      - /var/log/*.log

output.logstash:
  hosts: ["localhost:5044"]
```

---

## Ports Reference

| Service       | Port       | Description                      |
|---------------|------------|----------------------------------|
| Elasticsearch | 9200       | HTTP REST API                    |
| Elasticsearch | 9300       | Transport (cluster comm)         |
| Logstash      | 5044       | Beats input                      |
| Logstash      | 5000       | TCP JSON input                   |
| Logstash      | 5001/udp   | UDP JSON input                   |
| Logstash      | 5514       | Syslog input                     |
| Logstash      | 9600       | Logstash API                     |
| Kibana        | 5601       | Web UI                           |
| Grafana       | 3000       | Web UI                           |
| Graylog       | 9000       | Web UI & REST API                |
| Graylog       | 12201      | GELF UDP/TCP                     |
| Graylog       | 1514       | Syslog TCP/UDP                   |
| Jaeger        | 16686      | Jaeger UI                        |
| Jaeger        | 4317       | OTLP gRPC receiver               |
| Jaeger        | 4318       | OTLP HTTP receiver               |
| Jaeger        | 14268      | Jaeger collector HTTP            |
| Jaeger        | 6831/udp   | Jaeger agent compact thrift      |

---

## Resource Requirements

| Service       | Min RAM | Recommended RAM |
|---------------|---------|-----------------|
| Elasticsearch | 1 GB    | 2 GB            |
| Logstash      | 512 MB  | 1 GB            |
| Kibana        | 512 MB  | 1 GB            |
| Grafana       | 128 MB  | 256 MB          |
| MongoDB       | 256 MB  | 512 MB          |
| Graylog       | 512 MB  | 1 GB            |
| Jaeger        | 128 MB  | 512 MB          |
| **Total**     | **~3 GB** | **~6 GB**     |
