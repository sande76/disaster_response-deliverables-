# Monitoring & Observability Algorithm

**Version:** 1.0.0  
**Date:** May 7, 2026  
**Scope:** Core algorithms for metrics collection, analysis, alerting, and anomaly detection in the Disaster Response System observability layer

---

## Table of Contents

1. [Overview](#overview)
2. [Metrics Collection Algorithm](#metrics-collection-algorithm)
3. [Alert Evaluation & Firing Algorithm](#alert-evaluation--firing-algorithm)
4. [Alert Routing & Deduplication Algorithm](#alert-routing--deduplication-algorithm)
5. [Log Processing Pipeline Algorithm](#log-processing-pipeline-algorithm)
6. [Distributed Trace Correlation Algorithm](#distributed-trace-correlation-algorithm)
7. [Anomaly Detection Algorithm](#anomaly-detection-algorithm)
8. [Health Check & Recovery Algorithm](#health-check--recovery-algorithm)
9. [Data Retention & Compaction Algorithm](#data-retention--compaction-algorithm)

---

## Overview

The observability system operates on a **reactive + proactive** model:

- **Reactive:** Services emit metrics/logs/traces → collectors ingest → dashboards display
- **Proactive:** Rules evaluate metrics → alerts fire → routes dispatch → engineers notified

**Key Principles:**
- **Pull-based metrics:** Prometheus asks services for current state (vs. services pushing)
- **Event-driven logs:** Logs stream in real-time; Logstash processes on arrival
- **Sampling-based traces:** Jaeger samples traces to manage storage costs
- **Deterministic alerting:** Same input always produces same alert state

---

## Metrics Collection Algorithm

### 1. Prometheus Scrape Cycle

**Input:** Target list (jobs from `prometheus.yml` + service discovery)  
**Output:** Time-series data points stored in TSDB

**Algorithm:**

```
FUNCTION prometheus_scrape_cycle():
  FOR each scrape_interval (15 seconds):
    FOR each job in scrape_configs:
      FOR each target in job.targets:
        
        // Step 1: Fetch metrics endpoint
        SET request_timeout = 10s
        TRY:
          response = HTTP_GET(target.url + target.metrics_path, timeout=10s)
          IF response.status != 200:
            MARK_FAILURE(target, "HTTP " + response.status)
            CONTINUE
        CATCH timeout_exception:
          MARK_FAILURE(target, "Scrape timeout")
          CONTINUE
        CATCH connection_exception:
          MARK_FAILURE(target, "Connection refused")
          CONTINUE
        
        // Step 2: Parse Prometheus text format
        metrics = PARSE_TEXT_FORMAT(response.body)
        IF metrics is EMPTY:
          MARK_FAILURE(target, "No metrics returned")
          CONTINUE
        
        // Step 3: Add labels & metadata
        FOR each metric in metrics:
          metric.labels['job'] = job.job_name
          metric.labels['instance'] = target.instance_name
          metric.labels['__scrape_time'] = CURRENT_TIME()
        
        // Step 4: Write to TSDB
        FOR each metric in metrics:
          tsdb_write(
            metric_name = metric.name,
            timestamp = CURRENT_TIME_UNIX(),
            value = metric.value,
            labels = metric.labels
          )
        
        // Step 5: Record metadata
        SET up_metric = (response.status == 200) ? 1 : 0
        tsdb_write(
          metric_name = 'up',
          timestamp = CURRENT_TIME_UNIX(),
          value = up_metric,
          labels = metric.labels
        )

ENDFOR
ENDFOR
ENDFOR
```

**Time Complexity:** O(n * m) where n = jobs, m = targets per job  
**Space Complexity:** O(k) where k = number of unique time-series  
**Parallelization:** Scrape jobs run independently (no interdependencies)

### 2. TSDB Write & Compaction

**Input:** Metrics from scraper  
**Output:** Compressed time-series blocks

**Algorithm:**

```
FUNCTION tsdb_write(metric_name, timestamp, value, labels):
  // Step 1: Create or fetch series key
  series_key = HASH(metric_name + SORTED(labels))
  series = tsdb_memory_cache.get_or_create(series_key)
  
  // Step 2: Append data point
  series.append({
    timestamp: timestamp,
    value: value
  })
  
  // Step 3: Check if in-memory block should flush
  IF series.size() > memory_threshold (default: 10MB):
    tsdb_flush_to_disk(series)
    series.clear()
  
  // Step 4: Update metadata indices
  labels_index.add(series_key, labels)
  time_index.add(timestamp)

ENDFUNCTION

FUNCTION tsdb_compaction():
  // Background process runs every hour
  FOR each block_file in tsdb/blocks:
    IF block_file.age_hours > 2:
      FOR each 2 consecutive blocks:
        merged_block = MERGE_BLOCKS(block_1, block_2)
        COMPRESS_BLOCK(merged_block, codec='snappy')
        WRITE_DISK(merged_block)
        DELETE(block_1, block_2)

ENDFUNCTION
```

**Space Efficiency:** Snappy compression achieves ~3:1 ratio for time-series data

---

## Alert Evaluation & Firing Algorithm

### 1. Rule Evaluation Loop

**Input:** Alert rules from `alert_rules.yml`; current metrics from TSDB  
**Output:** Alert firing state (pending/firing/resolved)

**Algorithm:**

```
FUNCTION alert_evaluation_loop():
  FOR each evaluation_interval (15 seconds):
    FOR each alert_rule in alert_rules:
      
      // Step 1: Query metrics using PromQL
      current_value = prometheus_query(
        rule.expr,
        query_time = CURRENT_TIME(),
        range = rule.for_duration
      )
      
      // Step 2: Check if rule expression matches
      IF current_value > 0 (expression is true):
        
        // Step 3: Check if alert has already fired
        existing_alert = alert_state.get(rule.name)
        
        IF existing_alert is NULL:
          // First time matching; create pending alert
          pending_alert = {
            rule_name: rule.name,
            status: 'PENDING',
            fire_time: CURRENT_TIME() + rule.for_duration,
            value: current_value,
            labels: EXTRACT_LABELS(current_value),
            annotations: RENDER_ANNOTATIONS(rule, current_value)
          }
          alert_state.store(rule.name, pending_alert)
        
        ELSEIF existing_alert.status == 'PENDING':
          // Check if waiting period elapsed
          IF CURRENT_TIME() >= existing_alert.fire_time:
            existing_alert.status = 'FIRING'
            SEND_ALERT_TO_ALERTMANAGER(existing_alert)
        
        ELSEIF existing_alert.status == 'FIRING':
          // Keep alert firing; update value
          existing_alert.value = current_value
      
      ELSE:
        // Expression no longer matches
        existing_alert = alert_state.get(rule.name)
        
        IF existing_alert is NOT NULL AND existing_alert.status == 'FIRING':
          // Generate RESOLVED notification
          resolved_alert = CLONE(existing_alert)
          resolved_alert.status = 'RESOLVED'
          resolved_alert.resolved_time = CURRENT_TIME()
          SEND_ALERT_TO_ALERTMANAGER(resolved_alert)
          
          alert_state.delete(rule.name)

ENDFOR
ENDFOR
ENDFUNCTION
```

**Example: ServiceDown Alert**

```
Rule: up == 0
For: 1m (60 seconds)
Scrape interval: 15s

Timeline:
T=0s:    Scraper gets up=0, creates PENDING alert (fire_time = T+60s)
T=15s:   up still 0, alert remains PENDING
T=30s:   up still 0, alert remains PENDING
T=45s:   up still 0, alert remains PENDING
T=60s:   up still 0, AND fire_time elapsed → FIRING (notify)
T=75s:   Service recovers, up=1, FIRING alert → RESOLVED (notify)
```

### 2. Alert Label Extraction

**Purpose:** Capture label values from time-series for alert routing

**Algorithm:**

```
FUNCTION extract_labels(metric_value):
  // metric_value contains {value, labels}
  
  alert_labels = {
    job: metric_value.labels.job,
    instance: metric_value.labels.instance,
    severity: rule.severity,  // From rule definition
    environment: 'production'  // Static
  }
  
  // If rule expression uses regex/filters, extract those too
  IF rule.expr contains 'job=~"j[0-9]"':
    alert_labels.job_group = EXTRACT_GROUP(rule.expr, metric_value)
  
  RETURN alert_labels

ENDFUNCTION
```

---

## Alert Routing & Deduplication Algorithm

### 1. Alert Grouping

**Input:** Alert from Prometheus  
**Output:** Grouped alert sent to receiver

**Algorithm:**

```
FUNCTION alertmanager_group_alerts():
  incoming_alert = RECEIVE_FROM_PROMETHEUS()
  
  // Step 1: Create group key
  group_key = HASH({
    alertname: incoming_alert.labels.alertname,
    job: incoming_alert.labels.job,
    // ... other fields in route.group_by
  })
  
  // Step 2: Find or create group
  group = alert_groups.get_or_create(group_key)
  group.add_alert(incoming_alert)
  
  // Step 3: Determine wait time
  IF incoming_alert.severity == 'critical':
    wait_time = 0s  (send immediately)
  ELSEIF incoming_alert.severity == 'warning':
    wait_time = 30s  (batch with other warnings)
  ELSE:
    wait_time = 10s
  
  // Step 4: Schedule group for firing
  group.next_fire_time = CURRENT_TIME() + wait_time
  timer.schedule(group_key, wait_time, send_group_notification)

ENDFUNCTION
```

### 2. Deduplication & Suppression

**Input:** Grouped alerts  
**Output:** Deduplicated alert set

**Algorithm:**

```
FUNCTION deduplicate_suppress(group):
  
  // Step 1: Deduplicate by fingerprint
  fingerprints_seen = SET()
  unique_alerts = EMPTY_LIST()
  
  FOR each alert in group.alerts:
    fingerprint = HASH(alert.labels)
    IF fingerprint NOT IN fingerprints_seen:
      unique_alerts.append(alert)
      fingerprints_seen.add(fingerprint)
    ELSE:
      // Duplicate; skip
  
  // Step 2: Apply inhibition rules
  FOR each inhibit_rule in inhibit_rules:
    FOR each alert in unique_alerts:
      IF MATCHES_INHIBIT_RULE(alert, inhibit_rule):
        // Suppress this alert (don't send)
        unique_alerts.remove(alert)
  
  // Example inhibit rule:
  // IF alert.severity == 'warning' AND
  //    EXISTS alert with same job and severity=='critical':
  //   THEN suppress warning
  
  RETURN unique_alerts

ENDFUNCTION
```

### 3. Alert Delivery

**Input:** Deduplicated alerts  
**Output:** Notifications sent to receivers (webhook, email, Slack)

**Algorithm:**

```
FUNCTION send_alert_notification(group):
  
  // Step 1: Find receiver based on route
  receiver = route_match(group.alerts[0])  // Check first alert's labels
  
  // Step 2: Format notification payload
  payload = {
    status: 'firing' or 'resolved',
    alerts: group.alerts,
    groupLabels: group.group_key,
    timestamp: CURRENT_TIME()
  }
  
  // Step 3: Send to each receiver type
  FOR each receiver_config in receiver.receivers:
    
    IF receiver_config.type == 'webhook':
      HTTP_POST(
        url = receiver_config.url,
        body = JSON(payload),
        timeout = 5s
      )
    
    ELSEIF receiver_config.type == 'email':
      email = CREATE_EMAIL(
        from = 'alertmanager@disaster-response.local',
        to = receiver_config.to,
        subject = FORMAT_SUBJECT(group.alerts),
        body = FORMAT_BODY(group.alerts)
      )
      SMTP_SEND(email)
    
    ELSEIF receiver_config.type == 'slack':
      message = FORMAT_SLACK_MESSAGE(group.alerts)
      SLACK_POST(channel = receiver_config.channel, message = message)
  
  // Step 4: Schedule repeat notification
  repeat_time = CURRENT_TIME() + receiver.repeat_interval
  timer.schedule(group_key, repeat_time, send_alert_notification)

ENDFUNCTION
```

**Key Point:** If same alert matches at `repeat_interval`, notification is sent again (prevents alert fatigue via repeat_interval tuning)

---

## Log Processing Pipeline Algorithm

### 1. Filebeat Collection

**Input:** Docker container stdout/stderr  
**Output:** Log events sent to Logstash

**Algorithm:**

```
FUNCTION filebeat_collect_logs():
  
  // Background daemon
  FOR EACH docker_container:
    
    // Step 1: Subscribe to container logs
    log_stream = DOCKER_LOGS_STREAM(
      container_id = container.id,
      follow = true,
      stdout = true,
      stderr = true
    )
    
    // Step 2: Buffer logs
    log_buffer = []
    FOR each log_line in log_stream:
      
      log_event = {
        message: log_line,
        container: {
          id: container.id,
          name: container.name,
          image: container.image,
          labels: container.labels
        },
        timestamp: CURRENT_TIME_RFC3339()
      }
      
      log_buffer.append(log_event)
      
      // Step 3: Flush buffer periodically or on size
      IF CURRENT_TIME() - last_flush > 5s OR len(log_buffer) > 100:
        
        // Send to Logstash via Beats protocol
        FOR each event in log_buffer:
          beats_client.send(event)
        
        log_buffer.clear()
        last_flush = CURRENT_TIME()

ENDFOR
ENDFOR
ENDFUNCTION
```

**Backpressure Handling:** If Logstash is slow, Filebeat buffers up to 64KB; if buffer fills, drops oldest events

### 2. Logstash Processing

**Input:** Log events from Filebeat  
**Output:** Enriched logs sent to Elasticsearch

**Algorithm:**

```
FUNCTION logstash_process_logs():
  
  FOR each event in beats_input_stream:
    
    // Step 1: Initial enrichment
    event.service_name = event.container.name
    event.environment = 'production'
    
    // Step 2: Parse based on service
    IF event.service_name == 'j3-dms':
      
      // Try JSON parsing
      IF event.message starts_with '{':
        parsed = JSON_PARSE(event.message)
        IF parsed is not NULL:
          event.level = parsed.level
          event.requestId = parsed.requestId
          event.traceId = parsed.traceId
          event.message = parsed.message
          event.j3_parsed = parsed
    
    ELSEIF event.service_name == 'kong':
      
      // Grok-parse access log
      IF GROK_MATCH(event.message, kong_access_pattern):
        event.client_ip = grok_result.client_ip
        event.http_method = grok_result.method
        event.http_status = grok_result.status_code
        event.latency_ms = grok_result.latency
        event.request_path = grok_result.url
    
    // Step 3: Generic field extraction
    IF REGEX_MATCH(event.message, 'traceId[=:]([^ ]+)'):
      event.trace_id = regex_capture[1]
    
    IF REGEX_MATCH(event.message, 'requestId[=:]([^ ]+)'):
      event.request_id = regex_capture[1]
    
    // Step 4: Transform timestamp
    IF event.timestamp is valid:
      event.@timestamp = PARSE_RFC3339(event.timestamp)
    ELSE:
      event.@timestamp = CURRENT_TIME()
    
    // Step 5: Send to Elasticsearch
    es_index = FORMAT("drs-logs-%{YYYY.MM.dd}", event.@timestamp)
    elasticsearch_client.index(
      index = es_index,
      body = event
    )

ENDFOR
ENDFUNCTION
```

**Filter Performance:** ~1000 events/sec per Logstash instance

---

## Distributed Trace Correlation Algorithm

### 1. Trace ID Propagation

**Input:** HTTP request headers  
**Output:** Trace ID carries through all service calls

**Algorithm:**

```
FUNCTION propagate_trace_id(incoming_request):
  
  // Step 1: Extract or create trace ID
  IF incoming_request.headers['traceparent'] exists:
    // W3C standard: traceparent = version-trace_id-parent_id-traceflags
    traceparent = incoming_request.headers['traceparent']
    trace_id = EXTRACT_TRACE_ID(traceparent)
  ELSE:
    trace_id = UUID_V4()
  
  // Step 2: Create span
  current_span = {
    trace_id: trace_id,
    span_id: UUID_V4(),
    parent_span_id: EXTRACT_PARENT_ID(traceparent) or NULL,
    operation_name: incoming_request.method + ' ' + incoming_request.path,
    start_time: CURRENT_TIME_MICROS(),
    tags: {
      'http.method': incoming_request.method,
      'http.url': incoming_request.url,
      'span.kind': 'server'
    }
  }
  
  // Step 3: Propagate to child services
  FOR each outgoing_request to other_service:
    new_traceparent = FORMAT_TRACEPARENT(
      trace_id = trace_id,
      parent_span_id = current_span.span_id,
      traceflags = '01'  // sampled
    )
    outgoing_request.headers['traceparent'] = new_traceparent
    
    // Also pass in query/body for non-HTTP (Kafka, gRPC)
    outgoing_request.metadata['trace_id'] = trace_id
  
  // Step 4: Record span to Jaeger
  current_span.end_time = CURRENT_TIME_MICROS()
  current_span.duration_micros = current_span.end_time - current_span.start_time
  jaeger_agent.send_span(current_span)
  
  RETURN trace_id

ENDFUNCTION
```

### 2. Trace Sampling

**Input:** All traces generated  
**Output:** Sampled traces sent to Jaeger backend

**Algorithm:**

```
FUNCTION jaeger_trace_sampling(trace):
  
  // Step 1: Check if trace should be sampled
  // Strategy 1: Head-based sampling (decide at trace start)
  sampling_rate = 0.1  // Sample 10% of traces
  
  IF RANDOM() < sampling_rate:
    should_sample = true
  ELSE:
    should_sample = false
  
  // Step 2: Alternative - tail-based sampling
  // (Sample based on error/latency in trace)
  IF trace.has_errors() OR trace.duration_micros > 1000000:  // > 1s
    should_sample = true
  
  // Step 3: Adaptive sampling (reduce rate if backend is overloaded)
  IF jaeger_backend.cpu_usage > 80%:
    sampling_rate = 0.05  // Reduce to 5%
  
  // Step 4: Send sampled traces
  IF should_sample:
    FOR each span in trace.spans:
      jaeger_agent.send_span(span)
  
  RETURN should_sample

ENDFUNCTION
```

---

## Anomaly Detection Algorithm

### 1. Baseline Learning (Statistical)

**Input:** Historical metrics (e.g., 7 days)  
**Output:** Anomaly threshold bounds

**Algorithm:**

```
FUNCTION learn_anomaly_baseline(metric_name, labels, duration_days=7):
  
  // Step 1: Fetch historical data
  historical_data = prometheus_query(
    expr = metric_name,
    labels = labels,
    duration = 7 * 24 * 60 * 60 seconds
  )
  
  // Step 2: Compute statistical bounds
  mean = AVERAGE(historical_data.values)
  std_dev = STDEV(historical_data.values)
  
  // Standard deviation bands
  lower_bound = mean - (3 * std_dev)  // 3-sigma
  upper_bound = mean + (3 * std_dev)
  
  baseline = {
    metric: metric_name,
    labels: labels,
    mean: mean,
    std_dev: std_dev,
    lower_bound: lower_bound,
    upper_bound: upper_bound,
    last_trained: CURRENT_TIME()
  }
  
  anomaly_model.store(baseline)
  
  RETURN baseline

ENDFUNCTION

FUNCTION detect_anomaly_statistical(metric_name, labels, current_value):
  
  baseline = anomaly_model.get(metric_name, labels)
  
  IF baseline is NULL:
    // Model not trained yet
    RETURN { is_anomaly: false, reason: 'baseline_not_found' }
  
  IF CURRENT_TIME() - baseline.last_trained > 24 * 60 * 60:  // > 1 day old
    // Retrain baseline
    baseline = learn_anomaly_baseline(metric_name, labels)
  
  // Check if current value exceeds bounds
  IF current_value < baseline.lower_bound OR current_value > baseline.upper_bound:
    z_score = (current_value - baseline.mean) / baseline.std_dev
    
    RETURN {
      is_anomaly: true,
      z_score: z_score,
      deviation_percent: ((current_value - baseline.mean) / baseline.mean) * 100
    }
  ELSE:
    RETURN { is_anomaly: false }

ENDFUNCTION
```

**Use Case:** Detect abnormal CPU spike, memory leak, request latency increase

### 2. Rate of Change Detection

**Purpose:** Alert on rapid changes (e.g., error rate spike)

**Algorithm:**

```
FUNCTION detect_rate_change(metric_name, labels, threshold_percent=50):
  
  // Step 1: Get current and previous values
  current_value = prometheus_query(metric_name, labels, range_seconds=60)
  previous_value = prometheus_query(metric_name, labels, range_seconds=120)
  
  // Step 2: Calculate percent change
  IF previous_value == 0:
    percent_change = 100 * (current_value > 0 ? 1 : 0)
  ELSE:
    percent_change = ABS((current_value - previous_value) / previous_value * 100)
  
  // Step 3: Trigger anomaly
  IF percent_change > threshold_percent:
    RETURN {
      is_anomaly: true,
      percent_change: percent_change,
      severity: 'warning'
    }
  ELSE:
    RETURN { is_anomaly: false }

ENDFUNCTION
```

---

## Health Check & Recovery Algorithm

### 1. Service Health Assessment

**Input:** `up` metric, response time, error rate  
**Output:** Health status (healthy/degraded/down)

**Algorithm:**

```
FUNCTION assess_service_health(job_name):
  
  // Step 1: Check if service is up
  up_metric = prometheus_query('up{job="' + job_name + '"}')
  IF up_metric == 0:
    RETURN { status: 'DOWN', reason: 'not_responding' }
  
  // Step 2: Check response latency
  p95_latency = prometheus_query(
    'histogram_quantile(0.95, rate(request_duration_seconds_bucket[5m]))'
  )
  IF p95_latency > 2000:  // > 2 seconds
    RETURN { status: 'DEGRADED', reason: 'high_latency', p95_latency_ms: p95_latency }
  
  // Step 3: Check error rate
  error_rate = prometheus_query(
    'rate(request_errors_total[5m]) / rate(request_total[5m])'
  )
  IF error_rate > 0.05:  // > 5%
    RETURN { status: 'DEGRADED', reason: 'high_error_rate', error_rate: error_rate }
  
  // Step 4: All checks pass
  RETURN { status: 'HEALTHY' }

ENDFUNCTION
```

### 2. Automatic Recovery Attempts

**Input:** Down service alert  
**Output:** Recovery initiated, status tracked

**Algorithm:**

```
FUNCTION attempt_recovery(service_name):
  
  // Step 1: Log recovery attempt
  LOG("Attempting recovery for " + service_name)
  
  // Step 2: Try restart
  DOCKER_RESTART(service_name)
  
  // Step 3: Wait for recovery
  max_wait_seconds = 60
  wait_interval = 5
  attempts = 0
  
  FOR attempt = 1 TO (max_wait_seconds / wait_interval):
    SLEEP(wait_interval)
    
    health = assess_service_health(service_name)
    
    IF health.status == 'HEALTHY':
      LOG("Service " + service_name + " recovered after " + (attempt * wait_interval) + "s")
      CREATE_ALERT(
        type: 'recovery_success',
        service: service_name,
        recovery_time_seconds: attempt * wait_interval
      )
      RETURN { recovered: true, time_seconds: attempt * wait_interval }
    
    ELSEIF attempt == (max_wait_seconds / wait_interval):
      LOG("Service " + service_name + " failed to recover after " + max_wait_seconds + "s")
      CREATE_ALERT(
        type: 'recovery_failed',
        service: service_name
      )
      RETURN { recovered: false }

ENDFOR

ENDFUNCTION
```

---

## Data Retention & Compaction Algorithm

### 1. Metrics Retention Policy

**Input:** Time-series data age  
**Output:** Old data deleted, storage optimized

**Algorithm:**

```
FUNCTION prometheus_retention_cleanup():
  
  // Runs every 1 hour
  retention_seconds = 24 * 60 * 60  // 24 hours
  
  FOR each block_file in prometheus/tsdb/blocks:
    
    // Step 1: Check block age
    block_age_seconds = CURRENT_TIME() - block_file.creation_time
    
    // Step 2: Delete if older than retention
    IF block_age_seconds > retention_seconds:
      
      DELETE_BLOCK(block_file)
      freed_bytes = block_file.size
      
      LOG("Deleted block " + block_file.name + ", freed " + freed_bytes + " bytes")
    
    // Step 3: Compact blocks if needed
    ELSEIF block_age_seconds > 2 * retention_seconds / 3:
      
      // Compress older blocks more aggressively
      compressed_size = COMPRESS_BLOCK(block_file, level='max')
      saved_bytes = block_file.size - compressed_size
      
      LOG("Compressed block " + block_file.name + ", saved " + saved_bytes + " bytes")

ENDFOR

ENDFUNCTION
```

### 2. Log Retention & ILM (Index Lifecycle Management)

**Input:** Log indices age  
**Output:** Old indices deleted per policy

**Algorithm:**

```
FUNCTION elasticsearch_ilm_policy():
  
  // Check indices daily
  FOR each index matching "drs-logs-*":
    
    index_date = EXTRACT_DATE_FROM_INDEX_NAME(index)
    index_age_days = CURRENT_DATE() - index_date
    
    // ILM Policy phases:
    
    // Phase 1: Hot (0-1 days) - searchable, write optimized
    IF index_age_days <= 1:
      SET priority = 100
      SET refresh_interval = '1s'
    
    // Phase 2: Warm (1-7 days) - searchable, read optimized
    ELSEIF index_age_days <= 7:
      SET priority = 50
      SET refresh_interval = '30s'
      SHRINK_INDEX(num_shards = 1)  // Merge shards
    
    // Phase 3: Cold (7-30 days) - rarely searched, best compression
    ELSEIF index_age_days <= 30:
      SET priority = 0
      FORCE_MERGE(index, num_segments = 1)  // Single segment for compression
      SET allocation.require.node_type = 'cold'  // Move to cheaper storage
    
    // Phase 4: Delete (30+ days)
    ELSEIF index_age_days > 30:
      DELETE_INDEX(index)

ENDFOR

ENDFUNCTION
```

**Storage Efficiency:** ILM reduces storage by 90% vs. unmanaged indices (via merging + tiering)

---

## Summary: Key Algorithmic Principles

| Algorithm | Key Principle | Time Complexity |
|-----------|---------------|-----------------|
| **Scrape Cycle** | Parallel HTTP pulls every 15s | O(n * m) |
| **Alert Evaluation** | Deterministic PromQL queries on fixed schedule | O(a * t) |
| **Alert Routing** | Hierarchical routing + deduplication | O(g * r) |
| **Log Processing** | Streaming parse with buffer flushing | O(e) per event |
| **Trace Correlation** | Head-based sampling + W3C propagation | O(1) per request |
| **Anomaly Detection** | Statistical bounds + rate-of-change | O(h) for training |
| **Health Assessment** | Multi-signal decision tree | O(3) checks |
| **Data Retention** | Time-based cleanup + ILM phases | O(i) indices |

---

**Next Steps:**
- Implement custom rules in [Alert Configuration](./MONITORING_OBSERVABILITY_SETUP.md#alerting-rules)
- Tune sampling rates for [High-Scale Deployments](./MONITORING_OBSERVABILITY_ADMIN_MANUAL.md#scaling)
- Test anomaly detection (see [Testing Guide](./MONITORING_OBSERVABILITY_TESTING.md))
