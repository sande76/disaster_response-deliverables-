# FAQs & Troubleshooting Guide — Monitoring & Observability

**Version:** 1.0.0  
**Date:** May 7, 2026  
**Purpose:** Quick answers to common questions and troubleshooting problems

---

## Table of Contents

1. [General Questions](#general-questions)
2. [Grafana FAQs](#grafana-faqs)
3. [Kibana FAQs](#kibana-faqs)
4. [Jaeger FAQs](#jaeger-faqs)
5. [Prometheus FAQs](#prometheus-faqs)
6. [Troubleshooting Checklist](#troubleshooting-checklist)

---

## General Questions

### Q1: What's the difference between metrics, logs, and traces?

**A:**

| Aspect | Metrics | Logs | Traces |
|--------|---------|------|--------|
| **What** | Numerical aggregates | Raw events with context | Request journey |
| **Example** | "100 req/s", "500ms latency" | "ERROR: DB connection failed" | Full execution path |
| **Volume** | Low (1KB/min per service) | High (10MB/min) | Medium (100KB/req) |
| **Tool** | Prometheus | ELK (Kibana) | Jaeger |
| **Query Speed** | Fast (milliseconds) | Medium (seconds) | Medium (seconds) |
| **Usage** | Dashboards, alerting | Debugging, analysis | Root cause analysis |

**When to Use Each:**
- **Metrics:** "Is system healthy?" → Check dashboard
- **Logs:** "Why did request fail?" → Search Kibana
- **Traces:** "Where is the bottleneck?" → Open Jaeger

### Q2: How often are metrics collected?

**A:** Every **15 seconds** (configurable in `prometheus.yml: scrape_interval`)

```yaml
# Fast: 5s scrape interval (more data, higher CPU)
scrape_interval: 5s

# Balanced (default): 15s
scrape_interval: 15s

# Slow: 60s (less data, lower CPU)
scrape_interval: 60s
```

**Trade-off:** More frequent = faster alerts but more storage/CPU

### Q3: How long is data retained?

**A:**

| System | Retention | Policy |
|--------|-----------|--------|
| **Prometheus** | 24 hours | `prometheus.yml: retention.time` |
| **Elasticsearch** | 30 days | ILM policy (auto-delete) |
| **Jaeger** | 7 days | `jaeger.yml: storage.memory.max_traces` |

**To Increase Retention:**

```yaml
# Prometheus: 7 days instead of 24h
# prometheus.yml
storage:
  tsdb:
    retention:
      time: 7d  # Increased from 24h
    # WARNING: increases storage to ~10GB
```

### Q4: Can I query historical data (older than retention)?

**A:** **No.** Once retention expires, data is deleted permanently.

**Solution:** Set up remote storage (Thanos) for long-term retention:
```yaml
# prometheus.yml
remote_write:
  - url: "http://thanos-receive:19291/api/v1/receive"
```

### Q5: How do I know if the monitoring system itself is healthy?

**A:** Use the health check script:

```bash
bash scripts/daily_health_check.sh
```

Or manually check each component:

```bash
# Prometheus
curl http://localhost:9090/-/healthy

# Alertmanager
curl http://localhost:9093/-/healthy

# Grafana
curl http://localhost:3030/api/health

# Elasticsearch
curl http://localhost:9200/_cluster/health | jq '.status'

# Logstash
curl http://localhost:9600/_node/stats

# Jaeger
curl http://localhost:16686/api/services
```

---

## Grafana FAQs

### Q1: How do I create a new dashboard?

**A:**

1. Click **"+"** icon (top left)
2. Select **"New Dashboard"**
3. Click **"Add Panel"**
4. Select visualization type (Graph, Stat, Gauge, etc.)
5. Choose datasource (Prometheus or Elasticsearch)
6. Write query (PromQL or KQL)
7. Click **"Apply"** → **"Save"**

### Q2: How do I add a Prometheus query?

**A:**

**Simple Query (single metric):**
```
up{job="j3-dms"}
```

**Rate (requests per second):**
```
rate(request_total{job="j3-dms"}[5m])
```

**Error Rate (%):**
```
(rate(request_errors_total[5m]) / rate(request_total[5m])) * 100
```

**Latency percentile:**
```
histogram_quantile(0.95, rate(request_duration_seconds_bucket[5m]))
```

### Q3: My dashboard shows "no data"

**A:** Troubleshooting steps:

1. **Check time range:**
   - Click time selector (top right)
   - Ensure it's "Last 1 hour" or similar
   - Avoid "Last 30 days" (might not have data)

2. **Verify query:**
   - Click panel → "Edit"
   - Check query looks correct
   - Click "Refresh" button

3. **Check datasource:**
   - Panel → "Edit" → "Datasource"
   - Ensure "Prometheus" is selected
   - Click "Test" button

4. **Verify metric exists:**
   - Go to Prometheus UI: `http://localhost:9090`
   - Type metric name in search: `up`
   - Should autocomplete

### Q4: How do I export a dashboard?

**A:**

1. Dashboard → Settings (gear icon)
2. Click **"Save as"** → **"Export as JSON"**
3. Can share JSON or publish to dashboard library

### Q5: How do I set up alerts on a panel?

**A:**

1. Panel → **"Edit"** → **"Alert"** tab
2. Click **"Create Alert"**
3. Configure:
   - **Condition:** "Alert if value > 1000"
   - **For:** "1m" (wait 1 min before firing)
   - **Notification channels:** (select email, webhook, etc.)
4. Click **"Save"**

---

## Kibana FAQs

### Q1: How do I search for logs from a specific time?

**A:**

1. Click time picker (top right)
2. Choose preset: "Last 1 hour", "Last 24 hours", etc.
3. Or set custom range: "From" and "To"
4. Click "Apply"

### Q2: How do I find a specific request by ID?

**A:**

```
request_id: "req-xyz-123"
```

**Or from trace ID:**

```
trace.id: "trace-abc-456"
```

### Q3: How do I export log data?

**A:**

1. Build your query/dashboard
2. Click **"Share"** (top right)
3. Copy URL (queryable dashboard link)
4. Or click **"Export"** → CSV

### Q4: Logs are missing for a service

**A:** Troubleshooting:

```bash
# 1. Check Filebeat is running
docker ps | grep filebeat

# 2. Check Logstash receiving data
docker logs logstash | grep "received events"

# 3. Check Elasticsearch indexing
curl 'http://localhost:9200/_cat/indices?format=json' | jq '.[] | select(.index | contains("drs-logs"))'

# 4. Manually send test log
curl -X POST http://localhost:5044/api/logs \
  -H 'Content-Type: application/json' \
  -d '{"message":"test","container":{"name":"test-service"}}'

# 5. Check if appears in Kibana after 1-2 seconds
```

### Q5: How do I create an alert in Kibana?

**A:**

1. Click **"Management"** (bottom left)
2. Click **"Alerting"** → **"Rules"**
3. Click **"Create Rule"**
4. Set condition: "When logs match: [query]"
5. Set threshold: "count > 100 in 5m"
6. Choose action: email, webhook, etc.
7. Click "Save"

---

## Jaeger FAQs

### Q1: How do I find a trace by trace ID?

**A:**

1. Open Jaeger: `http://localhost:16686`
2. Top-left search box → Paste trace ID
3. Click "Find Traces"

### Q2: What's the difference between span and trace?

**A:**

```
TRACE (entire request journey)
  │
  ├─ Span 1: Service A (50ms)
  │   ├─ Sub-span: Database query (10ms)
  │   └─ Sub-span: Cache lookup (5ms)
  │
  ├─ Span 2: Service B (30ms)
  │
  └─ Span 3: Service C (20ms)

Total trace time: 100ms
```

**Span** = single operation in a service  
**Trace** = collection of all spans for one request

### Q3: How do I find slow requests?

**A:**

1. Select service (e.g., "j3-dms")
2. Select operation (e.g., "POST /api/incidents")
3. Set "Min Duration": "1s" (find traces > 1s)
4. Click "Find Traces"

### Q4: How do I compare two traces?

**A:**

1. Find first trace → copy trace ID
2. Find second trace
3. Open side-by-side:
   - Click "Compare Traces" (if available)
   - Or open two browser tabs

### Q5: Why don't I see all traces?

**A:** Traces are **sampled** to reduce storage:

```yaml
# Jaeger config: 10% sampling
sampler:
  type: probabilistic
  param: 0.1  # Only keep 10% of traces
```

**To increase sampling (higher storage cost):**
```yaml
sampler:
  param: 1.0  # Keep all traces (100%)
```

---

## Prometheus FAQs

### Q1: How do I test a PromQL query?

**A:**

1. Go to Prometheus UI: `http://localhost:9090`
2. Click **"Graph"** tab
3. Paste query in search box
4. Click "Execute"
5. See results as graph or table

### Q2: How do I list all available metrics?

**A:**

```
# In Prometheus graph tab, click the metric autocomplete
# Or use HTTP API:

curl 'http://localhost:9090/api/v1/label/__name__/values' | jq .
```

### Q3: How do I write a complex PromQL query?

**A:**

```
# Basic rate calculation
rate(http_requests_total[5m])

# Add filters
rate(http_requests_total{job="j3-dms"}[5m])

# Mathematical operations
rate(errors_total[5m]) / rate(requests_total[5m]) * 100

# Aggregation
sum(rate(requests_total[5m])) by (job)

# Histogram percentile
histogram_quantile(0.95, rate(request_duration_bucket[5m]))

# Alert condition
rate(errors_total[5m]) > 100
```

### Q4: Why is my alert not firing?

**A:** Checklist:

- [ ] Rule exists in `prometheus/alert_rules.yml`
- [ ] Config reloaded: `curl -X POST http://localhost:9090/-/reload`
- [ ] PromQL expression is valid (test in UI)
- [ ] Expression matches current data
- [ ] `for` duration has elapsed (e.g., 1m)
- [ ] Alertmanager is running and configured
- [ ] Webhook receiver is up

### Q5: How do I temporarily disable an alert?

**A:** Comment out in `prometheus/alert_rules.yml`:

```yaml
rules:
  # - alert: ServiceDown
  #   expr: up == 0
  #   for: 1m
  
  - alert: AnotherAlert
```

Then reload:
```bash
curl -X POST http://localhost:9090/-/reload
```

---

## Troubleshooting Checklist

### Problem: Services show as DOWN

**Checklist:**

- [ ] Container running: `docker ps | grep service-name`
- [ ] If not: `docker-compose up -d service-name`
- [ ] Metrics endpoint responding: `curl http://localhost:port/metrics`
- [ ] If not: check service logs: `docker logs service-name`
- [ ] Firewall blocking: check port with `netstat -tlnp | grep port`
- [ ] Prometheus refresh: wait 30s for next scrape

**If still down:** Try restart
```bash
docker-compose restart service-name
```

### Problem: "No data in dashboard"

**Checklist:**

- [ ] Correct time range selected (not "Last 30 days" if only 1 day data)
- [ ] Metric exists in Prometheus
- [ ] Query is valid (test in Prometheus UI)
- [ ] Datasource is "Prometheus" not "Elasticsearch"
- [ ] Try refreshing dashboard: F5

### Problem: Alert not sending email

**Checklist:**

- [ ] Alertmanager running: `docker ps | grep alertmanager`
- [ ] Email credentials correct in `.env`
- [ ] SMTP server accessible: `curl smtp.gmail.com:587`
- [ ] Gmail app password (not regular password): [get here](https://myaccount.google.com/apppasswords)
- [ ] Receiver configured in `alertmanager.yml`
- [ ] Webhook test: `curl -X POST http://localhost:9093/api/v1/alerts`

### Problem: Logs not appearing

**Checklist:**

- [ ] Filebeat running: `docker ps | grep filebeat`
- [ ] Logstash running: `docker ps | grep logstash`
- [ ] Elasticsearch running: `docker ps | grep elasticsearch`
- [ ] Logs flowing: `docker logs logstash | grep received`
- [ ] Index exists: `curl http://localhost:9200/_cat/indices`
- [ ] Time range: Set to "Last 5 minutes" in Kibana

### Problem: Elasticsearch slow/unresponsive

**Checklist:**

- [ ] Disk space: `df -h` (need > 5GB free)
- [ ] Cluster health: `curl http://localhost:9200/_cluster/health`
- [ ] Memory usage: `docker stats elasticsearch`
- [ ] Number of indices: `curl 'http://localhost:9200/_cat/indices?format=json' | jq length`
- [ ] If yellow/red: delete old indices
  ```bash
  curl -X DELETE "http://localhost:9200/drs-logs-2026.05.01"
  ```

### Problem: "Too many connections" error

**Checkl**

- [ ] Check active connections: `curl http://localhost:5432/metrics | grep pg_stat_activity`
- [ ] Connections > 80? May need to scale
- [ ] Restart PostgreSQL:
  ```bash
  docker-compose restart postgres
  ```
- [ ] Clear Logstash connections:
  ```bash
  docker-compose restart logstash
  ```

---

## Quick Reference

### Command Cheat Sheet

```bash
# Health checks
curl http://localhost:9090/-/healthy          # Prometheus
curl http://localhost:9093/-/healthy          # Alertmanager
curl http://localhost:3030/api/health        # Grafana
curl http://localhost:9200/_cluster/health   # Elasticsearch

# View data
curl http://localhost:9090/api/v1/targets                    # Scrape targets
curl 'http://localhost:9200/_cat/indices?v'                 # Indices
curl http://localhost:16686/api/services                     # Jaeger services

# Restart services
docker-compose restart prometheus
docker-compose restart alertmanager
docker-compose restart elasticsearch

# Check logs
docker logs prometheus | tail -50
docker logs alertmanager | tail -50
docker logs elasticsearch | tail -50

# Test endpoints
curl -X POST http://localhost:9093/api/v1/alerts -d '[{"status":"firing","labels":{"alertname":"Test"}}]'

# Reload configs (no downtime)
curl -X POST http://localhost:9090/-/reload              # Prometheus
curl -X POST http://localhost:9093/-/reload              # Alertmanager
```

### Useful URLs

| Purpose | URL |
|---------|-----|
| Prometheus query | `http://localhost:9090` |
| Grafana dashboards | `http://localhost:3030` |
| Kibana logs | `http://localhost:5601` |
| Jaeger traces | `http://localhost:16686` |
| Alertmanager alerts | `http://localhost:9093` |
| Elasticsearch APIs | `http://localhost:9200` |

---

**Need more help?** Contact the on-call engineer via [Slack/Email/Phone]  
**Report bug:** [GitHub Issues](https://github.com/Disaster-Response-System-Group-J/disaster-response-system/issues)
