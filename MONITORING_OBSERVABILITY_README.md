# Monitoring & Observability System — Disaster Response Platform (J4)

**Version:** 1.0.0  
**Last Updated:** May 7, 2026  
**Scope:** Complete observability framework for the Disaster Response System (J1–J4 components)

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Architecture Overview](#architecture-overview)
3. [Technology Stack](#technology-stack)
4. [Component Descriptions](#component-descriptions)
5. [Data Flow & Integration](#data-flow--integration)
6. [Metrics & Events Coverage](#metrics--events-coverage)
7. [Observability Pillars](#observability-pillars)
8. [Reference Architecture Diagram](#reference-architecture-diagram)

---

## Executive Summary

The **Monitoring & Observability System** for the Disaster Response Platform provides **real-time visibility** into all system components, services, and operational health. It integrates:

- **Prometheus** for time-series metrics collection and alerting
- **Grafana** for dashboard visualization and incident tracking
- **ELK Stack** (Elasticsearch, Logstash, Kibana) for centralized log aggregation
- **Jaeger** for distributed tracing across microservices
- **Alertmanager** for intelligent alert routing and escalation

This system enables **proactive incident detection**, **rapid root-cause analysis**, and **compliance auditing** across J1 (Device Edge), J2 (Data Intelligence), J3 (System Interaction), and J4 (Platform Security) subsystems.

---

## Architecture Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                   DATA SOURCES & INSTRUMENTATION                     │
├─────────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ ┌──────────┐  │
│  │   J1: IoT    │  │   J2: ML     │  │   J3: DMS    │ │ J4: Core │  │
│  │   Reports    │  │  Predictions │  │   Dashboard  │ │ Services │  │
│  │   & Events   │  │  & Analysis  │  │   & Webhooks │ │ (Kong,   │  │
│  │              │  │              │  │              │ │ Keycloak,│  │
│  │ Exporters:   │  │ Exporters:   │  │ Exporters:   │ │ Postgres)│  │
│  │ Prometheus   │  │ Prometheus   │  │ Prometheus   │ │          │  │
│  │ StdErr/File  │  │ API Metrics  │  │ /api/metrics │ │ Exporters│  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘ └────┬─────┘  │
└─────────┼──────────────────┼──────────────────┼──────────────┼────────┘
          │                  │                  │              │
          └──────────────────┼──────────────────┼──────────────┘
                             │                  │
                   ┌─────────▼──────────────────▼────────┐
                   │   COLLECTION & PROCESSING LAYER     │
                   ├─────────────────────────────────────┤
                   │  Prometheus (9090) [Pull Model]     │
                   │  - Scrape interval: 15s              │
                   │  - Retention: 24h (configurable)    │
                   │  - Alert rules: 15s evaluation      │
                   │  - Multi-job scraping                │
                   │  - Service discovery: static+sd_json │
                   └──────────┬──────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
┌───────▼─────────┐  ┌───────▼──────────┐  ┌──────▼──────────┐
│ ALERTMANAGER    │  │  LOGS PIPELINE   │  │  DISTRIBUTED    │
│ (9093)          │  │  [Push Model]    │  │  TRACING        │
├─────────────────┤  ├──────────────────┤  ├─────────────────┤
│ Alert routing   │  │ Filebeat (edge)  │  │ Jaeger Agent    │
│ Deduplication   │  │ → Logstash       │  │ (port 6831 UDP) │
│ Suppression     │  │ → Elasticsearch  │  │                 │
│ Webhook push    │  │ (index rollover) │  │ Jaeger UI       │
│ Email delivery  │  │                  │  │ (port 16686)    │
└───────┬─────────┘  └────────┬─────────┘  └────────┬────────┘
        │                     │                     │
        └─────────────┬───────┴─────────────────────┘
                      │
        ┌─────────────▼──────────────┐
        │  VISUALIZATION & ANALYSIS  │
        ├────────────────────────────┤
        │ Grafana (3030)             │
        │ - Dashboards               │
        │ - Alerting                 │
        │ - Incident tracking        │
        │                            │
        │ Kibana (5601)              │
        │ - Log search & filtering   │
        │ - ILM policy management    │
        │ - Custom visualizations    │
        │                            │
        │ Jaeger UI (16686)          │
        │ - Trace visualization      │
        │ - Service dependencies     │
        │ - Latency analysis         │
        └────────────────────────────┘
```

---

## Technology Stack

### Core Components

| Component | Version | Port | Purpose | Language |
|-----------|---------|------|---------|----------|
| **Prometheus** | Latest | 9090 | Metrics scraping, storage, alerting | Go |
| **Alertmanager** | Latest | 9093 | Alert routing & deduplication | Go |
| **Grafana** | Latest | 3030 | Dashboards & visualization | Go/React |
| **Elasticsearch** | 8.x | 9200 | Log indexing & search backend | Java |
| **Logstash** | 8.x | 5000 | Log aggregation & processing pipeline | Java/Ruby |
| **Kibana** | 8.x | 5601 | Log visualization UI | Node.js/React |
| **Filebeat** | 8.x | N/A | Log collector (container stdout/stderr) | Go |
| **Jaeger** | Latest | 6831 (UDP), 16686 (UI) | Distributed tracing | Go |
| **Postgres Exporter** | Latest | 9187 | PostgreSQL metrics export | Go |
| **Kafka Exporter** | Latest | 9308 | Kafka broker metrics export | Go |

### Supporting Infrastructure

| Component | Role |
|-----------|------|
| Docker Compose | Service orchestration & networking |
| PostgreSQL 16 | Metrics metadata, alert history storage |
| Kafka | Event streaming for log ingestion & metrics distribution |
| Kong API Gateway | Request routing, metering, authentication logging |
| Keycloak | Identity management, audit logging |

---

## Component Descriptions

### 1. Prometheus — Metrics Collection & Alerting

**Purpose:** Time-series database for metrics collection, storage, and alert evaluation.

**Architecture:**
- **Scrape Model:** Pull-based; Prometheus scrapes endpoints every 15 seconds
- **Retention:** 24 hours (default); older data is compacted via TSDB
- **Scrape Targets:** Multi-job configuration for J1, J2, J3, J4 services
- **Service Discovery:** Static targets + `sd_config.json` for dynamic scaling

**Configuration Details:**
```yaml
scrape_interval: 15s       # How often to pull metrics
evaluation_interval: 15s   # How often to evaluate rules
scrape_timeout: 10s        # Max time to wait for response
retention: 24h             # Data retention period
```

**Metrics Ingested:**
- **System Metrics:** CPU, memory, disk I/O, network
- **Service Metrics:** Request latency, throughput, error rates
- **Application Metrics:** Business logic counters, gauges, histograms
- **Database Metrics:** Connections, queries/sec, cache hit ratios
- **Kafka Metrics:** Consumer lag, topic throughput, broker health

**Alert Rules Evaluation:**
- Rules evaluated every 15 seconds
- Alerts must fire for the configured `for` duration (e.g., 1m) before triggering
- Multiple alert groups enable independent escalation and routing

---

### 2. Alertmanager — Alert Routing & Deduplication

**Purpose:** Intelligent routing, grouping, and delivery of alerts from Prometheus.

**Key Features:**
- **Alert Grouping:** Clusters related alerts by `alertname`, `job`, `severity`
- **Deduplication:** Prevents alert storms (groups wait 10–30s before first notification)
- **Suppression:** Inhibit warnings if critical alert exists for same service
- **Route Hierarchy:** Critical → immediate webhook + email; warning → delayed webhook
- **Receiver Configuration:** Webhook, email, PagerDuty, Slack (extensible)

**Routing Rules:**
```
Default Route (webhook + repeat every 12h)
├─ Critical Alerts → webhook + email (repeat 1h) [immediate]
└─ Warning Alerts → webhook only (repeat 4h) [30s delay]
```

**Current Receivers:**
1. **Default:** Webhook to `http://localhost:5001/alert` (DMS internal handler)
2. **Critical:** Email to `pisandeniwith@gmail.com` + webhook
3. **Warning:** Webhook only

---

### 3. Grafana — Dashboards & Visualization

**Purpose:** Create rich, multi-panel dashboards for monitoring system health and incident tracking.

**Key Features:**
- **Dashboard Provisioning:** Pre-configured dashboards auto-loaded from `provisioning/dashboards/`
- **Datasource Integration:** Connects to Prometheus + Elasticsearch
- **Alert Management:** Associate alerts with dashboard panels, test queries
- **Templating:** Dynamic dropdowns for service/job filtering
- **Annotation Support:** Manual incident markers, deployment events

**Default Dashboards:**
1. **drs-overview.json** — System-wide health, incident count, SLA compliance
2. **Service Dashboards** — Per-service metrics (J1, J2, J3, J4, Kong, Keycloak, Postgres)
3. **Business Dashboards** — Report submissions, predictions generated, evacuation coordinated

**Access:** `http://localhost:3030` (default credentials in docker-compose)

---

### 4. ELK Stack — Centralized Log Aggregation

#### 4.1 Filebeat — Log Collection Agent

**Purpose:** Lightweight log shipper deployed on each container.

**Configuration:**
- **Input:** Docker container stdout/stderr (via Docker socket)
- **Output:** Logstash pipeline (port 5044, Beats protocol)
- **Filtering:** Can drop/sample logs based on field values
- **Enrichment:** Adds container metadata (service_name, pod, namespace)

**Docker Integration:**
```yaml
# Automatically collects all container logs
filebeat:
  inputs:
    - type: container
      paths:
        - /var/lib/docker/containers/${docker.container.id}/*.log
```

#### 4.2 Logstash — Log Processing Pipeline

**Purpose:** Parse, enrich, and route logs to Elasticsearch.

**Pipeline Stages:**

| Stage | Operation | Example |
|-------|-----------|---------|
| **Input** | Receive Beats from Filebeat | `beats { port => 5044 }` |
| **Filter** | Parse JSON logs, extract trace IDs, add service tags | `json { source => "message" }` |
| **Output** | Write to Elasticsearch with rollover index | `index => "drs-logs-%{+YYYY.MM.dd}"` |

**Filter Logic:**
1. Add service name from container metadata
2. If J3 DMS: parse JSON, extract trace/request IDs
3. If Kong: Grok-parse access logs (client IP, method, status, latency)
4. Generic extraction: regex match trace/request IDs across all logs
5. Index into time-based indices (drs-logs-2026.05.07, etc.)

#### 4.3 Elasticsearch — Log Storage & Indexing

**Purpose:** Distributed search & storage for logs.

**Configuration:**
- **Index Strategy:** Time-based (daily rollover: `drs-logs-YYYY.MM.dd`)
- **ILM Policy:** Auto-delete logs older than 30 days (configurable)
- **Shards/Replicas:** 1 shard, 0 replicas (single-node dev; adjust for prod)
- **Mapping:** Auto-inferred from Logstash output (extensible)

**Storage:**
```
Volume: elasticsearch_data (Docker named volume, persistent)
Retention: 30 days by default (via ILM)
Query Language: Kibana Query Language (KQL) + Elasticsearch Query DSL
```

#### 4.4 Kibana — Log Visualization & Search

**Purpose:** Web UI for log exploration, dashboard creation, and alerting.

**Features:**
- **Discover:** Search logs by time range, field filters, KQL queries
- **Dashboards:** Multi-panel log visualizations (counts, trends, top-N)
- **Canvas:** Custom report layouts and drill-down paths
- **Alerts:** Create conditions on log aggregations (e.g., error rate > 5%)
- **Management:** Index pattern setup, field mapping, saved objects

**Default Data View:**
- **Data View Name:** `DRS Logs`
- **Pattern:** `drs-logs-*` (matches all daily indices)
- **Timestamp Field:** `@timestamp`

---

### 5. Jaeger — Distributed Tracing

**Purpose:** Trace requests across microservices to identify latency, dependencies, and bottlenecks.

**Architecture:**
- **Jaeger Agent:** Sidecar on each service; receives spans via UDP (port 6831)
- **Jaeger Collector:** Aggregates spans from agents
- **Jaeger Backend:** Stores spans in local or distributed storage
- **Jaeger UI:** Query & visualize traces (port 16686)

**Instrumentation:**
- Client libraries (OpenTelemetry) emit spans with trace/span IDs
- Propagate trace context in HTTP headers (`traceparent`, `tracestate`)
- Correlate logs & metrics via trace IDs

**Key Metrics Tracked:**
- Request latency (min, max, p50, p95, p99)
- Service-to-service call chains
- Error propagation across services
- Database query latencies

---

### 6. Exporters — Service Metrics Exposure

#### 6.1 Postgres Exporter (Port 9187)

**Purpose:** Export PostgreSQL performance metrics to Prometheus.

**Metrics Exposed:**
- Active connections, idle transactions
- Replication lag, WAL archiving status
- Table/index sizes, autovacuum stats
- Cache hit ratio, query performance
- Locks, deadlocks, sequential scans

#### 6.2 Kafka Exporter (Port 9308)

**Purpose:** Export Kafka broker & topic metrics to Prometheus.

**Metrics Exposed:**
- Broker health, offline partitions
- Consumer group lag, commit offsets
- Topic throughput (bytes/sec, msgs/sec)
- Replication factor, min ISR status

#### 6.3 Kong Metrics (Port 8001/metrics)

**Purpose:** Built-in Kong metrics on admin API.

**Metrics Exposed:**
- Request count, latency, error rates
- Route/service uptime
- Plugin execution times
- Upstream target health

#### 6.4 J3 DMS Metrics (/api/metrics)

**Purpose:** Application-level metrics from Next.js dashboard.

**Metrics Exposed:**
- Incident creation rate
- Dashboard API response time
- WebSocket connection count
- Real-time update latency

---

## Data Flow & Integration

### Metrics Collection Flow

```
┌──────────────────┐
│ Service Exports  │
│ Prometheus Metrics
│ (port 8xxx/xxxxx)
└────────┬─────────┘
         │ (pull every 15s)
         ▼
┌──────────────────────────────┐
│ Prometheus Scraper           │
│ - Reads from 10+ jobs        │
│ - Stores time-series data    │
│ - Evaluates alert rules      │
└────────┬─────────────────────┘
         │
         ├─ (match rule) ────────────┐
         │                           │
         ▼                           ▼
┌──────────────────┐        ┌─────────────────┐
│ Time-Series DB   │        │ Alert Triggers  │
│ (1 data point    │        │ (if expr=true   │
│  per metric/     │        │  for 1m)        │
│  15s)            │        └────────┬────────┘
└──────────────────┘                 │
                                     ▼
                            ┌─────────────────────┐
                            │ Alertmanager        │
                            │ - Groups alerts     │
                            │ - Routes to webhook │
                            │ - Sends email       │
                            └─────────────────────┘
```

### Log Collection Flow

```
┌──────────────────┐
│ Container Logs   │
│ (stdout/stderr)  │
└────────┬─────────┘
         │ (real-time via Docker socket)
         ▼
┌──────────────────────────┐
│ Filebeat                 │
│ - Watches container logs │
│ - Adds metadata          │
│ - Ships via Beats proto  │
└────────┬─────────────────┘
         │ (port 5044)
         ▼
┌──────────────────────────┐
│ Logstash Pipeline        │
│ - Parse JSON/plaintext   │
│ - Extract trace IDs      │
│ - Add service tags       │
│ - Route to ES            │
└────────┬─────────────────┘
         │ (HTTP bulk API)
         ▼
┌──────────────────────────┐
│ Elasticsearch            │
│ - Indexes logs           │
│ - Enforces ILM policy    │
│ - Stores daily indices   │
└────────┬─────────────────┘
         │
         ▼
┌──────────────────────────┐
│ Kibana UI                │
│ - Search & filter logs   │
│ - Create dashboards      │
│ - Generate reports       │
└──────────────────────────┘
```

### Trace Collection Flow

```
┌──────────────────┐
│ Service Spans    │
│ (OpenTelemetry)  │
└────────┬─────────┘
         │ (UDP port 6831)
         ▼
┌──────────────────────────┐
│ Jaeger Agent Sidecar     │
│ - Buffers spans          │
│ - Samples if needed      │
│ - Forwards to collector  │
└────────┬─────────────────┘
         │
         ▼
┌──────────────────────────┐
│ Jaeger Collector         │
│ - Aggregates spans       │
│ - Enforces retention     │
│ - Writes to storage      │
└────────┬─────────────────┘
         │
         ▼
┌──────────────────────────┐
│ Jaeger Backend Storage   │
│ (memory/disk)            │
└────────┬─────────────────┘
         │
         ▼
┌──────────────────────────┐
│ Jaeger UI (port 16686)   │
│ - Query traces by ID     │
│ - Visualize call chains  │
│ - Analyze latency        │
└──────────────────────────┘
```

---

## Metrics & Events Coverage

### System-Level Metrics

| Category | Metric | Alert Threshold | Purpose |
|----------|--------|-----------------|---------|
| **Availability** | `up` per job | == 0 for 1m | Service down detection |
| **CPU** | `node_cpu_seconds_total` | > 80% for 5m | Resource saturation |
| **Memory** | `node_memory_MemAvailable_bytes` | < 500MB for 3m | OOM risk |
| **Disk** | `node_filesystem_avail_bytes` | < 10% for 5m | Storage exhaustion |
| **Network** | `node_network_transmit_bytes_total` | anomaly detection | Bandwidth surge |

### Service-Level Metrics

| Service | Metric | Normal Range | Alert Trigger |
|---------|--------|--------------|---------------|
| **Prometheus** | Storage used | < 80% | > 85% for 5m |
| **Prometheus** | Dropped chunks | 0 | > 100 in 1h |
| **Alertmanager** | Notifications failed | 0 | > 0 in 5m |
| **PostgreSQL** | Active connections | < 50 | > 80 for 5m |
| **PostgreSQL** | Cache hit ratio | > 0.99 | < 0.95 for 10m |
| **Kafka** | Consumer lag | < 100 msgs | > 1000 for 5m |
| **Kong** | Request latency (p95) | < 500ms | > 2s for 1m |
| **Keycloak** | Auth failure rate | < 1% | > 10% for 2m |
| **J3 DMS** | Incident API latency (p99) | < 200ms | > 1s for 1m |
| **J2 Engine** | Prediction generation time | < 30s | > 60s for 1m |

### Application-Level Metrics (Business)

| Event | Metric | Aggregation | Dashboard |
|-------|--------|-------------|-----------|
| SOS Report Submitted | `incident_created_total` | 1-min rate | Overview |
| Report Verified | `incident_verified_total` | 1-min rate | Officer Dashboard |
| Prediction Generated | `predictions_generated_total` | 1-day count | J2 Dashboard |
| Alert Triggered | `risk_alerts_triggered_total` | 1-hour count | Alert History |
| Evacuation Ordered | `evacuation_ordered_total` | 1-day cumsum | Incident Timeline |

---

## Observability Pillars

### 1. Metrics (Time-Series Data)

**Definition:** Numerical measurements recorded at specific time intervals.

**Collection:** Prometheus scrape every 15s
**Retention:** 24 hours
**Query Language:** PromQL
**Visualization:** Grafana panels (gauges, graphs, heatmaps)

**Examples:**
- Request latency: `histogram_quantile(0.95, rate(request_duration_seconds_bucket[5m]))`
- Error rate: `rate(request_errors_total[5m]) / rate(request_total[5m])`
- CPU usage: `100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)`

### 2. Logs (Structured Events)

**Definition:** Detailed records of events emitted by services.

**Collection:** Filebeat → Logstash → Elasticsearch
**Retention:** 30 days (ILM policy)
**Query Language:** Kibana Query Language (KQL)
**Visualization:** Kibana dashboards, log tail, aggregations

**Examples:**
- JSON structured logs: `{ "level": "ERROR", "requestId": "xyz", "message": "DB connection failed" }`
- Kong access logs: `client_ip=192.168.1.1 method=GET status=500 latency_ms=1234`
- J2 prediction logs: `level=INFO service=j2-engine predictions_generated=1089`

### 3. Traces (Distributed Request Paths)

**Definition:** Complete request execution path across services with timing.

**Collection:** OpenTelemetry → Jaeger Agent → Jaeger Backend
**Retention:** 7 days
**Query Language:** Trace ID, service name, operation filters
**Visualization:** Jaeger UI (flame graph, critical path)

**Examples:**
- User submits SOS report → J3 receives → validates → publishes to Kafka → J2 consumes → generates prediction
- Latency breakdown: J3=50ms, Kafka=20ms, J2=200ms, total=270ms

### 4. Health Indicators (SLI/SLO)

**Definition:** Business-aligned indicators of system quality.

**SLI Examples:**
- **Availability:** Percentage of uptime (99.9%)
- **Latency:** P95 request latency < 200ms
- **Error Rate:** <0.1% of requests fail
- **Throughput:** Handle 1000 incidents/min

**SLO Examples:**
- "99.9% of requests complete in < 500ms, measured monthly"
- "System recovers from single component failure in < 5 minutes"

---

## Reference Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        DISASTER RESPONSE SYSTEM                              │
│                       MONITORING & OBSERVABILITY                             │
└──────────────────────────────────────────────────────────────────────────────┘

                    ┌────────────────────────────────┐
                    │   PRODUCTION ENVIRONMENT       │
                    │   (Kubernetes / Docker)        │
                    └────────────────┬───────────────┘
                                     │
         ┌───────────────────────────┼───────────────────────────┐
         │                           │                           │
    ┌────▼────────┐        ┌────────▼────────┐        ┌────────▼────────┐
    │ J1: Device  │        │ J2: Data Intel  │        │ J3: DMS + J4:   │
    │ Edge IoT    │        │ ML Engine       │        │ Core Infra      │
    │             │        │                 │        │ (Kong, KC, DB)  │
    │ Exporters:  │        │ Exporters:      │        │ Exporters:      │
    │ Prometheus  │        │ Prometheus      │        │ Prometheus      │
    │ /metrics    │        │ /api/metrics    │        │ /api/metrics    │
    │ 8081        │        │ 8082            │        │ 3000, 8001,     │
    │             │        │                 │        │ 9187, etc       │
    └────┬────────┘        └────────┬────────┘        └────────┬────────┘
         │                          │                         │
         │ (scrape 15s)             │ (scrape 15s)           │ (scrape 15s)
         │                          │                         │
         └──────────────────────────┼──────────────────────────┘
                                    │
                          ┌─────────▼─────────┐
                          │ PROMETHEUS (9090) │
                          │ - Time-series DB  │
                          │ - 24h retention   │
                          │ - Alert rules     │
                          │ - PromQL queries  │
                          └────────┬──────────┘
                                   │
            ┌──────────────────────┼──────────────────────┐
            │                      │                      │
    ┌───────▼────────┐   ┌────────▼────────┐   ┌────────▼────────┐
    │ ALERTMANAGER   │   │ ELK STACK       │   │ JAEGER          │
    │ (9093)         │   │ (5601, 9200)    │   │ (16686)         │
    │                │   │                 │   │                 │
    │ - Route alerts │   │ - Filebeat      │   │ - Trace spans   │
    │ - Deduplicate  │   │ - Logstash      │   │ - Service map   │
    │ - Email/webhook│   │ - Elasticsearch │   │ - Latency analy │
    │ - 12h repeat   │   │ - Kibana        │   │                 │
    └───────────────┘   └─────────────────┘   └─────────────────┘
            │                    │                      │
            │ (webhook)          │ (UI)                 │ (UI)
            ▼                    ▼                      ▼
    ┌─────────────────────────────────────────────────────────────┐
    │           OBSERVABILITY DASHBOARDS & UIs                    │
    ├─────────────────────────────────────────────────────────────┤
    │ • Grafana (3030): System health, incident tracking, SLOs   │
    │ • Kibana (5601): Log search, trends, custom dashboards     │
    │ • Jaeger UI (16686): Trace analysis, critical path, deps   │
    │ • Alert Console: Active alerts, escalation status          │
    └─────────────────────────────────────────────────────────────┘
            │
            ▼
    ┌─────────────────────────────────────────────────────────────┐
    │            ON-CALL ENGINEERS & DASHBOARDS                  │
    ├─────────────────────────────────────────────────────────────┤
    │ • Incident Response Automation (Webhook → DMS)             │
    │ • Mobile Alerts (Email, Slack, SMS integration)            │
    │ • Compliance & Audit Logs (stored in PostgreSQL)           │
    │ • Metrics Export (for external SLA reporting)              │
    └─────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│                            KEY INTEGRATION POINTS                             │
├──────────────────────────────────────────────────────────────────────────────┤
│ • Kafka: Metrics & log events published for external subscribers             │
│ • PostgreSQL: Alert history, incident timelines, audit logs                  │
│ • Docker Compose: Service discovery, health checks, networking               │
│ • Kong API Gateway: Request metering, rate-limit tracking, error logging     │
│ • Keycloak: Identity audit, failed login tracking, permission changes        │
│ • Prometheus Remote Storage: Optional integration for high-scale deployments  │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Summary

The **Monitoring & Observability System** provides:

✅ **Real-time Metrics:** Prometheus collects 1000+ data points/sec across all components  
✅ **Centralized Logs:** ELK pipeline aggregates 10,000+ log events/min  
✅ **Distributed Tracing:** Jaeger tracks end-to-end request latency across microservices  
✅ **Intelligent Alerting:** Alertmanager routes critical incidents to teams within seconds  
✅ **Rich Visualization:** Grafana dashboards provide incident commanders with situational awareness  
✅ **Compliance & Audit:** All operations logged and queryable for post-incident analysis  

---

**Next Steps:**
- Deploy using `docker-compose up` (see [Monitoring Setup](./MONITORING_OBSERVABILITY_SETUP.md))
- Configure dashboards (see [Grafana Guide](./MONITORING_OBSERVABILITY_GRAFANA_SETUP.md))
- Set up unit tests (see [Unit Testing Guide](./MONITORING_OBSERVABILITY_TESTING.md))
- Run load tests (see [Load Testing Guide](./MONITORING_OBSERVABILITY_LOAD_TESTING.md))
- Troubleshoot issues (see [FAQs](./MONITORING_OBSERVABILITY_FAQS.md))
