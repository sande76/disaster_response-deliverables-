# Operational Runbook — Monitoring & Observability

**Version:** 1.0.0  
**Date:** May 7, 2026  
**Audience:** Operations team, incident responders, on-call engineers

---

## Table of Contents

1. [Daily Operations](#daily-operations)
2. [Common Incidents](#common-incidents)
3. [Emergency Procedures](#emergency-procedures)
4. [Maintenance Windows](#maintenance-windows)
5. [Backup & Recovery](#backup--recovery)

---

## Daily Operations

### Morning Checklist (Every Day @ 09:00 UTC)

**Time: 5 minutes**

```bash
#!/bin/bash
# scripts/daily_health_check.sh

echo "=== Daily Monitoring Health Check ==="

# 1. Check all services running
echo "1. Service Status:"
docker-compose ps | grep -E "prometheus|alertmanager|grafana|elasticsearch|logstash|kibana|jaeger"

# 2. Verify no critical alerts
echo ""
echo "2. Active Critical Alerts:"
CRITICAL_ALERTS=$(curl -s http://localhost:9093/api/v1/alerts | jq '.data[] | select(.labels.severity=="critical") | .labels.alertname' | wc -l)
if [ $CRITICAL_ALERTS -eq 0 ]; then
  echo "✓ No critical alerts"
else
  echo "✗ $CRITICAL_ALERTS critical alerts firing - INVESTIGATE"
fi

# 3. Check storage usage
echo ""
echo "3. Storage Usage:"
echo "Prometheus:"
du -sh /data/prometheus | awk '{print "  Size: " $1}'

echo "Elasticsearch:"
curl -s http://localhost:9200/_cluster/health | jq '.active_shards_percent_as_number, .status'

# 4. Check data freshness
echo ""
echo "4. Data Freshness:"
echo "Last Prometheus scrape:"
curl -s 'http://localhost:9090/api/v1/query?query=up' | jq '.data.result | length' | awk '{print "  Targets UP: " $1}'

echo "Latest Elasticsearch index:"
curl -s 'http://localhost:9200/_cat/indices?format=json' | jq '.[-1].index' | head -1

# 5. Alert configuration
echo ""
echo "5. Alert Rules:"
curl -s 'http://localhost:9090/api/v1/rules' | jq '.data.groups[0].rules | length' | awk '{print "  Rules loaded: " $1}'

echo ""
echo "=== Daily Check Complete ==="
```

**Run:**
```bash
bash scripts/daily_health_check.sh
```

### Weekly Maintenance (Every Monday @ 02:00 UTC)

**Time: 30 minutes**

```bash
#!/bin/bash
# scripts/weekly_maintenance.sh

echo "=== Weekly Maintenance ==="

# 1. Backup Prometheus data
echo "1. Backing up Prometheus..."
docker exec prometheus tar czf /prometheus-backup.tar.gz /prometheus
docker cp prometheus:/prometheus-backup.tar.gz ./backups/prometheus-$(date +%Y%m%d).tar.gz
echo "✓ Prometheus backup complete"

# 2. Backup Elasticsearch indices
echo ""
echo "2. Creating Elasticsearch snapshot..."
curl -X PUT "http://localhost:9200/_snapshot/backup/weekly-$(date +%Y%m%d)" \
  -H 'Content-Type: application/json' \
  -d '{"indices":"drs-logs-*"}'
echo "✓ Elasticsearch snapshot created"

# 3. Verify ILM policies running
echo ""
echo "3. Verifying ILM policies..."
curl -s http://localhost:9200/_ilm/explain | jq '.indices | keys | length' | awk '{print "  Indices managed: " $1}'

# 4. Test alert routing
echo ""
echo "4. Testing alert routing..."
curl -X POST http://localhost:9093/api/v1/alerts \
  -H 'Content-Type: application/json' \
  -d '[{
    "status": "firing",
    "labels": {"alertname": "WeeklyTest", "severity": "warning"},
    "annotations": {"summary": "Weekly routing test"},
    "startsAt": "'"$(date -u +'%Y-%m-%dT%H:%M:%SZ')"'"
  }]'
echo "✓ Alert routing test sent"

# 5. Check disk space
echo ""
echo "5. Disk Space Check:"
echo "Prometheus:"
du -sh /data/prometheus
echo "Elasticsearch:"
du -sh /data/elasticsearch
echo "Logstash:"
du -sh /data/logstash

echo ""
echo "=== Weekly Maintenance Complete ==="
```

---

## Common Incidents

### Incident 1: Prometheus Scrape Failures

**Symptom:**
```
Alert: ScrapeErrors
- Gaps in metrics graphs
- Alert: "Prometheus Scrape Failing"
```

**Diagnosis:**

```bash
# Check Prometheus UI for failed scrapes
curl http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | select(.health=="down")'

# Check target health
# Output example:
# {
#   "labels": {"job": "j3-dms"},
#   "lastScrapeTime": "2026-05-07T10:00:00Z",
#   "lastError": "connection refused",
#   "health": "down"
# }
```

**Resolution:**

```bash
# Step 1: Verify target service is running
docker ps | grep j3-dms

# Step 2: If not running, restart
docker-compose up -d j3-dms

# Step 3: If running, check metrics endpoint
curl http://localhost:3000/api/metrics | head -20

# Step 4: If endpoint not responding, check logs
docker logs j3-dms | tail -20

# Step 5: If needed, reload Prometheus targets
curl -X POST http://localhost:9090/-/reload
```

**Recovery Time:** 2-5 minutes

---

### Incident 2: Elasticsearch Cluster Health Degraded

**Symptom:**
```
Alert: ElasticsearchUnhealthy
- Kibana slow or unresponsive
- Logs not indexing
- Status: RED or YELLOW (not GREEN)
```

**Diagnosis:**

```bash
# Check cluster health
curl http://localhost:9200/_cluster/health | jq .

# Output (unhealthy):
# {
#   "status": "red",
#   "number_of_nodes": 1,
#   "active_shards": 8,
#   "unassigned_shards": 5,
#   "initializing_shards": 0,
#   "relocating_shards": 0
# }
```

**Resolution:**

```bash
# Step 1: Check available disk space
df -h /data/elasticsearch

# Step 2: If < 5GB free, enable read-only
curl -X PUT "http://localhost:9200/_all/_settings" \
  -H 'Content-Type: application/json' \
  -d '{"settings":{"index.blocks.read_only_allow_delete":"true"}}'

# Step 3: Delete old indices to free space
curl -X DELETE "http://localhost:9200/drs-logs-2026.05.01"

# Step 4: Remove read-only block
curl -X PUT "http://localhost:9200/_all/_settings" \
  -H 'Content-Type: application/json' \
  -d '{"settings":{"index.blocks.read_only_allow_delete":null}}'

# Step 5: Check health recovered
curl http://localhost:9200/_cluster/health | jq '.status'
```

**Recovery Time:** 5-10 minutes

---

### Incident 3: Alert Manager Not Sending Notifications

**Symptom:**
```
Alert fires in Prometheus
But no email/webhook received
```

**Diagnosis:**

```bash
# Step 1: Check Alertmanager is running
docker ps | grep alertmanager

# Step 2: Check recent logs
docker logs alertmanager | tail -50

# Step 3: Verify configuration
docker exec alertmanager cat /etc/alertmanager/alertmanager.yml | grep -A 10 "receivers:"

# Step 4: Test webhook manually
curl -X POST http://localhost:9001/api/alerts \
  -H 'Content-Type: application/json' \
  -d '[{"status":"firing","labels":{"alertname":"Test"}}]'

# Step 5: Check webhook receiver is running
netstat -tlnp | grep 5001
```

**Resolution:**

```bash
# If configuration error:
# 1. Fix alertmanager.yml
# 2. Reload config:
curl -X POST http://localhost:9093/-/reload

# If webhook endpoint is down:
# 1. Check DMS service: docker ps | grep j3-dms
# 2. Check J3 logs: docker logs j3-dms
# 3. Restart: docker-compose up -d j3-dms

# If email not sending:
# 1. Verify SMTP credentials in .env
# 2. Check email server connectivity:
curl -X CONNECT smtp.gmail.com:587

# 3. If auth fails, use Google app password instead of regular password
```

**Recovery Time:** 5-15 minutes

---

### Incident 4: High Latency (API Responses Slow)

**Symptom:**
```
Alert: HighLatency (p95 > 500ms)
Users report slow incident submissions
```

**Resolution Flow:**

```
┌─ Step 1: Identify affected service
│  └─ Check Grafana dashboard for each service
│
├─ Step 2: Get traces
│  └─ Kibana search → find slow requests
│  └─ Copy trace IDs to Jaeger
│
├─ Step 3: Analyze trace
│  └─ Which span is slowest?
│  ├─ If J3: Check J3 CPU/memory
│  ├─ If J2: Check J2 database / ML model
│  ├─ If Kong: Check Kong connection pool
│  └─ If DB: Check slow query log
│
├─ Step 4: Take action
│  ├─ If resource starved: Scale service (↑ memory/CPU)
│  ├─ If connection pool empty: Restart service
│  ├─ If query slow: Add index / optimize
│  └─ If model slow: Use smaller model temporarily
│
└─ Step 5: Verify
   └─ Check latency returns to normal
```

**Example Investigation:**

```bash
# 1. Check Prometheus latency metric
curl 'http://localhost:9090/api/v1/query?query=histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))'

# 2. Query logs for traces with latency > 1s
# In Kibana:
# latency_ms: > 1000 AND http_status_code: 200

# 3. Copy trace ID and open in Jaeger
# Look at timeline, find longest span

# 4. If J2 ML is slow (> 1s):
# - Scale J2: increase memory
# docker-compose.yml: increase j2-data-intelligence memory: "2GB" → "4GB"
# docker-compose up -d j2-data-intelligence

# 5. Monitor latency recovery
# Re-run query every 1 minute until latency < 200ms
```

**Recovery Time:** 10-30 minutes (depending on root cause)

---

## Emergency Procedures

### EMERGENCY: System Completely Down

**Symptom:** All services down, all dashboards unavailable

**Response:**

```bash
#!/bin/bash
# IMMEDIATE ACTIONS (< 1 minute)

echo "EMERGENCY: System Down"

# 1. Check if Docker is running
docker ps
# If error: systemctl restart docker

# 2. Check compose file
cd /path/to/disaster-response-system
docker-compose status

# 3. Try restart all
docker-compose restart

# 4. If still down: full restart
docker-compose down
docker-compose up -d

# 5. Monitor logs
docker-compose logs -f prometheus

# 6. Verify services coming up (allow 30-60 seconds)
sleep 30
docker-compose ps | grep -E "Up|Exited"
```

**If services won't start:**

```bash
# 1. Check disk space
df -h
# If < 1GB free: DELETE OLD BACKUPS

# 2. Check logs for errors
docker-compose logs | head -100

# 3. Common fixes:
# - Port conflict: docker ps -a | grep 9090
# - Volume issues: docker volume ls | grep disaster
# - Network issues: docker network ls | grep disaster

# 4. Nuclear option (DATA LOSS - LAST RESORT):
# docker-compose down -v  # Delete all volumes
# docker-compose up -d     # Recreate from scratch
```

**Recovery Time:** 5-15 minutes

---

### EMERGENCY: Data Loss (Accidental Deletion)

**Symptom:** Important index/metrics accidentally deleted

**Recovery:**

```bash
# 1. STOP all write operations
docker-compose stop elasticsearch logstash

# 2. Restore from backup (if exists)
# For Elasticsearch:
curl -X POST "http://localhost:9200/_snapshot/backup/snapshot-1/_restore" \
  -H 'Content-Type: application/json' \
  -d '{
    "indices": "drs-logs-*",
    "rename_pattern": "(.+)",
    "rename_replacement": "restored-$1"
  }'

# 3. Verify restored indices
curl -s http://localhost:9200/_cat/indices | grep restored

# 4. Resume operations
docker-compose start elasticsearch logstash

# 5. Re-index if needed
```

**Prevention:** Enable automated snapshots (daily)

---

## Maintenance Windows

### Planned Prometheus Upgrade

**Notification Period:** 24-48 hours before

**Steps:**

```bash
# 1. Announce maintenance window to users
# "Monitoring will be unavailable for 30 min on May 15 @ 02:00 UTC"

# 2. Create backup
docker exec prometheus tar czf /prometheus-backup.tar.gz /prometheus
docker cp prometheus:/prometheus-backup.tar.gz ./backups/

# 3. Drain in-flight requests (graceful shutdown)
docker-compose down prometheus

# 4. Backup config
cp prometheus/prometheus.yml backups/prometheus.yml.bak

# 5. Update Prometheus version
# Edit docker-compose.yml: prom/prometheus:latest → prom/prometheus:new-version

# 6. Restart
docker-compose up -d prometheus

# 7. Verify
# Check http://localhost:9090 loads
# Check dashboards in Grafana
# Check alerts fire

# 8. Notify users of completion
```

### Planned Elasticsearch Reindex

**Steps:**

```bash
# 1. Create new index with better settings
curl -X PUT "http://localhost:9200/drs-logs-reindexed" \
  -H 'Content-Type: application/json' \
  -d '{
    "settings": {
      "number_of_shards": 5,
      "number_of_replicas": 1
    }
  }'

# 2. Reindex data
curl -X POST "http://localhost:9200/_reindex" \
  -H 'Content-Type: application/json' \
  -d '{
    "source": {"index": "drs-logs-*"},
    "dest": {"index": "drs-logs-reindexed"}
  }'

# 3. Update alias
curl -X POST "http://localhost:9200/_aliases" \
  -H 'Content-Type: application/json' \
  -d '{
    "actions": [
      {"remove": {"index": "drs-logs-*", "alias": "drs-logs"}},
      {"add": {"index": "drs-logs-reindexed", "alias": "drs-logs"}}
    ]
  }'

# 4. Delete old indices
curl -X DELETE "http://localhost:9200/drs-logs-old-*"
```

---

## Backup & Recovery

### Automated Backup Schedule

```bash
# Run daily via cron
0 3 * * * /path/to/backup_all.sh >> /var/log/monitoring-backup.log 2>&1
```

**Backup Script:**

```bash
#!/bin/bash
# backup_all.sh

BACKUP_DIR="/backups/monitoring"
DATE=$(date +%Y%m%d_%H%M%S)

# Prometheus
echo "Backing up Prometheus..."
docker exec prometheus tar czf /prometheus-backup.tar.gz /prometheus
docker cp prometheus:/prometheus-backup.tar.gz "$BACKUP_DIR/prometheus-$DATE.tar.gz"

# Elasticsearch
echo "Backing up Elasticsearch..."
curl -X PUT "http://localhost:9200/_snapshot/backup/snapshot-$DATE" \
  -H 'Content-Type: application/json' \
  -d '{"indices":"drs-logs-*"}'

# Config files
echo "Backing up configurations..."
tar czf "$BACKUP_DIR/configs-$DATE.tar.gz" \
  prometheus/prometheus.yml \
  prometheus/alert_rules.yml \
  alertmanager/alertmanager.yml \
  logstash/pipeline/logstash.conf

# Remove old backups (keep last 30 days)
find "$BACKUP_DIR" -name "*.tar.gz" -mtime +30 -delete

echo "Backup complete: $BACKUP_DIR"
```

### Disaster Recovery (Full Restoration)

**Time to Recovery: 1-2 hours**

```bash
# 1. Restore Prometheus data
tar xzf prometheus-20260507.tar.gz -C /data/

# 2. Restore Elasticsearch snapshot
curl -X POST "http://localhost:9200/_snapshot/backup/snapshot-20260507/_restore"

# 3. Restore config files
tar xzf configs-20260507.tar.gz -C ./

# 4. Restart all services
docker-compose restart

# 5. Verify all services healthy
bash scripts/health_check.sh
```

---

**On-Call Contact:** [Phone/Slack/Email]  
**Escalation:** See [Training Manual](./MONITORING_OBSERVABILITY_TRAINING.md#module-7-escalation-procedures)
