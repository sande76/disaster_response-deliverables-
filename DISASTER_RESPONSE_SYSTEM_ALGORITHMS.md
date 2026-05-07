# Disaster Response System Algorithms

**Version:** 1.0.0  
**Date:** May 8, 2026  
**Scope:** Algorithmic behavior for J1, J2, J3, J4, and the entire integrated Disaster Response System

---

## Table of Contents

1. [Purpose](#purpose)
2. [Algorithm Design Principles](#algorithm-design-principles)
3. [J1 Algorithm — Device & Edge Systems](#j1-algorithm--device--edge-systems)
4. [J2 Algorithm — Data & Intelligence](#j2-algorithm--data--intelligence)
5. [J3 Algorithm — System Engineering & Interaction](#j3-algorithm--system-engineering--interaction)
6. [J4 Algorithm — Platform Security & Integration](#j4-algorithm--platform-security--integration)
7. [Whole System Algorithm](#whole-system-algorithm)
8. [Complexity & Reliability Summary](#complexity--reliability-summary)

---

## Purpose

This document describes how the Disaster Response System behaves as a set of coordinated algorithms rather than as isolated services. The goal is to show:

- how each subsystem makes decisions,
- what data each subsystem consumes and produces,
- how information moves through the platform,
- how failures are handled without stopping the whole system,
- and how the full end-to-end emergency response workflow completes.

The system is organized into four major algorithmic domains:

- **J1** detects and reports field events from edge devices.
- **J2** analyzes the incoming data and generates predictions or risk scores.
- **J3** presents incidents to operators and synchronizes the live command center.
- **J4** protects, routes, observes, and audits the platform.

---

## Algorithm Design Principles

The platform uses the following algorithmic principles throughout:

1. **Event-first design**
   - Every important action is represented as an event.
   - Events are published to Kafka, persisted in PostgreSQL, and surfaced in the dashboard.

2. **Edge-to-core escalation**
   - Detection begins at the edge when possible.
   - The core platform receives only actionable or synchronized state changes.

3. **Idempotent processing**
   - Repeated device reports, retries, and webhook deliveries must not create duplicate incidents.

4. **Eventually consistent UI**
   - The dashboard is updated immediately through Socket.IO, but database persistence remains the source of truth.

5. **Observability as a control loop**
   - Metrics, logs, traces, and alerts are part of the platform algorithm, not an afterthought.

6. **Security at every boundary**
   - Requests are authenticated, authorized, rate-limited, logged, and auditable.

---

## J1 Algorithm — Device & Edge Systems

### Objective

J1 detects physical danger signals at the edge, evaluates them locally, and forwards only relevant reports or emergency signals to the rest of the system.

### Inputs

- Sensor readings from flood, landslide, and weather devices
- Local thresholds configured for each device type
- Connectivity status to gateway or network
- Battery and health state of the node

### Outputs

- Regular telemetry packets
- SOS or emergency reports
- Local buffer entries when offline
- Retry queue items when delivery fails

### Algorithm Description

J1 follows a **sense → validate → classify → buffer → transmit** loop.

#### Step-by-step behavior

1. **Sense data**
   - Each device polls its sensors at a fixed interval.
   - Readings are timestamped immediately to preserve ordering.

2. **Validate data**
   - The device checks whether readings are within physically possible ranges.
   - Obvious sensor corruption is rejected or flagged.

3. **Classify risk locally**
   - The node compares readings against thresholds.
   - If the reading is normal, the device sends telemetry only.
   - If the reading crosses a warning or critical threshold, the node creates an event with severity.

4. **Create an edge event**
   - The node packages the event with device ID, location, readings, and severity.
   - A checksum or request signature can be attached for integrity.

5. **Transmit or buffer**
   - If network connectivity exists, the report is sent to the gateway.
   - If the network is unavailable, the event is placed in a local retry buffer.

6. **Retry delivery**
   - Buffered events are retried in FIFO order.
   - Duplicate prevention is applied by event ID or timestamp hash.

7. **Escalate emergency signals**
   - Critical reports are sent with higher priority.
   - These events are intended to trigger downstream incident creation quickly.

### J1 Pseudocode

```text
FOR each device node DO
    read sensor values
    validate sensor values

    IF values are invalid THEN
        flag sensor fault
        continue
    END IF

    severity = evaluate_thresholds(values)

    event = build_event(device_id, location, values, severity, timestamp)

    IF network_available() THEN
        transmit(event)
    ELSE
        buffer_locally(event)
    END IF

    WHILE buffered_events_exist() AND network_available() DO
        retry_oldest_buffered_event()
    END WHILE
END FOR
```

### Failure Handling

- If a sensor fails, the device reports a health fault rather than a normal reading.
- If the connection is lost, events stay in local storage until delivery resumes.
- If duplicate transmission occurs, the downstream services must deduplicate by event identity.

### Complexity

- Threshold evaluation per reading: **O(1)**
- Buffer retry: **O(k)** for k buffered events
- Per node cycle: **O(s)** where s is the number of sensor readings processed in one interval

---

## J2 Algorithm — Data & Intelligence

### Objective

J2 converts incoming operational and environmental data into predictions, risk scores, and analytical outputs that support early response.

### Inputs

- Kafka device reports from J1
- Incident history from PostgreSQL
- Historical datasets and model training artefacts
- Geographic and environmental feature data

### Outputs

- Risk scores per region or hazard type
- Prediction events published to Kafka
- Persisted prediction records in PostgreSQL
- Model performance metrics

### Algorithm Description

J2 follows an **ingest → clean → enrich → feature-engineer → infer → publish** workflow.

#### Step-by-step behavior

1. **Ingest new data**
   - J2 reads live events from Kafka and optionally batches from the database.
   - It uses the newest available state for each region or device group.

2. **Clean the data**
   - Missing values are imputed or rejected according to the feature type.
   - Duplicate or stale records are dropped.
   - Outliers are capped or marked for special handling.

3. **Enrich the data**
   - Sensor values are combined with location, history, and trend context.
   - Spatial and temporal context is added.

4. **Build features**
   - Time-windowed statistics are calculated.
   - For example: moving average rainfall, rate of rise, soil saturation trend.
   - Features are normalized into model-ready form.

5. **Run inference**
   - A prediction model is selected based on hazard type.
   - The model returns a risk score and confidence value.

6. **Apply business rules**
   - If the risk score is below threshold, store and return it.
   - If the risk score is above threshold, generate a high-priority prediction event.

7. **Persist and publish**
   - The result is stored in PostgreSQL for traceability.
   - The result is published to Kafka for J3 and other subscribers.

### J2 Pseudocode

```text
FOR each incoming event from Kafka OR batch source DO
    cleaned = clean_record(event)
    enriched = enrich_with_history_and_geo(cleaned)
    features = build_features(enriched)

    model = select_model(features.hazard_type)
    prediction = model.predict(features)

    score = normalize_prediction(prediction)
    confidence = calculate_confidence(prediction)

    store_prediction(score, confidence, metadata)
    publish_prediction_event(score, confidence, metadata)
END FOR
```

### Failure Handling

- If a feature is missing, the system falls back to defaults or drops the record.
- If a model is unavailable, the system returns a degraded prediction based on a fallback rule.
- If Kafka delivery fails, the result is queued and retried.

### Complexity

- Cleaning and enrichment: **O(n)** for n records
- Feature generation: **O(n × f)** for f features
- Inference per record: depends on model, typically **O(f)** to **O(f log f)** for tree models
- End-to-end batch processing: **O(n × f)**

---

## J3 Algorithm — System Engineering & Interaction

### Objective

J3 turns platform events into a live operational interface, maintains incident state, and synchronizes operators, alerts, and dashboards in real time.

### Inputs

- Incident requests from J1 and external users
- Prediction events from J2
- Alert webhooks from J4/Alertmanager
- Kafka event streams
- Database records from PostgreSQL

### Outputs

- Real-time dashboard updates
- Incident state transitions
- Notification broadcasts
- User-facing API responses

### Algorithm Description

J3 follows a **receive → normalize → persist → broadcast → reconcile** pattern.

#### Step-by-step behavior

1. **Receive an event or request**
   - An incident may enter through a REST call, a device report, or a Kafka message.

2. **Normalize the payload**
   - Payload fields are mapped to the internal incident schema.
   - Required fields are validated.
   - Missing or malformed fields trigger rejection or defaulting.

3. **Persist the canonical state**
   - The incident, alert, or prediction is stored in PostgreSQL.
   - The database acts as the source of truth for the UI.

4. **Broadcast live updates**
   - Socket.IO pushes the change to connected dashboards.
   - All active clients receive the updated state with minimal delay.

5. **Reconcile asynchronous updates**
   - Kafka messages, webhook callbacks, and manual edits may arrive in different orders.
   - J3 resolves these by comparing timestamps, version numbers, or state precedence.

6. **Update user interface**
   - Panels, maps, tables, and alert widgets are refreshed.
   - The UI reflects the newest persisted incident state.

### J3 Pseudocode

```text
ON event_or_request_received(payload) DO
    valid = validate_schema(payload)
    IF NOT valid THEN
        return error_response()
    END IF

    normalized = map_to_internal_model(payload)
    current_state = load_current_state(normalized.id)
    merged_state = reconcile(current_state, normalized)

    save_to_database(merged_state)
    emit_socket_event(merged_state)
    return success_response(merged_state)
END ON
```

### Failure Handling

- If the socket layer is unavailable, the database still persists the state.
- If Kafka is delayed, the dashboard remains consistent after the next reconciliation cycle.
- If a user updates an already modified incident, last-write rules or version checks prevent silent overwrites.

### Complexity

- Validation and normalization: **O(f)** for f payload fields
- Reconciliation: **O(1)** to **O(log n)** depending on lookup strategy
- Broadcast to c clients: **O(c)**
- Database write: amortized **O(1)** per record, excluding indexing costs

---

## J4 Algorithm — Platform Security & Integration

### Objective

J4 secures the system boundary, routes requests, monitors the platform, and keeps audit evidence for operational trust.

### Inputs

- Incoming client requests
- Authentication tokens
- Service metrics and logs
- Trace spans
- Audit events and system health signals

### Outputs

- Routed requests to downstream services
- Authentication/authorization decisions
- Metrics, alerts, dashboards, and logs
- Immutable or durable audit records

### Algorithm Description

J4 is not one single procedure. It is a set of coordinated algorithms that run together:

1. **Gateway request algorithm**
2. **Identity and access algorithm**
3. **Monitoring and alerting algorithm**
4. **Logging pipeline algorithm**
5. **Trace correlation algorithm**
6. **Audit algorithm**

### J4.1 Gateway Request Algorithm

1. Accept request at Kong.
2. Verify request method, route, and rate limit.
3. Validate token or client credential.
4. Resolve the upstream service.
5. Forward the request to J1, J2, J3, or J4 audit API.
6. Log the request metadata.

```text
ON request_received DO
    if rate_limit_exceeded(request.client) then
        reject 429
    end if

    if not authenticate(request) then
        reject 401
    end if

    if not authorize(request, route) then
        reject 403
    end if

    upstream = resolve_route(request.path)
    forward(request, upstream)
    log_request(request)
END
```

### J4.2 Identity and Access Algorithm

1. User opens the login flow.
2. Keycloak authenticates credentials.
3. Keycloak issues a signed token.
4. Kong validates the token on each request.
5. Role and scope checks decide access.

### J4.3 Monitoring and Alerting Algorithm

1. Prometheus scrapes service metrics on schedule.
2. Rules are evaluated at the configured interval.
3. Alerts are grouped, deduplicated, and routed.
4. Notifications are sent via webhook, email, or other receivers.
5. Grafana visualizes the same signal for human operators.

### J4.4 Logging Pipeline Algorithm

1. Filebeat collects container logs.
2. Logstash parses, enriches, and transforms events.
3. Elasticsearch indexes the normalized records.
4. Kibana queries and visualizes them.
5. ILM policies move old data through lifecycle tiers.

### J4.5 Trace Correlation Algorithm

1. An incoming request receives or propagates a trace context.
2. Each downstream span inherits the trace ID.
3. Spans are exported to Jaeger.
4. Operators use traces to identify latency bottlenecks.

### J4.6 Audit Algorithm

1. Significant actions are recorded.
2. The audit record is stored in durable storage.
3. If blockchain audit is enabled, the hash is written immutably.
4. Audit records are later used for verification and compliance.

### Complexity

- Gateway auth/routing: **O(1)** per request, excluding upstream latency
- Metrics scraping: **O(t)** for t targets per interval
- Alert routing: **O(a)** for a active alerts
- Log indexing: **O(l)** for l log events
- Trace propagation: **O(h)** for h request hops

---

## Whole System Algorithm

### Objective

The entire platform converts a field signal into an operational response, while monitoring itself continuously to ensure the response path is healthy.

### End-to-end lifecycle

The system algorithm can be described as a closed loop with six phases:

1. **Sense**
   - J1 detects a hazardous condition at the edge.

2. **Transmit**
   - J1 sends the event to the platform through Kong or Kafka.

3. **Analyze**
   - J2 enriches the data and predicts the likely risk trend.

4. **Coordinate**
   - J3 persists the state, updates the dashboard, and notifies operators.

5. **Protect**
   - J4 validates, routes, logs, traces, and alerts across the system.

6. **Learn and recover**
   - Historical incidents, logs, metrics, and traces are retained for review, tuning, and recovery.

### Whole System Pseudocode

```text
LOOP forever
    j1_event = collect_edge_signal()

    IF j1_event exists THEN
        send_to_gateway(j1_event)
        enqueue_for_streaming(j1_event)
    END IF

    j2_output = analyze_incoming_data()

    IF j2_output risk is high THEN
        create_or_update_incident(j2_output)
        broadcast_to_dashboard(j2_output)
    END IF

    monitor_system_health()
    route_alerts_if_needed()
    collect_logs_and_traces()
    preserve_audit_history()
END LOOP
```

### End-to-End Detailed Flow

#### Phase 1: Detection at J1
- Sensors notice water rise, slope movement, or abnormal weather.
- J1 applies local threshold checks and creates a structured event.

#### Phase 2: Admission through J4
- The event passes through Kong.
- Kong authenticates, authorizes, and rate-limits the request.

#### Phase 3: Persistence and Ingestion
- J3 stores the incident in PostgreSQL.
- The event is streamed to Kafka.
- J2 consumes the stream for analysis.

#### Phase 4: Prediction and Decision Support
- J2 computes a risk score or hazard class.
- If the score crosses a response threshold, the result is published.

#### Phase 5: Operational Coordination
- J3 updates the live dashboard and incident queue.
- Socket.IO pushes the change to all connected users.
- Operators receive the latest actionable state.

#### Phase 6: Observability and Control
- Prometheus scrapes all services.
- Alertmanager routes anomalies.
- Filebeat and Logstash centralize logs.
- Jaeger collects traces.
- Grafana, Kibana, and Jaeger expose the operational truth to humans.

### System-Level Failure Handling

The platform is designed so that one failure does not stop the whole response loop.

- If J1 is offline, the system continues working with existing events and alerts the operator.
- If J2 is unavailable, incident intake still works and predictions resume when J2 recovers.
- If J3 fails, data is still collected and can be replayed from Kafka or the database.
- If J4 monitoring fails, meta-alerts are generated from health checks and logs.

### System-Level Complexity

The overall system is a distributed pipeline, so the complexity is determined by the slowest active stage in the current path.

- Edge sensing: **O(1)** per device reading
- Gateway routing: **O(1)** per request
- Analytics: **O(n × f)** per batch
- Dashboard updates: **O(c)** for connected clients
- Observability: **O(t + a + l + h)** per cycle across targets, alerts, logs, and trace hops

---

## Complexity & Reliability Summary

| Section | Core Algorithm | Typical Complexity | Main Reliability Mechanism |
|---------|----------------|-------------------|----------------------------|
| **J1** | Sense, classify, buffer, transmit | O(s) per cycle | Local buffering and retry |
| **J2** | Ingest, enrich, infer, publish | O(n × f) | Kafka retry + fallback logic |
| **J3** | Validate, persist, broadcast, reconcile | O(f + c) | Database source of truth |
| **J4** | Auth, route, monitor, log, trace | O(1) to O(t + a + l) | Redundant observability path |
| **Whole system** | Event-driven emergency response loop | Distributed pipeline | Idempotency, retries, observability |

---

## Summary

The Disaster Response System works as a coordinated set of algorithms rather than a single application:

- **J1** detects danger as close to the field as possible.
- **J2** transforms raw signals into risk intelligence.
- **J3** turns those signals into an operator-ready incident workflow.
- **J4** secures, routes, monitors, logs, traces, and audits the entire platform.
- **The whole system** forms an event-driven feedback loop that supports disaster response from detection to recovery.

If you want, I can also turn this into a more formal README-style chapter and link it from the main project README.
