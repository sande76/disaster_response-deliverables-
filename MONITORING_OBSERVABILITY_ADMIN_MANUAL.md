# Admin Manual — Monitoring & Observability

**Version:** 1.0.0  
**Date:** May 7, 2026  
**Audience:** System administrators, DevOps engineers, platform maintainers

---

## Table of Contents

1. [User & Access Management](#user--access-management)
2. [System Configuration](#system-configuration)
3. [Backup & Recovery](#backup--recovery)
4. [Scaling & Performance](#scaling--performance)
5. [Security & Hardening](#security--hardening)
6. [Monitoring the Monitoring](#monitoring-the-monitoring)
7. [Cost Optimization](#cost-optimization)

---

## User & Access Management

### 1. Grafana User Management

**Add New Admin User:**

```bash
curl -X POST http://localhost:3030/api/admin/users \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "name": "John Doe",
    "email": "john@example.com",
    "login": "john.doe",
    "password": "secure_password",
    "isAdmin": true
  }'
```

**Add Viewer-Only User (Read-Only):**

```bash
curl -X POST http://localhost:3030/api/admin/users \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "name": "Jane Viewer",
    "email": "jane@example.com",
    "login": "jane.viewer",
    "password": "password",
    "isAdmin": false,
    "role": "Viewer"
  }'
```

**Reset Admin Password:**

```bash
# In Grafana container
docker exec grafana grafana-cli admin reset-admin-password newpassword
```

### 2. RBAC (Role-Based Access Control)

**Grafana Roles:**
- **Admin:** Full access (create/delete dashboards, manage users)
- **Editor:** Can create/edit dashboards
- **Viewer:** Read-only access

**Assign Role to User:**

```bash
curl -X PUT http://localhost:3030/api/users/1/roles \
  -H "Content-Type: application/json" \
  -d '{"role": "Editor"}'
```

### 3. Kibana User Management

**Note:** Default Kibana has no authentication. For production, enable Elasticsearch security:

```bash
# Enable Elasticsearch security
curl -X POST http://localhost:9200/_security/user/kibana_admin \
  -H 'Content-Type: application/json' \
  -d '{
    "password": "changeme",
    "roles": ["kibana_admin"]
  }'

# Configure Kibana to use it
# In kibana.yml:
# elasticsearch.username: kibana_admin
# elasticsearch.password: changeme
```

---

## System Configuration

### 1. Prometheus Configuration Updates

**Modify Alert Rules (hot reload):**

```bash
# Edit alert_rules.yml
nano prometheus/alert_rules.yml

# Trigger reload (no downtime)
curl -X POST http://localhost:9090/-/reload

# Verify rules loaded
curl http://localhost:9090/api/v1/rules | jq '.data.groups[0].rules | length'
```

**Add New Scrape Target (hot reload):**

```bash
# Update sd_config.json (service discovery)
cat > prometheus/sd_config.json << 'EOF'
[
  {
    "labels": {
      "__meta_service_name": "new-service",
      "instance": "new-service:9090"
    }
  }
]
EOF

# Reload
curl -X POST http://localhost:9090/-/reload
```

### 2. Logstash Pipeline Updates

**Update Logstash Filter (requires restart):**

```bash
# Edit pipeline
nano logstash/pipeline/logstash.conf

# Restart container
docker-compose restart logstash

# Verify logs flowing
docker logs logstash | tail -20
```

### 3. Elasticsearch ILM Policy Updates

**Modify Index Lifecycle Policy:**

```bash
curl -X PUT "http://localhost:9200/_ilm/policy/drs-logs-policy" \
  -H 'Content-Type: application/json' \
  -d '{
    "policy": "drs-logs-policy",
    "phases": {
      "hot": {
        "min_age": "0d",
        "actions": {
          "rollover": {
            "max_size": "100GB",
            "max_age": "2d"
          }
        }
      },
      "delete": {
        "min_age": "60d",
        "actions": {
          "delete": {}
        }
      }
    }
  }'
```

---

## Backup & Recovery

### 1. Prometheus Data Backup

**Backup Metrics (TSDB blocks):**

```bash
# Create backup
docker exec prometheus tar czf /prometheus-backup-$(date +%Y%m%d).tar.gz /prometheus

# Copy from container
docker cp prometheus:/prometheus-backup-20260507.tar.gz ./backups/

# Restore (if needed)
tar xzf prometheus-backup-20260507.tar.gz -C /data/prometheus/
docker-compose restart prometheus
```

**Retention Policy (automatic cleanup):**
```yaml
# prometheus.yml
storage:
  tsdb:
    retention:
      time: 24h    # Keep 24 hours
      size: 50GB   # OR max 50GB
```

### 2. Elasticsearch Data Backup

**Backup Indices (snapshot):**

```bash
# Register snapshot repository
curl -X PUT "http://localhost:9200/_snapshot/backup" \
  -H 'Content-Type: application/json' \
  -d '{
    "type": "fs",
    "settings": {
      "location": "/snapshots"
    }
  }'

# Create snapshot
curl -X PUT "http://localhost:9200/_snapshot/backup/snapshot-1" \
  -H 'Content-Type: application/json' \
  -d '{
    "indices": "drs-logs-*",
    "ignore_unavailable": true,
    "include_global_state": false
  }'

# Monitor snapshot
curl http://localhost:9200/_snapshot/backup/snapshot-1

# Restore (if needed)
curl -X POST "http://localhost:9200/_snapshot/backup/snapshot-1/_restore"
```

### 3. Alertmanager Configuration Backup

```bash
# Backup config
cp alertmanager/alertmanager.yml backups/alertmanager-$(date +%Y%m%d).yml

# Store in version control
git add alertmanager/alertmanager.yml
git commit -m "chore: update alertmanager configuration"
```

---

## Scaling & Performance

### 1. Prometheus Scaling

**Problem:** High memory usage, slow queries

**Solution 1 - Federation (distribute scraping):**

```yaml
# Primary Prometheus scrapes secondary instances
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'secondary-prometheus'
    static_configs:
      - targets: ['secondary-prometheus:9090']
    scrape_interval: 15s
    metric_relabel_configs:
      - source_labels: [__name__]
        regex: '(prometheus_|up|alert_.*)'
        action: drop  # Only scrape important metrics
```

**Solution 2 - Remote Storage (Thanos, VictoriaMetrics):**

```yaml
# Send metrics to remote storage
global:
  external_labels:
    cluster: 'prod'
    region: 'us-west'

remote_write:
  - url: "http://thanos-receive:19291/api/v1/receive"
```

### 2. Elasticsearch Scaling

**Problem:** Slow queries, high disk usage

**Solution 1 - Data Tiering:**

```bash
# Create ILM policy with tiering
curl -X PUT "http://localhost:9200/_ilm/policy/tiered-policy" \
  -H 'Content-Type: application/json' \
  -d '{
    "phases": {
      "hot": {
        "actions": {
          "rollover": {"max_size": "50GB", "max_age": "1d"}
        }
      },
      "warm": {
        "min_age": "1d",
        "actions": {
          "set_priority": {"priority": 50},
          "forcemerge": {"max_num_segments": 1}
        }
      },
      "cold": {
        "min_age": "7d",
        "actions": {
          "set_priority": {"priority": 0},
          "searchable_snapshot": {}
        }
      }
    }
  }'
```

**Solution 2 - Index Sharding:**

```bash
# Update index template to use more shards
curl -X PUT "http://localhost:9200/_index_template/drs-logs-template" \
  -H 'Content-Type: application/json' \
  -d '{
    "index_patterns": ["drs-logs-*"],
    "template": {
      "settings": {
        "number_of_shards": 5,        # Increased from 1
        "number_of_replicas": 1       # For HA
      }
    }
  }'
```

### 3. Logstash Scaling (Horizontal)

**Run Multiple Logstash Instances:**

```yaml
# docker-compose.yml
services:
  logstash-1:
    image: logstash:8.6.0
    ports:
      - "5044:5044"
    environment:
      - pipeline.workers=8
      - pipeline.batch.size=1000
  
  logstash-2:
    image: logstash:8.6.0
    ports:
      - "5045:5044"
    environment:
      - pipeline.workers=8
      - pipeline.batch.size=1000

# Configure Filebeat to use load balancer
# filebeat.yml
output.logstash:
  hosts: ["logstash-1:5044", "logstash-2:5044"]
  loadbalance: true
```

---

## Security & Hardening

### 1. Enable HTTPS

**Prometheus (via Nginx reverse proxy):**

```nginx
# /etc/nginx/sites-available/prometheus
server {
    listen 443 ssl http2;
    server_name prometheus.example.com;
    
    ssl_certificate /etc/ssl/certs/prometheus.crt;
    ssl_certificate_key /etc/ssl/private/prometheus.key;
    
    location / {
        proxy_pass http://localhost:9090;
    }
}
```

**Grafana (built-in):**

```yaml
# grafana.ini
[server]
protocol = https
cert_file = /etc/ssl/certs/grafana.crt
cert_key = /etc/ssl/private/grafana.key
```

### 2. Network Segmentation

**Restrict Access to Monitoring Stack:**

```yaml
# docker-compose.yml
services:
  prometheus:
    ports:
      - "127.0.0.1:9090:9090"  # Localhost only
  
  alertmanager:
    ports:
      - "127.0.0.1:9093:9093"  # Localhost only

# Only allow internal access via Kong API Gateway
```

### 3. Data Encryption

**Elasticsearch Encryption at Rest:**

```yaml
# elasticsearch.yml
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.http.ssl.enabled: true
```

**Prometheus Encryption (via TLS reverse proxy):**

```bash
# Use Nginx or Kong to terminate TLS
# All traffic to Prometheus internally is unencrypted
# External traffic is encrypted via reverse proxy
```

---

## Monitoring the Monitoring

### 1. Alert on Prometheus Issues

```yaml
# alert_rules.yml
- alert: PrometheusDown
  expr: up{job="prometheus"} == 0
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Prometheus is down"

- alert: PrometheusHighMemory
  expr: process_resident_memory_bytes{job="prometheus"} > 2GB
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Prometheus memory usage high"

- alert: PrometheusHighDiskUsage
  expr: prometheus_tsdb_blocks_bytes > 50GB
  for: 5m
  labels:
    severity: warning
```

### 2. Alert on Elasticsearch Issues

```yaml
- alert: ElasticsearchClusterHealth
  expr: elasticsearch_cluster_health_status < 1
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Elasticsearch cluster not green"

- alert: ElasticsearchDiskSpace
  expr: elasticsearch_fs_data_available_bytes < 5GB
  for: 5m
  labels:
    severity: warning
```

### 3. Alert on Logstash Issues

```yaml
- alert: LogstashEventsDropped
  expr: logstash_queue_events_count > 1000
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Logstash queue backlog"
```

---

## Cost Optimization

### 1. Storage Cost Reduction

**Current Usage:**
- Prometheus: 2-3GB/day
- Elasticsearch: 5-10GB/day (varies by volume)
- Total/month: ~200GB = $10-20 (cloud storage)

**Optimizations:**

1. **Reduce Prometheus Retention:**
   ```yaml
   storage:
     tsdb:
       retention:
         time: 7d  # Instead of 24h
   ```
   Saves: ~50% storage

2. **Compress ELK Indices:**
   ```bash
   # Force merge + compress
   curl -X POST "http://localhost:9200/drs-logs-*/_forcemerge?max_num_segments=1"
   ```
   Saves: ~60% (via compression)

3. **Use ILM Tiering:**
   ```yaml
   hot: 50GB indices (fast SSD)
   warm: merged indices (standard disk)
   cold: compressed (archive storage)
   ```
   Saves: ~70% via tiering

**Estimated Savings:**
- Before: 200GB/month = $20/month
- After: 60GB/month = $6/month (70% reduction)

### 2. Compute Cost Reduction

**Rightsize Containers:**

```yaml
# docker-compose.yml

prometheus:
  deploy:
    resources:
      limits:
        memory: 500M    # Reduced from 2GB
        cpus: '1'

logstash:
  environment:
    - LS_JAVA_OPTS=-Xmx512m -Xms256m  # Reduced from 1GB

elasticsearch:
  environment:
    - "ES_JAVA_OPTS=-Xmx512m -Xms512m"  # Reduced from 2GB
```

**Estimated Savings:**
- Before: 4GB RAM total
- After: 1.5GB RAM total (62% reduction)

---

**Next Steps:**
- Set up automated backups (see [Operational Runbook](./MONITORING_OBSERVABILITY_OPERATIONAL_RUNBOOK.md))
- Configure alerting for admin tasks
- Document runbook procedures
