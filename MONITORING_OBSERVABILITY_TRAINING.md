# Training Manual — Monitoring & Observability

**Version:** 1.0.0  
**Date:** May 7, 2026  
**Audience:** New team members, on-call rotations, incident responders

---

## Training Objectives

By end of this training, you will be able to:
- ✅ Navigate all monitoring dashboards
- ✅ Interpret alerts and take appropriate action
- ✅ Search and analyze logs using Kibana
- ✅ Trace requests across services using Jaeger
- ✅ Handle common incidents with confidence
- ✅ Escalate properly when needed

**Estimated Duration:** 4-8 hours (self-paced)

---

## Module 1: Monitoring Fundamentals (1 hour)

### What is Observability?

**Three Pillars:**

```
METRICS          LOGS            TRACES
  ↓                ↓                ↓
[Prometheus]  [ELK Stack]      [Jaeger]
  ↓                ↓                ↓
Aggregated    Raw events       Request flow
counters,     with context     across services
gauges,
histograms
```

**Real Example:**

```
User reports: "Submitting SOS report is slow"

METRICS tell us:
- Request latency increased from 200ms → 2s
- Error rate went from 0.5% → 5%

LOGS show us:
- "Database connection failed" errors
- "Kafka broker unavailable"

TRACES reveal:
- J3 DMS: 50ms (normal)
- Kong: 20ms (normal)
- J2 Engine: 1.8s (BOTTLENECK)
  - Database query: 800ms (slow!)
  - ML model: 1000ms (normal)

Conclusion: Database is slow → needs investigation
```

### Disaster Response System Architecture

```
┌─────────────────────────────────────────────────┐
│              INCIDENT LIFECYCLE                 │
├─────────────────────────────────────────────────┤
│                                                  │
│ 1. USER REPORTS SOS                             │
│    ↓                                             │
│ 2. METRICS SPIKE (latency, errors)              │
│    ↓                                             │
│ 3. ALERT FIRES                                  │
│    ↓                                             │
│ 4. YOU INVESTIGATE (dashboards, logs, traces)  │
│    ↓                                             │
│ 5. ROOT CAUSE IDENTIFIED                        │
│    ↓                                             │
│ 6. ACTION TAKEN (restart, scale, fix)           │
│    ↓                                             │
│ 7. METRICS RETURN TO NORMAL                     │
│    ↓                                             │
│ 8. INCIDENT CLOSED                              │
└─────────────────────────────────────────────────┘
```

### Key Metrics to Know

| Metric | Normal Range | Warning | Critical |
|--------|-------------|---------|----------|
| Service UP | 1 | 0 (pending) | 0 (> 1m) |
| Error Rate | <1% | 1-5% | >5% |
| Latency (p95) | <200ms | 200-500ms | >500ms |
| Active Incidents | 0-10 | 11-50 | 51+ |

---

## Module 2: Grafana Dashboards (1 hour)

### Hands-On Lab

**Task 1: Navigate to Grafana**

1. Open browser: `http://localhost:3030`
2. Login: `admin / admin`
3. Click "Home" → "DRS System Overview"

**Expected Output:**

```
Dashboard shows green status ✓
- All services UP
- Error rate < 1%
- Latency < 200ms
```

**Task 2: Understand Panel Types**

| Panel | Shows | How to Read |
|-------|-------|------------|
| **Stat** | Single number | Color-coded (green/yellow/red) |
| **Graph** | Time series | Trend over time |
| **Gauge** | Percentage/bar | Visual fill % |
| **Table** | Rows/columns | Sorted by value |

**Task 3: Drill Down**

1. Click on "Request Rate" panel
2. Click "Inspect" → "Query"
3. See PromQL: `rate(request_total[5m])`
4. Click "Refresh" to update query

**Task 4: Set Alert Threshold**

1. Click "Alert" tab on any panel
2. Set condition: "Alert if value > 1000"
3. Set "For duration": 5m
4. Click "Save"

### Exercise

**Scenario:** Dashboard shows error rate spiked to 10% at 10:15 AM

**Your Tasks:**
1. Identify which service has errors
2. Find the first error log
3. Note the timestamp and error message
4. Take a screenshot for incident report

---

## Module 3: Alert Management (1 hour)

### Alert Lifecycle

```
FIRING (Alert triggered)
  ↓
[Acknowledge] - Tell team you're investigating
  ↓
[Investigate] - Use dashboards, logs, traces
  ↓
[Resolve] - Either auto-resolves or manual action
  ↓
RESOLVED (Alert cleared)
```

### Common Alerts

**Alert 1: ServiceDown**

```
What it means: Service is not responding to health checks
Severity: CRITICAL
Typical Response:
1. Check service status in docker: docker ps | grep service-name
2. Check recent logs: docker logs service-name
3. If stuck: docker restart service-name
4. Verify recovery: Check alert auto-resolves
```

**Alert 2: HighErrorRate**

```
What it means: > 5% of requests are failing
Severity: WARNING (can escalate to CRITICAL)
Typical Response:
1. Check error logs in Kibana
2. Find common error pattern
3. Identify which endpoint(s) failing
4. Check if related service is down
5. Take corrective action
```

**Alert 3: HighLatency**

```
What it means: API response time > 500ms
Severity: WARNING
Typical Response:
1. Check Jaeger traces for bottleneck
2. Identify slow service/database query
3. Check CPU/Memory utilization
4. May indicate need to scale
```

### Hands-On Lab

**Task 1: Trigger Test Alert**

```bash
# SSH into Alertmanager container
docker exec alertmanager sh -c 'curl -X POST http://alertmanager:9093/api/v1/alerts \
  -H "Content-Type: application/json" \
  -d "[{
    \"status\": \"firing\",
    \"labels\": {
      \"alertname\": \"TestAlert\",
      \"severity\": \"warning\"
    },
    \"annotations\": {
      \"summary\": \"Training test alert\"
    },
    \"startsAt\": \"$(date -u +'%Y-%m-%dT%H:%M:%SZ')\",
    \"endsAt\": \"0001-01-01T00:00:00Z\"
  }]"'
```

**Task 2: Acknowledge Alert**

1. Go to Grafana Alerts
2. Find "TestAlert"
3. Click "Acknowledge"
4. Add note: "Testing acknowledgement"

**Task 3: Silence Alert**

1. Click alert again
2. Click "Silence"
3. Choose duration: 15 minutes
4. Observe: Alert disappears from "Firing" list
5. Wait 15 min or manually "Unsilence"

---

## Module 4: Log Analysis with Kibana (1.5 hours)

### Understanding Log Structure

**Example Log Entry:**

```json
{
  "@timestamp": "2026-05-07T10:15:23.456Z",  ← When
  "service_name": "j3-dms",                   ← Which service
  "log.level": "ERROR",                       ← Severity
  "message": "Database connection failed",    ← What
  "request_id": "req-xyz-123",               ← Trace it
  "trace.id": "trace-abc-456",               ← Full trace
  "http_status_code": 500,                    ← Result
  "latency_ms": 1234,                         ← How long
  "container": { "name": "j3-dms", "id": "..." }
}
```

### Hands-On Lab

**Task 1: Basic Search**

1. Open Kibana: `http://localhost:5601`
2. Click "Discover"
3. See logs streaming in real-time
4. Set time: "Last 1 hour"

**Task 2: Find Errors**

```
Search: log.level: ERROR
Expected: See all ERROR level logs
Count: Usually < 10 per hour (normal)
```

**Task 3: Find Service-Specific Logs**

```
Search: service_name: "j3-dms"
Expected: Only J3 DMS logs
Filter more: Add "AND log.level: ERROR"
Result: Only J3 errors
```

**Task 4: Analyze Error Pattern**

```
Search: log.level: ERROR AND latency_ms: > 500
Expected: Slow errors
Group by: service_name
Result: Which service has slow errors?
```

**Task 5: Create Dashboard**

1. Create a visualization: "Error Rate by Hour"
2. Query: `log.level: ERROR` → count per hour
3. Save as dashboard widget

---

## Module 5: Trace Analysis with Jaeger (1.5 hours)

### Understanding Traces

**Trace = Full Request Journey**

```
User clicks "Submit Incident"
  ↓
J3 receives request (span: 0-50ms)
  ├─ Validate (5ms)
  ├─ Call Kong (10ms)
  │   └─ Kong routes to next service (2ms)
  ├─ Call J2 Engine (20ms)
  │   ├─ Database query (8ms)
  │   ├─ ML prediction (10ms)
  │   └─ Response (2ms)
  └─ Send response (5ms)
Total: 50ms
```

### Hands-On Lab

**Task 1: Access Jaeger**

1. Open `http://localhost:16686`
2. Service dropdown: Select "j3-dms"
3. Operation dropdown: Select "POST /api/incidents"
4. Click "Find Traces"

**Task 2: Analyze a Trace**

1. Click on any trace
2. See timeline on left
3. Identify longest spans (bars)
4. Click span to see details

**Task 3: Compare Fast vs. Slow**

1. Find a fast trace (200ms)
2. Find a slow trace (2s)
3. Compare side-by-side
4. Note where slowness occurs

**Task 4: Identify Bottleneck**

```
Scenario: Total trace = 2s (slow!)

Analysis:
J3: 100ms ✓
Kong: 20ms ✓
J2: 1800ms ✗ BOTTLENECK

Next step: Investigate J2
- Check J2 logs
- Check J2 database query
- Check J2 ML model latency
```

---

## Module 6: Incident Response Scenarios (2 hours)

### Scenario 1: Service is Down

**Alert:** ServiceDown - j1-device-edge

**Steps:**

1. **Acknowledge:** Grafana → Click alert → "Acknowledge"
2. **Verify Status:** 
   ```bash
   docker ps | grep j1-device-edge
   # If not running: restart
   docker-compose up -d j1-device-edge
   ```
3. **Check Logs:**
   ```
   Kibana → search: service_name: "j1-device-edge" 
          AND log.level: ERROR (last 5 min)
   ```
4. **Verify Recovery:**
   - Wait 1 minute
   - Alert should auto-resolve
   - Check service health dashboard

### Scenario 2: High Error Rate

**Alert:** HighErrorRate - j2-data-intelligence (10%)

**Steps:**

1. **Check Dashboard:** J2 dashboard → Error Rate panel
2. **Find Root Cause:** Kibana search
   ```
   service_name: "j2-data-intelligence" AND log.level: ERROR
   → Scroll through logs
   → Common error: "Out of Memory"?
   ```
3. **Check Resources:**
   ```
   Prometheus: check memory usage
   If high: Scale up J2 service
   docker-compose.yml: increase memory limit
   docker-compose up -d j2-data-intelligence
   ```
4. **Verify:** Check error rate drops

### Scenario 3: Slow API Response

**Alert:** J3APIHighLatency (p95 = 2s)

**Steps:**

1. **Verify Issue:** Grafana → J3 dashboard → Latency panel
2. **Get Trace:** Kibana → recent POST /api/incidents
   - Copy trace.id
3. **Open Jaeger:** Paste trace.id
4. **Find Bottleneck:** Which service/span is slow?
5. **Investigate:** 
   - If J2: Check ML model performance
   - If Kafka: Check consumer lag
   - If DB: Check slow queries
6. **Action:** Scale / optimize / restart
7. **Verify:** Latency returns to < 200ms

### Scenario 4: Out of Disk Space

**Alert:** PrometheusStorageRunningOut (85%)

**Steps:**

1. **Check Storage:**
   ```bash
   df -h /prometheus
   # Or via Prometheus UI:
   # Query: prometheus_tsdb_blocks_bytes
   ```
2. **Options:**
   - Reduce retention: 24h → 7d
   - Delete old blocks manually
   - Add new storage volume
3. **Take Action:**
   ```yaml
   # prometheus.yml
   storage:
     tsdb:
       retention:
         time: 7d  # Reduced from 24h
   ```
4. **Restart Prometheus**
   ```bash
   docker-compose restart prometheus
   ```

---

## Module 7: Escalation Procedures (30 minutes)

### When to Escalate

| Issue | Time to Resolve | Escalate If |
|-------|-----------------|------------|
| Service Down | 5 min | Not resolved in 5 min |
| High Error Rate | 10 min | Not improving after 10 min |
| High Latency | 15 min | Still high after investigation |
| Storage Full | 20 min | Can't free space or expand |

### Escalation Path

```
You (First Responder)
    ↓ (No resolution in time limit)
Senior Engineer (On-call)
    ↓ (Still unresolved)
Platform Lead (System owner)
    ↓ (Critical business impact)
Incident Commander (Mobilize team)
```

### Communication Template

```
When escalating, provide:
1. What is the issue?
   "Service X is down for Y minutes"

2. Impact?
   "Users cannot submit incidents"

3. What have you tried?
   "Restarted service, checked logs, found error: ..."

4. Current status?
   "Still investigating, needs senior review"

5. Any relevant artifacts?
   "Trace ID: trace-xyz, Alert: ServiceDown"
```

---

## Final Exam

**Scenario:** You receive this alert at 10:00 AM:

```
Alert: HighErrorRate - kong (8%)
Severity: WARNING
Duration: 5 minutes

Users report: Some requests getting "503 Service Unavailable"
```

**You Must:**

1. Acknowledge the alert (screenshot)
2. Find error logs in Kibana (show query, count errors)
3. Identify affected endpoints (which paths failing?)
4. Check if related service is down (which service?)
5. Find a slow trace in Jaeger (copy trace ID)
6. Hypothesize root cause (1-2 sentences)
7. Propose action (restart/scale/fix)
8. Provide incident summary (time to resolution, outcome)

**Correct Answer Example:**

```
1. ✓ Acknowledged at 10:01 AM
2. ✓ Found 420 ERROR logs in last 5 min from kong
3. ✓ Affected endpoints: POST /api/incidents, GET /api/status
4. ✓ J2-data-intelligence service is having connection issues
5. ✓ Trace ID: trace-xyz shows J2 timeout
6. ✓ Root cause: J2 database slow queries (800ms+)
7. ✓ Action: Restart J2 service to clear connection pool
8. ✓ Resolution: 12 minutes, error rate dropped to 0.2%
```

---

**Next Steps:**
- Complete training quiz
- Shadow senior engineer for 1 incident
- Join on-call rotation
- Refer to [FAQs](./MONITORING_OBSERVABILITY_FAQS.md) for common questions
