# Monitoring & Observability Setup Guide

**Version:** 1.0.0  
**Date:** May 7, 2026  
**Scope:** Complete configuration and deployment of Prometheus, Grafana, ELK, Jaeger, and Alertmanager

---

## Table of Contents

1. [Quick Start](#quick-start)
2. [Prometheus Configuration](#prometheus-configuration)
3. [Grafana Setup](#grafana-setup)
4. [ELK Stack Setup](#elk-stack-setup)
5. [Jaeger Configuration](#jaeger-configuration)
6. [Alertmanager Configuration](#alertmanager-configuration)
7. [Health Checks](#health-checks)

---

## Quick Start

### Deploy Full Stack (5 minutes)

```bash
# Clone repository
git clone https://github.com/Disaster-Response-System-Group-J/disaster-response-system.git
cd disaster-response-system

# Create environment file
cat > .env << 'EOF'
POSTGRES_USER=postgres
POSTGRES_PASSWORD=changeme
POSTGRES_DB=disaster_response
KEYCLOAK_ADMIN=admin
KEYCLOAK_ADMIN_PASSWORD=changeme
ALERTMANAGER_EMAIL_PASSWORD=your_gmail_app_password
EOF

# Start monitoring stack
docker-compose up -d prometheus alertmanager grafana elasticsearch logstash kibana filebeat jaeger

# Verify all services are running
docker-compose ps

# Check logs
docker-compose logs -f prometheus
```

### Access URLs

| Service | URL | Credentials |
|---------|-----|-------------|
| Prometheus | `http://localhost:9090` | None (read-only) |
| Grafana | `http://localhost:3030` | admin / admin |
| Kibana | `http://localhost:5601` | None (no auth) |
| Jaeger | `http://localhost:16686` | None |
| Alertmanager | `http://localhost:9093` | None |

---

## Prometheus Configuration

### 1. Basic Configuration

```yaml
# prometheus/prometheus.yml

global:
  scrape_interval: 15s        # Scrape every 15 seconds
  evaluation_interval: 15s    # Evaluate rules every 15 seconds
  scrape_timeout: 10s         # Wait 10s for response
  external_labels:
    monitor: 'disaster-response'
    environment: 'production'

# Alert management
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

# Load alert rules
rule_files:
  - 'alert_rules.yml'

# Scrape configurations
scrape_configs:
  # Prometheus itself
  - job_name: prometheus
    static_configs:
      - targets: ['prometheus:9090']
  
  # Kong API Gateway
  - job_name: kong
    metrics_path: /metrics
    static_configs:
      - targets: ['kong:8001']
  
  # Keycloak IAM
  - job_name: keycloak
    metrics_path: /metrics
    static_configs:
      - targets: ['keycloak:8080']
  
  # Grafana
  - job_name: grafana
    metrics_path: /metrics
    static_configs:
      - targets: ['grafana:3000']
  
  # PostgreSQL exporter
  - job_name: postgres
    static_configs:
      - targets: ['postgres-exporter:9187']
  
  # Kafka exporter
  - job_name: kafka
    static_configs:
      - targets: ['kafka-exporter:9308']
  
  # J1 Device Edge
  - job_name: j1-device-edge
    static_configs:
      - targets: ['j1-device-edge:8081']
  
  # J2 Data Intelligence
  - job_name: j2-data-intelligence
    static_configs:
      - targets: ['j2-data-intelligence:8082']
  
  # J3 DMS
  - job_name: j3-dms
    metrics_path: /api/metrics
    static_configs:
      - targets: ['j3-dms:3000']
  
  # Service discovery (optional)
  - job_name: 'services'
    file_sd_configs:
      - files:
          - 'sd_config.json'
    relabel_configs:
      - source_labels: [__meta_service_name]
        target_label: job
```

### 2. Alert Rules

```yaml
# prometheus/alert_rules.yml

groups:
  - name: disaster_response_alerts
    interval: 15s
    rules:
      
      # Service Availability
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
          component: core
        annotations:
          summary: "{{ $labels.job }} service is down"
          description: "{{ $labels.instance }} has been unavailable for > 1 minute"
      
      # PostgreSQL Health
      - alert: PostgreSQLDown
        expr: up{job="postgres"} == 0
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL is down"
          description: "Cannot connect to PostgreSQL for > 30 seconds"
      
      - alert: PostgreSQLConnectionHigh
        expr: pg_stat_activity_count > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL connection count is high"
          description: "{{ $value }} active connections (threshold: 80)"
      
      # Prometheus Health
      - alert: PrometheusStorageRunningOut
        expr: (prometheus_tsdb_blocks_bytes / (prometheus_tsdb_blocks_bytes + prometheus_tsdb_wal_storage_size_bytes)) * 100 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Prometheus storage running low"
          description: "Storage usage is {{ $value | humanize }}%"
      
      # Kafka Health
      - alert: KafkaConsumerLagHigh
        expr: kafka_consumer_lag > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Kafka consumer lag is high"
          description: "Consumer lag: {{ $value }} messages"
      
      # J3 DMS Performance
      - alert: J3APIHighLatency
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{job="j3-dms"}[5m])) > 1
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "J3 DMS API latency is high"
          description: "p95 latency: {{ $value | humanize }}s"
      
      # J2 Engine Health
      - alert: J2PredictionGenerationSlowdown
        expr: rate(predictions_generated_total[5m]) < 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "J2 prediction generation is slow"
          description: "Generation rate: {{ $value | humanize }}/sec"
```

### 3. Service Discovery Configuration

```json
// prometheus/sd_config.json

[
  {
    "labels": {
      "__meta_service_name": "j1-device-edge",
      "instance": "j1-device-edge:8081"
    }
  },
  {
    "labels": {
      "__meta_service_name": "j2-data-intelligence",
      "instance": "j2-data-intelligence:8082"
    }
  },
  {
    "labels": {
      "__meta_service_name": "j3-dms",
      "instance": "j3-dms:3000"
    }
  }
]
```

### 4. Storage Configuration

```yaml
# prometheus/prometheus.yml (storage section)

storage:
  tsdb:
    path: '/prometheus'
    retention:
      time: 24h          # Keep data for 24 hours
      size: 10GB         # Or max 10GB
    wal:
      sample_limit: 100000
      checkpoint_interval: 5m
```

**Data Retention Calculation:**
- Metrics per scrape: ~1,000
- Scrapes per hour: 240 (every 15s)
- Hours of retention: 24
- Total data points: 1,000 × 240 × 24 = 5.76M points
- Storage size: ~2-3GB (with compression)

---

## Grafana Setup

### 1. Install Grafana Plugins

```bash
# Access Grafana container
docker exec -it grafana bash

# Install plugins
grafana-cli plugins install grafana-piechart-panel
grafana-cli plugins install grafana-worldmap-panel
grafana-cli plugins install grafana-clock-panel

# Restart Grafana
exit
docker-compose restart grafana
```

### 2. Add Prometheus Datasource

**Via UI:**
1. Login to Grafana (`http://localhost:3030`)
2. Settings → Data Sources → Add data source
3. Select "Prometheus"
4. URL: `http://prometheus:9090`
5. Click "Save & Test"

**Via API:**

```bash
curl -X POST http://localhost:3030/api/datasources \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Prometheus",
    "type": "prometheus",
    "url": "http://prometheus:9090",
    "access": "proxy",
    "isDefault": true
  }'
```

### 3. Add Elasticsearch Datasource

```bash
curl -X POST http://localhost:3030/api/datasources \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Elasticsearch",
    "type": "elasticsearch",
    "url": "http://elasticsearch:9200",
    "access": "proxy",
    "database": "drs-logs-*",
    "jsonData": {
      "esVersion": "8.0.0",
      "timeField": "@timestamp",
      "logMessageField": "message"
    }
  }'
```

### 4. Import Dashboards

**System Overview Dashboard:**

```json
{
  "dashboard": {
    "title": "DRS System Overview",
    "panels": [
      {
        "title": "Service Availability",
        "targets": [
          {
            "expr": "up",
            "refId": "A"
          }
        ],
        "type": "stat"
      },
      {
        "title": "Request Rate",
        "targets": [
          {
            "expr": "rate(request_total[5m])",
            "refId": "A"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Error Rate",
        "targets": [
          {
            "expr": "rate(request_errors_total[5m]) / rate(request_total[5m])",
            "refId": "A"
          }
        ],
        "type": "gauge"
      }
    ]
  }
}
```

---

## ELK Stack Setup

### 1. Elasticsearch Initialization

```bash
# Create index template
curl -X PUT "http://localhost:9200/_index_template/drs-logs-template" -H 'Content-Type: application/json' -d'{
  "index_patterns": ["drs-logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0,
      "index.lifecycle.name": "drs-logs-policy",
      "index.lifecycle.rollover_alias": "drs-logs-write"
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "service_name": { "type": "keyword" },
        "log.level": { "type": "keyword" },
        "http_status_code": { "type": "integer" },
        "latency_ms": { "type": "integer" },
        "trace.id": { "type": "keyword" },
        "request_id": { "type": "keyword" },
        "message": { "type": "text" }
      }
    }
  }
}'

# Create ILM policy
curl -X PUT "http://localhost:9200/_ilm/policy/drs-logs-policy" -H 'Content-Type: application/json' -d'{
  "policy": "drs-logs-policy",
  "phases": {
    "hot": {
      "min_age": "0d",
      "actions": {
        "rollover": {
          "max_size": "50GB",
          "max_age": "1d"
        }
      }
    },
    "warm": {
      "min_age": "1d",
      "actions": {
        "set_priority": { "priority": 50 }
      }
    },
    "cold": {
      "min_age": "7d",
      "actions": {
        "set_priority": { "priority": 0 }
      }
    },
    "delete": {
      "min_age": "30d",
      "actions": {
        "delete": {}
      }
    }
  }
}'
```

### 2. Logstash Pipeline

```conf
# logstash/pipeline/logstash.conf

input {
  beats {
    port => 5044
  }
}

filter {
  mutate {
    add_field => { "service_name" => "%{[container][name]}" }
  }
  
  # J3 DMS JSON parsing
  if [container][name] =~ /j3-dms$/ {
    json {
      source => "message"
      target => "j3"
      skip_on_invalid_json => true
      tag_on_failure => ["_j3_json_parse_failure"]
    }
    
    if [j3][level] {
      mutate {
        add_field => { "log.level" => "%{[j3][level]}" }
      }
    }
    if [j3][requestId] {
      mutate {
        add_field => { "request_id" => "%{[j3][requestId]}" }
      }
    }
  }
  
  # Kong access log parsing
  if [container][name] =~ /kong$/ {
    grok {
      match => {
        "message" => "%{IPORHOST:client_ip} - %{DATA:user_name} \[%{HTTPDATE:kong_access_time}\] \"%{WORD:http_method} %{DATA:url_original} HTTP/%{NUMBER:http_version}\" %{NUMBER:http_status_code:int} %{NUMBER:http_response_body_bytes:int}"
      }
      tag_on_failure => ["_kong_grok_failure"]
    }
    
    date {
      match => ["kong_access_time", "dd/MMM/yyyy:HH:mm:ss Z"]
      target => "@timestamp"
    }
  }
  
  # Generic trace ID extraction
  grok {
    match => {
      "message" => [
        "(?:trace[_-]?id|traceId)[=: ]%{NOTSPACE:trace_id}",
        "(?:request[_-]?id|requestId)[=: ]%{NOTSPACE:request_id}"
      ]
    }
    break_on_match => false
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "drs-logs-%{+YYYY.MM.dd}"
    ilm_enabled => false
    manage_template => false
  }
}
```

### 3. Kibana Data View Setup

```bash
# Create data view
curl -X POST "http://localhost:5601/api/data_views/data_view" \
  -H 'Content-Type: application/json' \
  -H 'kbn-xsrf: true' \
  -d '{
    "data_view": {
      "title": "drs-logs-*",
      "timeFieldName": "@timestamp",
      "name": "DRS Logs"
    }
  }'
```

---

## Jaeger Configuration

### 1. Enable Jaeger in Services

**J3 DMS (Next.js):**

```typescript
// j3-system-interaction/dms/lib/tracing.ts

import { BasicTracerProvider, ConsoleSpanExporter, SimpleSpanProcessor } from '@opentelemetry/sdk-trace-node';
import { JaegerExporter } from '@opentelemetry/exporter-trace-jaeger';

const jaegerExporter = new JaegerExporter({
  endpoint: 'http://jaeger:6831',
  serviceName: 'j3-dms',
});

const tracerProvider = new BasicTracerProvider();
tracerProvider.addSpanProcessor(new SimpleSpanProcessor(jaegerExporter));

export const tracer = tracerProvider.getTracer('j3-dms-tracer');
```

### 2. Configure Jaeger Backend

```yaml
# jaeger/jaeger.yml

collector:
  port: 14250
  grpc:
    enabled: true
    host_port: ":4317"

agent:
  servers:
    - host: jaeger
      port: 6831

storage:
  type: memory
  memory:
    max_traces: 10000

sampler:
  type: probabilistic
  param: 0.1  # Sample 10% of traces
```

### 3. Trace Context Propagation

**HTTP Headers:**

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
tracestate: ext=some-value

# Standard W3C format:
# version-trace_id-parent_id-trace_flags
# 00 = version 0
# 4bf9...0e4736 = trace ID (32 hex chars)
# 00f0...02b7 = parent/span ID (16 hex chars)
# 01 = traced flag
```

---

## Alertmanager Configuration

### 1. Routing Configuration

```yaml
# alertmanager/alertmanager.yml

global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_auth_username: 'alerts@example.com'
  smtp_auth_password: '${ALERTMANAGER_EMAIL_PASSWORD}'
  smtp_from: 'alerts@example.com'

route:
  receiver: 'default'
  group_by: ['alertname', 'job', 'instance']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  
  routes:
    - match:
        severity: critical
      receiver: 'critical'
      group_wait: 0s
      repeat_interval: 1h
    
    - match:
        severity: warning
      receiver: 'warning'
      repeat_interval: 4h

receivers:
  - name: 'default'
    webhook_configs:
      - url: 'http://j3-dms:3000/api/alerts'
        send_resolved: true

  - name: 'critical'
    webhook_configs:
      - url: 'http://j3-dms:3000/api/alerts/critical'
        send_resolved: true
    email_configs:
      - to: 'oncall@example.com'
        headers:
          Subject: '[CRITICAL] {{ .GroupLabels.alertname }}'

  - name: 'warning'
    webhook_configs:
      - url: 'http://j3-dms:3000/api/alerts/warning'
        send_resolved: true

inhibit_rules:
  # Suppress warning if critical alert exists
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['job', 'instance']
```

### 2. Testing Alert Routing

```bash
# Send test alert
curl -X POST http://localhost:9093/api/v1/alerts \
  -H 'Content-Type: application/json' \
  -d '[{
    "status": "firing",
    "labels": {
      "alertname": "TestAlert",
      "severity": "critical",
      "job": "test"
    },
    "annotations": {
      "summary": "Test alert"
    },
    "startsAt": "2026-05-07T10:00:00Z",
    "endsAt": "0001-01-01T00:00:00Z"
  }]'

# Check alerts in UI
curl http://localhost:9093/api/v1/alerts
```

---

## Health Checks

### 1. Component Health Verification

```bash
#!/bin/bash
# scripts/health_check.sh

echo "=== Monitoring Stack Health Check ==="

# Prometheus
echo -n "Prometheus: "
if curl -s http://localhost:9090/-/healthy > /dev/null; then
  echo "✓ Healthy"
else
  echo "✗ Down"
fi

# Alertmanager
echo -n "Alertmanager: "
if curl -s http://localhost:9093/-/healthy > /dev/null; then
  echo "✓ Healthy"
else
  echo "✗ Down"
fi

# Grafana
echo -n "Grafana: "
if curl -s http://localhost:3030/api/health > /dev/null; then
  echo "✓ Healthy"
else
  echo "✗ Down"
fi

# Elasticsearch
echo -n "Elasticsearch: "
if curl -s http://localhost:9200/_cluster/health | grep -q green; then
  echo "✓ Green"
else
  echo "✗ Not Green"
fi

# Logstash
echo -n "Logstash: "
if curl -s http://localhost:9600/_node/stats | grep -q "status" > /dev/null; then
  echo "✓ Running"
else
  echo "✗ Down"
fi

# Jaeger
echo -n "Jaeger: "
if curl -s http://localhost:16686/api/services > /dev/null; then
  echo "✓ Healthy"
else
  echo "✗ Down"
fi

echo ""
echo "=== Data Flow Verification ==="

# Test metrics collection
echo -n "Prometheus scraping: "
SCRAPE_COUNT=$(curl -s 'http://localhost:9090/api/v1/query?query=up' | jq '.data.result | length')
echo "$SCRAPE_COUNT targets UP"

# Test log ingestion
echo -n "Elasticsearch indices: "
INDEX_COUNT=$(curl -s 'http://localhost:9200/_cat/indices?format=json' | jq 'length')
echo "$INDEX_COUNT indices"

# Test alert rules
echo -n "Alert rules loaded: "
RULE_COUNT=$(curl -s 'http://localhost:9090/api/v1/rules' | jq '.data.groups[0].rules | length')
echo "$RULE_COUNT rules"
```

---

**Next Steps:**
- Configure custom dashboards (see [Grafana Guide](./MONITORING_OBSERVABILITY_README.md#grafana--dashboards--visualization))
- Set up user authentication (see [Admin Manual](./MONITORING_OBSERVABILITY_ADMIN_MANUAL.md))
- Configure backup & recovery (see [Operational Runbook](./MONITORING_OBSERVABILITY_OPERATIONAL_RUNBOOK.md))
