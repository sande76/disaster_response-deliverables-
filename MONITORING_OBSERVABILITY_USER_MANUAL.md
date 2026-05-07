# User Manual — Monitoring & Observability

**Version:** 1.0.0  
**Date:** May 7, 2026  
**Audience:** Incident responders, engineers, operators

---

## Table of Contents

1. [Getting Started](#getting-started)
2. [Dashboard Overview](#dashboard-overview)
3. [Alert Management](#alert-management)
4. [Log Search](#log-search)
5. [Trace Analysis](#trace-analysis)
6. [Common Tasks](#common-tasks)
7. [Troubleshooting](#troubleshooting)

---

## Getting Started

### Access the Monitoring System

**URLs:**
- **Grafana (Dashboards):** `http://localhost:3030`
- **Kibana (Logs):** `http://localhost:5601`
- **Jaeger (Traces):** `http://localhost:16686`
- **Alertmanager (Alerts):** `http://localhost:9093`

**Default Credentials:**
- Grafana: `admin / admin`
- Kibana: No authentication
- Alertmanager: No authentication

---

## Dashboard Overview

### 1. Main System Dashboard (Grafana)

**Location:** Grafana → Home → DRS System Overview

**Key Panels:**

| Panel | Purpose | Green | Yellow | Red |
|-------|---------|-------|--------|-----|
| **Service Availability** | Show which services are UP | All UP | 1-2 DOWN | 3+ DOWN |
| **Request Rate** | Requests per second | 100-500 req/s | 50-100 or 500-1000 | <50 or >1000 |
| **Error Rate** | % of failed requests | <1% | 1-5% | >5% |
| **Response Latency (p95)** | 95th percentile latency | <200ms | 200-500ms | >500ms |
| **Active Incidents** | Current disaster reports | 0-10 | 11-50 | 51+ |

**How to Read the Dashboard:**

1. **Green Status** → All systems normal, no action needed
2. **Yellow Status** → System degraded, monitor closely
3. **Red Status** → Critical issue, action required

**Example Scenario:**

```
Dashboard shows:
- Service Availability: J3-DMS DOWN
- Error Rate: 15% (RED)
- Response Latency: 2.5s (RED)

Action: Click on "J3-DMS" panel → View metric details → 
Check alert history → Identify root cause
```

### 2. Service-Specific Dashboards

**Navigate:** Grafana → Choose from:
- J1 Device Edge
- J2 Data Intelligence
- J3 DMS
- Kong API Gateway
- PostgreSQL
- Kafka

**Example: J3 DMS Dashboard**

```
┌─────────────────────────────────────┐
│ J3 DMS Performance Dashboard        │
├─────────────────────────────────────┤
│ API Latency (p95): 150ms ✓         │
│ Incidents Created (1h): 45         │
│ WebSocket Connections: 238          │
│ Error Rate: 0.2%                    │
│ Database Queries/sec: 250           │
└─────────────────────────────────────┘
```

**Click on any panel to drill down into details**

---

## Alert Management

### 1. View Active Alerts

**Grafana Path:** Alerts → Alert Rules

**Alert States:**
- **Pending:** Rule matched, waiting for `for` duration (e.g., 1 minute)
- **Firing:** Alert active, notification sent
- **Resolved:** Alert cleared, recovery notification sent

**Example Alert:**

```
Alert: ServiceDown
Severity: CRITICAL
Status: FIRING
Duration: 5 minutes

Description: j3-dms service is down
Instance: j3-dms:3000
Started at: 2026-05-07 10:15:00 UTC

Actions:
[Acknowledge] [Resolve] [View Details]
```

### 2. Acknowledge & Silence Alerts

**Acknowledge Alert (I've seen it):**
```
1. Click alert in Grafana
2. Click [Acknowledge] button
3. Add note: "Investigating database connection issue"
4. Click [Save]
```

**Silence Alert (Suppress notifications for 1h):**
```
1. Click alert
2. Click [Silence] button
3. Choose duration: 5m / 15m / 30m / 1h / custom
4. Click [Create Silence]
```

### 3. Recent Alerts (Alertmanager)

**Access:** `http://localhost:9093`

**View Alert History:**
1. Click "Alerts" tab
2. Filter by:
   - Status (firing, resolved)
   - Severity (critical, warning)
   - Job (j1-device-edge, j2-data-intelligence, etc.)

**Example:**
```
[Firing] [Critical] ServiceDown - j3-dms (5 min ago)
[Firing] [Warning] HighErrorRate - kong (2 min ago)
[Resolved] [Warning] PostgreSQLConnectionHigh (1 hour ago)
```

---

## Log Search

### 1. Access Kibana

**URL:** `http://localhost:5601`

**Left Panel → Discover** (search logs)

### 2. Basic Log Search

**Query Examples:**

| Search | Query | Purpose |
|--------|-------|---------|
| All errors | `log.level: ERROR` | Find all error logs |
| Specific service | `service_name: "j3-dms"` | Logs from J3 DMS only |
| Time range | Click calendar, select "Last 1 hour" | Focus on recent logs |
| Response time | `latency_ms: > 1000` | Find slow requests |
| Failed requests | `http_status_code: 500` | Find HTTP 500 errors |

**Advanced Query:**

```
service_name: "j3-dms" AND log.level: ERROR AND latency_ms: > 500
```

Returns: J3 DMS errors with latency > 500ms

### 3. Log Details

**Click on any log entry to expand:**

```
{
  "@timestamp": "2026-05-07T10:15:23.456Z",
  "service_name": "j3-dms",
  "log.level": "ERROR",
  "message": "Database connection failed",
  "request_id": "req-xyz-123",
  "trace.id": "trace-abc-456",
  "latency_ms": 1234,
  "http_status_code": 500,
  "http_method": "POST",
  "http_path": "/api/incidents"
}
```

**Click on trace.id to jump to Jaeger trace view**

### 4. Create Dashboard

**To create a custom dashboard:**

1. Click "Visualizations" → "Create visualization"
2. Choose chart type (line, bar, pie, etc.)
3. Add query (e.g., `error rate by service`)
4. Click "Save" → give it a name → "Add to Dashboard"

**Example: Error Rate by Service**

```
X-axis: service_name
Y-axis: COUNT of logs with level:ERROR
Result: Shows which service has most errors
```

---

## Trace Analysis

### 1. Access Jaeger

**URL:** `http://localhost:16686`

### 2. Search Traces

**Find a trace by:**

1. **Service:** Select from dropdown (j1-device-edge, j2-data-intelligence, j3-dms, kong)
2. **Operation:** Choose endpoint (e.g., POST /api/incidents)
3. **Trace ID:** Paste trace ID (from logs or annotations)

**Example Trace:**

```
Service: j3-dms
Operation: POST /api/incidents
Trace ID: trace-abc-123

Timeline:
│
├─ J3 DMS (50ms) ─────────┐
│                          │
│  ├─ Validate Request (5ms)
│  ├─ Call Kong (15ms)
│  │   └─ Kong route to J2
│  ├─ Call J2 API (20ms)
│  │   ├─ Query DB (8ms)
│  │   ├─ ML Prediction (10ms)
│  │   └─ Format response (2ms)
│  └─ Send response (10ms)
│
Total: 50ms
```

### 3. Analyze Latency

**Use Jaeger to find bottleneck:**

1. Open trace
2. Look for the **longest span** (bar)
3. Click on it to see details
4. Compare with expected duration
5. If > threshold: investigate that service

**Example:**

```
Scenario: J3 API slow (p95 = 2s instead of 200ms)

Trace Analysis:
├─ J3 DMS: 200ms ✓
├─ Kong: 50ms ✓
└─ J2 Engine: 1500ms ✗ (BOTTLENECK)

Action: Check J2 service for issues
         - Database query slow?
         - ML model inference slow?
         - Memory/CPU overloaded?
```

### 4. Compare Traces

**Compare fast vs. slow requests:**

1. Find a slow trace (e.g., 2s)
2. Copy trace ID
3. Find a fast trace (e.g., 200ms)
4. Open side-by-side
5. Compare timing of each span

---

## Common Tasks

### Task 1: I Got an Alert — What Do I Do?

**Alert: ServiceDown - j3-dms**

**Steps:**

1. **Acknowledge:** Click "Acknowledge" in Grafana (tells team you're investigating)
2. **Check Dashboard:** Open J3 DMS dashboard
3. **Search Logs:** Kibana → search for `service_name: j3-dms AND log.level: ERROR`
4. **Find Root Cause:** Look for error messages in logs
5. **Take Action:** Restart service / fix issue / escalate
6. **Verify Recovery:** Check that `up{j3-dms}` returns to 1
7. **Alert Resolves:** Grafana automatically resolves alert after 1 minute

### Task 2: User Reported Slow Response

**Scenario:** User says incident submission is slow

**Steps:**

1. **Check Dashboard:** Go to J3 DMS → look at "Response Latency" graph
2. **If Elevated:** Find recent slow request
3. **Get Trace ID:** Search Kibana for recent POST requests to /api/incidents
4. **Open Jaeger:** Paste trace ID → analyze span breakdown
5. **Identify Slow Service:** J2? Kong? J3?
6. **Check Service Logs:** Kibana search for that service's error logs
7. **Report:** Provide findings with trace ID to relevant team

### Task 3: Verify System Health (Pre-Deployment)

**Steps:**

1. **Open Grafana Dashboard** → System Overview
2. **Check All Panels are Green:**
   - ✓ All services UP
   - ✓ Error rate < 1%
   - ✓ Latency < 200ms
3. **Check No Active Alerts:**
   - Grafana → Alerts → Show "Firing" alerts (should be 0)
4. **Recent Logs Clean:**
   - Kibana → search last 5 minutes for ERROR logs (should be < 5)
5. **Green Light:** System ready for deployment

### Task 4: Investigate Performance Degradation

**Scenario:** Error rate spiked to 10% in last 30 minutes

**Steps:**

1. **View Alert:** Error rate alert should be firing
2. **Time Range:** Kibana → set time to "Last 30 minutes"
3. **Search Errors:** `log.level: ERROR` → count by service
4. **Identify Affected Service:** Which service has most errors?
5. **Look for Patterns:** Are errors from specific endpoint? User? Time?
6. **Check Resource Usage:** Prometheus dashboard → CPU/Memory/Disk
7. **Hypothesis:** Out of memory? Database slow? Kafka backlog?
8. **Remediation:** Restart service / scale up / increase cache

---

## Troubleshooting

### Q1: "I Don't See My Alert"

**Check:**
1. Is alert rule enabled? Grafana → Alerts → Rules
2. Does alert match current state? (e.g., is service actually down?)
3. Is alert in "Pending" state? (waiting for `for` duration)
4. Check Alertmanager routing: is it going to your receiver?

**Debug:**
```
# Query directly in Prometheus
curl 'http://localhost:9090/api/v1/query?query=up{job="j3-dms"}'

# Should return value=0 if service down
```

### Q2: "Logs are Missing"

**Check:**
1. Is Filebeat running? `docker ps | grep filebeat`
2. Is Logstash receiving events? Check Logstash logs
3. Is Elasticsearch indexing? Check index count: `curl http://localhost:9200/_cat/indices`
4. Check log volume: Kibana → time range → count total logs

### Q3: "Metrics Scrap Failing"

**Symptoms:**
- Gaps in Prometheus graphs
- Alert: PrometheusDown or ServiceDown

**Debug:**
```bash
# Test scrape manually
curl http://localhost:8080/metrics

# Check if exporter returns valid Prometheus format
# Should see: # HELP, # TYPE, metric_name value

# If fails: check exporter logs
docker logs j3-dms | grep metrics
```

### Q4: "Dashboard Shows Old Data"

**Solutions:**
1. **Refresh:** Press F5 to refresh page
2. **Clear Cache:** Ctrl+Shift+Delete → clear cache → refresh
3. **Set Time Range:** Click time picker → "Last 5 minutes" → refresh
4. **Check Prometheus:** Verify data being scraped
   ```
   curl http://localhost:9090/api/v1/query?query=up
   ```

### Q5: "Can't Acknowledge Alert"

**Troubleshooting:**
1. Ensure you're clicking [Acknowledge], not [Silence]
2. Check Grafana permissions (admin vs. viewer)
3. Try refreshing page
4. Check browser console for JavaScript errors

---

**Need Help?** See [FAQs & Troubleshooting](./MONITORING_OBSERVABILITY_FAQS.md) for more detailed troubleshooting.
