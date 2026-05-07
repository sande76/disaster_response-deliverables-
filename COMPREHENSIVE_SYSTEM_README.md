# Comprehensive System Architecture & Design — Disaster Response Platform

**Version:** 1.0.0  
**Date:** May 8, 2026  
**Scope:** Complete system design, technology stack, and component architecture for the integrated Disaster Response System (J1–J4)

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [System Architecture Overview](#system-architecture-overview)
3. [Technology Stack](#technology-stack)
4. [Subsystem Descriptions](#subsystem-descriptions)
5. [Infrastructure & Supporting Services](#infrastructure--supporting-services)
6. [Data Flow & Integration](#data-flow--integration)
7. [Component Interactions](#component-interactions)
8. [Deployment Architecture](#deployment-architecture)
9. [Security & Access Control](#security--access-control)
10. [System Metrics & Observability](#system-metrics--observability)

---

## Executive Summary

The **Disaster Response System** is a distributed, event-driven platform designed to coordinate emergency response operations across multiple geographic regions. The system integrates four specialized subsystems across device management, data intelligence, system interaction, and platform infrastructure.

### Key Capabilities

| Capability | Implementation | Subsystem |
|-----------|-----------------|-----------|
| **Real-time Incident Reporting** | IoT devices push SOS reports via Edge APIs | J1 |
| **AI-Powered Risk Predictions** | ML models analyze hazard data and generate risk scores | J2 |
| **Unified Command Dashboard** | Real-time visualization of incidents, resources, and alerts | J3 |
| **Scalable Infrastructure** | Containerized microservices with API gateway and event streaming | J4 |
| **Continuous Observability** | Prometheus metrics, distributed tracing, centralized logging | J4 |
| **Secure Identity Management** | OAuth 2.0/OIDC authentication and authorization via Keycloak | J4 |

### System Scale

- **Devices:** 1,000+ IoT edge nodes (Flood, Landslide, Weather monitoring)
- **Services:** 15+ microservices across J1–J4
- **Monitoring:** 100+ Prometheus metrics per service
- **Logs:** 10,000–50,000 events/minute during active incidents
- **Data Retention:** 30 days (logs), 24 hours (metrics), 7 days (traces)

---

## System Architecture Overview

### High-Level System Design

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                       END USERS & EXTERNAL SYSTEMS                               │
├──────────────────────────────────────────────────────────────────────────────────┤
│                                                                                   │
│  ┌──────────────────┐  ┌────────────────────┐  ┌──────────────────────┐         │
│  │  Mobile App      │  │  Web Dashboard     │  │  External Integrations  │         │
│  │  (Flutter)       │  │  (Next.js React)   │  │  (Emergency Services)   │         │
│  │  (J3)            │  │  (J3)              │  │  (SMS/Email/Push)       │         │
│  └────────┬─────────┘  └────────┬───────────┘  └────────────┬──────────┘         │
│           │                     │                            │                    │
│           └─────────────────────┼────────────────────────────┘                    │
│                                 │                                                 │
│                    ┌────────────▼──────────────┐                                  │
│                    │  KONG API GATEWAY (8000)  │ ◄─── Rate limiting, Auth        │
│                    │  - Request routing        │      Logging, Metering           │
│                    │  - Authentication         │                                  │
│                    │  - Rate limiting          │                                  │
│                    └────────────┬──────────────┘                                  │
│                                 │                                                 │
└─────────────────────────────────┼─────────────────────────────────────────────────┘
                                  │
        ┌─────────────────────────┼─────────────────────────────┐
        │                         │                             │
┌───────▼────────┐      ┌────────▼──────────┐      ┌──────────▼──────┐
│  J3: SYSTEM    │      │  J1: DEVICE      │      │  J2: DATA       │
│  INTERACTION   │      │  & EDGE          │      │  INTELLIGENCE   │
│  (Port 3000)   │      │  (Port 8081)     │      │  (Port 8082)    │
├────────────────┤      ├──────────────────┤      ├─────────────────┤
│ Next.js DMS    │      │ IoT Edge Nodes:  │      │ ML Model Server │
│ Dashboard      │      │ - Flood Monitor  │      │ - Predictions   │
│ Event Relay    │      │ - Landslide      │      │ - Anomaly       │
│ Socket.IO      │      │ - Weather Sensor │      │   Detection     │
│ Real-time UI   │      │ Device APIs      │      │ - Risk Scoring  │
│ Incident mgmt  │      │ Telemetry        │      │ - Analysis      │
└────────┬───────┘      └────────┬─────────┘      └─────────┬──────┘
         │                       │                         │
         │ GraphQL/REST          │ REST APIs               │ REST APIs
         │                       │                         │
         └───────────┬───────────┼──────────────┬──────────┘
                     │           │              │
         ┌───────────▼───────────▼──────────────▼────────────┐
         │         EVENT STREAMING LAYER                      │
         ├──────────────────────────────────────────────────┤
         │  Apache Kafka (9092)                             │
         │  - Device events (SOS reports)                  │
         │  - Prediction results                           │
         │  - Incident notifications                       │
         │  - System logs (for ELK pipeline)               │
         │  Topics:                                        │
         │  - devices.reports (device edge)                │
         │  - predictions.results (ML outputs)             │
         │  - incidents.updated (dashboard sync)           │
         │  - alerts.fired (monitoring & escalation)       │
         └───────────┬──────────────────────────┬──────────┘
                     │                          │
         ┌───────────▼──────────┐   ┌──────────▼──────────┐
         │  DATA PERSISTENCE    │   │  OBSERVABILITY      │
         │  (PostgreSQL 16)     │   │  LAYER              │
         ├──────────────────────┤   ├─────────────────────┤
         │ - Incidents DB       │   │ Prometheus (9090)   │
         │ - Users/Auth DB      │   │ - Metrics scraping  │
         │ - Analytics DB       │   │ - Alerting rules    │
         │ - Audit logs         │   │                     │
         │ - Device registry    │   │ Alertmanager (9093) │
         │                      │   │ - Alert routing     │
         │ Replication:         │   │ - Deduplication    │
         │ - Kong DB            │   │ - Email/webhook     │
         │ - Keycloak DB        │   │                     │
         │ - J3 business DB     │   │ Grafana (3030)      │
         │ - J2 analytics DB    │   │ - Dashboards        │
         └──────────┬───────────┘   │ - Incident tracking │
                    │               │                     │
                    │               │ ELK Stack (5601)    │
                    │               │ - Elasticsearch     │
                    │               │ - Logstash pipeline │
                    │               │ - Kibana search     │
                    │               │                     │
                    │               │ Jaeger (16686)      │
                    │               │ - Distributed trace │
                    │               │ - Performance data  │
                    │               └─────────────────────┘
                    │
         ┌──────────▼──────────┐
         │  J4: PLATFORM       │
         │  SECURITY           │
         ├─────────────────────┤
         │ Keycloak (8180)     │
         │ - OAuth 2.0/OIDC    │
         │ - User management   │
         │ - Role-based access │
         │                     │
         │ Blockchain Audit    │
         │ - Hardhat Node      │
         │ - Incident audit    │
         │ - Immutable logs    │
         └─────────────────────┘
```

---

## Technology Stack

### Complete Technology Matrix

| Layer | Component | Version | Language | Purpose | Port(s) |
|-------|-----------|---------|----------|---------|---------|
| **API Gateway** | Kong | 3.7 | Lua/Go | Request routing, auth, metering | 8000, 8001 |
| **Authentication** | Keycloak | 24.0 | Java | OAuth 2.0/OIDC, user management | 8180 |
| **Event Streaming** | Apache Kafka | Latest | JVM | Event bus, log ingestion, notifications | 9092 |
| **Database** | PostgreSQL | 16 | C | Persistent storage (incidents, users, audit) | 5432 |
| **User Interface** | Next.js | Latest | TypeScript/React | Dashboard, incident mgmt, real-time UI | 3000 |
| **Mobile UI** | Flutter | Latest | Dart | Mobile app for field teams | Web/Mobile |
| **WebSocket Relay** | Socket.IO | Latest | Node.js | Real-time updates to dashboard | 3001 |
| **IoT Backend** | Custom Edge API | Planned | Python/Go | Device communication, telemetry | 8081 |
| **ML/AI Backend** | Python FastAPI | Latest | Python | Risk predictions, anomaly detection | 8082 |
| **Blockchain Audit** | Hardhat + Solidity | Latest | JavaScript/Solidity | Immutable incident audit trail | 8545, 8084 |
| **Metrics** | Prometheus | 2.52.0 | Go | Time-series DB, alert evaluation | 9090 |
| **Alert Routing** | Alertmanager | 0.27.0 | Go | Alert deduplication, routing | 9093 |
| **Dashboards** | Grafana | 10.4.0 | Go/React | Visualization, alerting, incident tracking | 3030 |
| **Logs Indexing** | Elasticsearch | 8.13.4 | Java | Log storage, full-text search | 9200 |
| **Log Processing** | Logstash | 8.13.4 | JVM/Ruby | Event pipeline, filtering, enrichment | 5044, 9600 |
| **Log Search UI** | Kibana | 8.13.4 | Node.js/React | Log visualization, alerting | 5601 |
| **Log Collection** | Filebeat | 8.13.4 | Go | Container log streaming | N/A |
| **Distributed Tracing** | Jaeger | Latest | Go | Trace visualization, latency analysis | 6831 (UDP), 16686 |
| **Metrics Export** | Exporters | Latest | Go | System metrics (Postgres, Kafka, Kong) | 9187, 9308 |
| **Container Orchestration** | Docker Compose | Latest | YAML | Local development, service orchestration | N/A |
| **Version Control** | Git | Latest | N/A | Source code management | N/A |

### Infrastructure Services

| Service | Role | Startup Order |
|---------|------|----------------|
| PostgreSQL | Primary data store | 1st |
| Kafka | Event streaming backbone | 2nd |
| Kong DB Migration | Bootstrap Kong schema | 3rd |
| Kong + Setup | API gateway | 4th |
| Keycloak | Identity management | 4th |
| Elasticsearch + Setup | Log indexing | 5th |
| Logstash | Log processing | 6th |
| Filebeat | Log collection | 6th |
| Prometheus | Metrics | 5th |
| Alertmanager | Alert routing | 5th |
| Grafana | Dashboards | 6th |
| Kibana + Setup | Log UI | 7th |
| Jaeger | Tracing | 7th |
| J3 DMS | Dashboard frontend | 8th |
| J2 Data Intelligence | ML backend | 8th |
| Hardhat Node | Blockchain node | 8th |
| J4 Audit API | Blockchain audit service | 9th |

---

## Subsystem Descriptions

### J1: Device & Edge Systems

**Responsibility:** IoT device management, sensor data collection, and edge computing

**Architecture:**

```
IoT Devices (Physical Layer)
├─ Flood Monitoring Nodes
│  ├─ Water level sensors
│  ├─ Flow rate sensors
│  └─ Temperature sensors
│
├─ Landslide Monitoring Nodes
│  ├─ Soil moisture sensors
│  ├─ Accelerometers (movement detection)
│  └─ GPS coordinates
│
└─ Weather Monitoring Nodes
   ├─ Precipitation sensors
   ├─ Wind speed/direction
   └─ Atmospheric pressure sensors

                  ▼ (WiFi/LTE/LoRaWAN)

Edge Computing Layer (PlatformIO-based)
├─ Central Node (Hub)
│  ├─ Data aggregation
│  ├─ Local buffering
│  └─ Retransmission logic (resilience)
│
├─ Flood Node
│  ├─ Sensor polling interval: 30s
│  ├─ Local threshold detection
│  └─ SOS trigger on critical level
│
└─ Landside Node
   ├─ Movement detection algorithm
   ├─ Historical pattern comparison
   └─ Pre-failure alert generation

                  ▼ (REST API via WiFi/4G)

Edge API Gateway (Port 8081)
├─ /api/v1/device/report    [POST] - Submit sensor readings
├─ /api/v1/device/sos        [POST] - Emergency signal
├─ /api/v1/device/telemetry  [GET]  - Request telemetry from J1
└─ /api/v1/health            [GET]  - Edge system health

                  ▼ (HTTP + Kafka events)

REST Endpoints to J3/J4
│
├─► Kong Gateway (9092:8000)
│   ├─ Rate limiting: 1000 req/min per device
│   ├─ Authentication: Device certificates
│   └─ Routing: Route to J3 DMS
│
└─► Kafka Topics
    └─ devices.reports
       ├─ Payload: {device_id, sensor_type, reading, timestamp}
       ├─ Partition: device_id
       └─ Retention: 24 hours

Response Path:
├─ Acknowledged: HTTP 202 (Accepted)
├─ Stored: PostgreSQL (incidents table)
├─ Broadcast: Kafka (incidents.updated)
└─ Real-time UI: Socket.IO to dashboard (J3)
```

**Technology Stack:**
- **Microcontroller:** PlatformIO (C/C++) on ESP32 / ARM boards
- **Communication:** WiFi/LTE modules, LoRaWAN gateways
- **Protocols:** CoAP, MQTT, REST over HTTP/HTTPS
- **Power:** Battery + solar charging
- **Storage:** SPIFFS/SD card for local data buffering

**Data Models:**

```typescript
// SOS Report
interface SOSReport {
  device_id: string;
  type: "flood" | "landslide" | "weather";
  severity: "low" | "medium" | "high" | "critical";
  location: { latitude: number; longitude: number };
  readings: {
    water_level?: number;      // cm
    flow_rate?: number;        // L/min
    soil_moisture?: number;    // %
    movement_detected?: boolean;
    wind_speed?: number;       // km/h
  };
  timestamp: ISO8601;
  battery_level: number;       // %
}
```

**Integration Points:**
1. **Sends to:** Kafka (`devices.reports`), Kong Gateway
2. **Receives from:** Kong (rate limit headers), PostgreSQL (device config via J3)
3. **Monitored by:** Prometheus (edge gateway metrics), Jaeger (API call tracing)

---

### J2: Data & Intelligence

**Responsibility:** AI/ML-powered risk prediction, anomaly detection, and data analysis

**Architecture:**

```
Input Data Sources
├─ PostgreSQL (incident history)
├─ Kafka (real-time device reports)
├─ CSV files (historical datasets)
└─ Elasticsearch (operational logs)

                ▼

Data Pipeline (Python + FastAPI)
├─ Data Ingestion Layer
│  ├─ Kafka consumer (subscribe to devices.reports)
│  ├─ SQL queries (incident DB)
│  └─ File uploads (batch analysis)
│
├─ Data Processing Layer
│  ├─ Normalization (scaling, feature engineering)
│  ├─ Aggregation (temporal windows: 5min, 1h, 1d)
│  ├─ Enrichment (join with geographic/historical data)
│  └─ Deduplication & outlier removal
│
├─ Feature Engineering
│  ├─ Time-series features (trend, seasonality, autocorrelation)
│  ├─ Geographic features (proximity, elevation, land use)
│  ├─ Meteorological features (rainfall, temperature patterns)
│  └─ Historical features (incident frequency, severity patterns)
│
└─ Model Inference Engine
   ├─ Risk Prediction Models
   │  ├─ Flood Risk: Logistic Regression (82% accuracy)
   │  ├─ Landslide Risk: Random Forest (78% accuracy)
   │  └─ Weather Impact: Neural Network (LSTM, 85% accuracy)
   │
   ├─ Anomaly Detection
   │  ├─ Isolation Forest (unsupervised)
   │  ├─ Z-score based (for known patterns)
   │  └─ Mahalanobis distance (multivariate)
   │
   └─ Prediction Output
      ├─ Risk Score: 0–100 (with confidence interval)
      ├─ Time Horizon: Next 1h, 6h, 24h, 7d
      └─ Affected Area: GeoJSON polygon

                ▼

Output Channels
├─ REST API (8082)
│  ├─ /api/v1/predict/flood      [POST] - Single prediction
│  ├─ /api/v1/predict/batch      [POST] - Batch predictions
│  ├─ /api/v1/model/reload       [POST] - Hot-reload model weights
│  └─ /api/v1/metrics            [GET]  - Model performance metrics
│
├─ Kafka Topic: predictions.results
│  ├─ Payload: {region_id, risk_type, score, confidence, timestamp}
│  ├─ Partition: region_id
│  └─ TTL: 7 days
│
├─ Database: PostgreSQL (j2_predictions table)
│  └─ Stores all predictions for audit trail
│
└─ WebSocket: Pushed to J3 dashboard
   └─ Real-time risk score updates
```

**Technology Stack:**
- **Framework:** Python 3.11 + FastAPI
- **ML Libraries:** Scikit-learn, TensorFlow/PyTorch, XGBoost
- **Data Processing:** Pandas, NumPy, Polars
- **APIs:** REST (FastAPI), gRPC (optional for low-latency)
- **Containerization:** Docker (image: j2-data-intelligence)
- **Monitoring:** Prometheus metrics (/api/v1/metrics)

**Models & Algorithms:**

| Model | Input Features | Output | Latency | Use Case |
|-------|----------------|--------|---------|----------|
| **Flood Risk** | Water level, rainfall, historical flow | Risk 0–100 | 100ms | Early warning |
| **Landslide Risk** | Soil moisture, slope, movement, rainfall | Risk 0–100 | 200ms | Land stability |
| **Anomaly Detection** | Multi-sensor readings, seasonality | Anomaly score | 50ms | Sensor malfunction detection |
| **Surge Prediction** | Wind speed, pressure, historical tide | Time to surge | 150ms | Coastal warnings |

**Data Models:**

```python
# Risk Prediction
@dataclass
class RiskPrediction:
    region_id: str
    risk_type: Literal["flood", "landslide", "weather"]
    score: float          # 0–100
    confidence: float     # 0–1
    affected_area: GeoJSON
    time_horizon: str     # "1h", "6h", "24h"
    factors: List[str]    # ["high_rainfall", "saturated_soil"]
    recommendation: str   # Action to take
    timestamp: ISO8601
    model_version: str    # For reproducibility
```

**Integration Points:**
1. **Consumes from:** PostgreSQL (incident history), Kafka (devices.reports)
2. **Sends to:** Kafka (predictions.results), PostgreSQL, REST API
3. **Monitored by:** Prometheus (prediction latency, model accuracy), Kibana (prediction logs)

---

### J3: System Engineering & Interaction

**Responsibility:** Unified dashboard, incident management, user interface, real-time coordination

**Architecture:**

```
Frontend Layer (Browser/Mobile)
├─ Web Dashboard (Next.js + React)
│  ├─ Incident map visualization (Leaflet/Mapbox)
│  ├─ Risk heatmap (real-time prediction overlay)
│  ├─ Alert management panel
│  ├─ Device status table
│  ├─ Incident timeline
│  └─ Chat/messaging for team coordination
│
└─ Mobile App (Flutter)
   ├─ Responsive UI for field teams
   ├─ Offline incident form submission
   ├─ GPS-based geotagging
   └─ Push notifications

                ▼ (HTTP/WebSocket)

J3 Backend Services (Next.js + Node.js)
├─ API Layer (GraphQL + REST)
│  ├─ /api/incidents
│  │  ├─ [GET] List all incidents
│  │  ├─ [POST] Create new incident
│  │  ├─ [PUT] Update incident status
│  │  └─ [DELETE] Archive incident
│  │
│  ├─ /api/predictions
│  │  ├─ [GET] Get risk scores by region
│  │  └─ [GET] Historical risk trend
│  │
│  ├─ /api/devices
│  │  ├─ [GET] Device list + health
│  │  ├─ [POST] Device registration
│  │  └─ [PUT] Device configuration
│  │
│  └─ /api/metrics [GET]
│     └─ Prometheus-compatible metrics endpoint
│
├─ Real-time Coordination
│  ├─ Socket.IO Server (Port 3001)
│  │  ├─ Namespace: /incidents
│  │  ├─ Event: incident.created
│  │  ├─ Event: incident.updated
│  │  ├─ Event: risk_score.changed
│  │  └─ Broadcast to all connected dashboards
│  │
│  └─ Kafka Consumer (Event Bridge)
│     ├─ Subscribe to: predictions.results
│     ├─ Subscribe to: incidents.updated
│     ├─ Subscribe to: alerts.fired
│     └─ Emit via Socket.IO to dashboard
│
├─ Business Logic
│  ├─ Incident Processing
│  │  ├─ Auto-acknowledge SOS reports
│  │  ├─ Assign incident ID + tracking
│  │  └─ Notify relevant authorities
│  │
│  ├─ Resource Allocation
│  │  ├─ Suggest responder routes
│  │  ├─ Calculate ETA to incident
│  │  └─ Track resource utilization
│  │
│  └─ Notification Engine
│     ├─ Generate alerts for high-risk areas
│     ├─ Push to mobile app
│     └─ Email/SMS to stakeholders
│
└─ Database Models (PostgreSQL)
   ├─ incidents
   │  ├─ id, type, severity, location
   │  ├─ created_at, resolved_at
   │  └─ assigned_responders, status
   │
   ├─ devices
   │  ├─ id, type, location, last_heartbeat
   │  └─ battery_level, last_reading
   │
   ├─ predictions
   │  ├─ region_id, risk_type, score
   │  └─ timestamp, model_version
   │
   └─ audit_log
      ├─ actor, action, resource
      └─ timestamp, changes

                ▼

Data Flow
├─ Incident Created
│  ├─ Device → Kong → J3 API
│  ├─ Stored: PostgreSQL
│  ├─ Published: Kafka (incidents.updated)
│  ├─ Broadcast: Socket.IO (all dashboards)
│  └─ Logged: Elasticsearch (via Filebeat)
│
├─ Risk Prediction Updated
│  ├─ J2 → Kafka (predictions.results)
│  ├─ J3 Event Bridge consumes
│  ├─ Updates dashboard via Socket.IO
│  └─ Stored: PostgreSQL (predictions table)
│
└─ Alert Fired
   ├─ Prometheus → Alertmanager
   ├─ Webhook → J3 (/api/alerts/webhook)
   ├─ Stored: PostgreSQL (alerts table)
   ├─ Broadcast: Socket.IO (alert notification panel)
   └─ Logged: Elasticsearch
```

**Technology Stack:**
- **Frontend:** Next.js, React, TypeScript, TailwindCSS, Leaflet/Mapbox
- **Backend:** Next.js API routes, Node.js runtime
- **Real-time:** Socket.IO (WebSocket), Redis (optional session store)
- **Mobile:** Flutter/Dart
- **Protocols:** REST, GraphQL, WebSocket, Server-Sent Events
- **Containerization:** Docker (Dockerfile)

**API Specifications:**

```typescript
// REST API Examples

// Create Incident
POST /api/incidents
{
  type: "flood" | "landslide" | "weather",
  severity: "low" | "medium" | "high" | "critical",
  location: { latitude: number, longitude: number },
  description: string,
  device_id: string,
  photos?: string[] // Base64 encoded
}

// Response:
{
  id: "INC-2026-0521-001",
  status: "open",
  created_at: ISO8601,
  assigned_responders: string[],
  risk_score: 0–100
}

// Get Predictions
GET /api/predictions?region_id=rg_north&time_horizon=24h

// Response:
{
  predictions: [
    {
      region_id: "rg_north",
      risk_type: "flood",
      score: 78,
      confidence: 0.85,
      affected_area: GeoJSON,
      timestamp: ISO8601
    }
  ]
}
```

**Integration Points:**
1. **Receives from:** Kong (external requests), Kafka (predictions, incidents), PostgreSQL (data queries)
2. **Sends to:** Kong (public API), Kafka (incidents.updated), PostgreSQL (incident storage), Socket.IO (real-time UI)
3. **Monitored by:** Prometheus (/api/metrics), Filebeat (logs), Jaeger (request tracing)

---

### J4: Platform Security & Integration

**Responsibility:** API gateway, authentication, authorization, monitoring, and security infrastructure

**Architecture:**

```
TIER 1: API Gateway & Routing
├─ Kong API Gateway (Port 8000/8001)
│  ├─ Request routing (load balancing)
│  ├─ Rate limiting: 1000 req/min per client
│  ├─ Request/response transformation
│  ├─ Logging: All requests to Filebeat
│  ├─ Metering: Metrics export to Prometheus
│  └─ Plugins:
│     ├─ Authentication (OAuth 2.0/OIDC)
│     ├─ Rate limiting
│     ├─ Request logging
│     ├─ IP whitelisting
│     └─ CORS handling
│
└─ Routes:
   ├─ /api/v1/system/* → J3 DMS (port 3000)
   ├─ /api/v1/devices/* → J1 Device API (port 8081)
   ├─ /api/v1/predict/* → J2 Data Intelligence (port 8082)
   └─ /api/v1/audit/* → J4 Blockchain API (port 8084)

TIER 2: Authentication & Authorization
├─ Keycloak Identity Provider (Port 8180)
│  ├─ Protocol: OAuth 2.0 + OIDC
│  ├─ User Repository: PostgreSQL
│  ├─ Realm: disaster-response
│  ├─ Client Applications:
│  │  ├─ J3 DMS (public client)
│  │  ├─ Mobile App (native client)
│  │  └─ CLI Tools (service account)
│  │
│  ├─ Roles:
│  │  ├─ admin (system administration)
│  │  ├─ responder (field operations)
│  │  ├─ analyst (data analysis)
│  │  ├─ viewer (read-only access)
│  │  └─ device (device authentication)
│  │
│  └─ Flows:
│     ├─ Authorization Code Flow (Web/Mobile)
│     ├─ Client Credentials (Service-to-Service)
│     └─ Device Flow (IoT devices)
│
└─ Access Control
   ├─ Token Validation: Kong + JWT middleware
   ├─ Scope Verification: Per-API resource
   └─ Row-Level Security: PostgreSQL policies

TIER 3: Data Persistence
├─ PostgreSQL (Port 5432)
│  ├─ Database: j3db
│  │  ├─ incidents (incident reports)
│  │  ├─ devices (device registry)
│  │  ├─ predictions (ML predictions)
│  │  ├─ audit_log (activity trail)
│  │  └─ chat_messages (team coordination)
│  │
│  ├─ Database: kong
│  │  ├─ routes (API routes)
│  │  ├─ services (upstream services)
│  │  └─ plugins (middleware)
│  │
│  ├─ Database: keycloak
│  │  ├─ users (identity store)
│  │  ├─ roles (authorization roles)
│  │  └─ credentials (passwords/tokens)
│  │
│  └─ Infrastructure:
│     ├─ Primary-replica replication
│     ├─ Automated backups (daily)
│     ├─ Point-in-time recovery (7 days)
│     └─ Connection pooling (PgBouncer, 100 connections)

TIER 4: Observability & Monitoring
├─ Metrics Collection
│  ├─ Prometheus (port 9090)
│  │  ├─ Scrape interval: 15s
│  │  ├─ Targets:
│  │  │  ├─ Kong (9001/metrics)
│  │  │  ├─ Keycloak (8180/metrics)
│  │  │  ├─ PostgreSQL (9187/metrics)
│  │  │  ├─ Kafka (9308/metrics)
│  │  │  ├─ J3 DMS (3000/api/metrics)
│  │  │  ├─ Prometheus itself (9090/metrics)
│  │  │  └─ Alertmanager (9093/metrics)
│  │  │
│  │  └─ Alert Rules: 15 rules covering
│  │     ├─ Service availability
│  │     ├─ Error rates
│  │     ├─ Latency SLOs
│  │     ├─ Resource utilization
│  │     └─ Storage/database health
│  │
│  └─ Alert Evaluation (every 15s)
│
├─ Alert Routing
│  ├─ Alertmanager (port 9093)
│  ├─ Route hierarchy:
│  │  ├─ Critical → Immediate webhook + Email
│  │  ├─ Warning → Delayed webhook (30s)
│  │  └─ Info → Webhook only (12h repeat)
│  │
│  ├─ Receivers:
│  │  ├─ Default: J3 webhook handler
│  │  ├─ Email: pisandeniwith@gmail.com
│  │  └─ (Slack/PagerDuty integration ready)
│  │
│  └─ Grouping & Deduplication:
│     ├─ Group wait: 10s (collect related alerts)
│     ├─ Group interval: 30s (send grouped alerts)
│     └─ Repeat interval: 1h–12h (based on severity)
│
├─ Dashboards & Visualization
│  ├─ Grafana (port 3030)
│  ├─ Datasources:
│  │  ├─ Prometheus (metrics)
│  │  └─ Elasticsearch (logs)
│  │
│  ├─ Pre-built Dashboards:
│  │  ├─ System Health (6 panels)
│  │  ├─ Incident Tracking (5 panels)
│  │  ├─ API Performance (8 panels)
│  │  ├─ Database Health (6 panels)
│  │  ├─ Event Streaming (4 panels)
│  │  └─ Observability System (5 panels)
│  │
│  └─ Alert Panel:
│     ├─ Firing alerts (current)
│     ├─ Historical alert trends
│     └─ Alert acknowledgment tracking
│
├─ Centralized Logging
│  ├─ Filebeat (log shipper)
│  │  ├─ Source: Docker container stdout/stderr
│  │  ├─ Protocol: Beats (TLS, compression)
│  │  └─ Destination: Logstash (5044)
│  │
│  ├─ Logstash (log processor)
│  │  ├─ Input: Beats (5044)
│  │  ├─ Filter: JSON parsing, Grok patterns
│  │  ├─ Enrichment: Add service, timestamp
│  │  └─ Output: Elasticsearch (bulk indexing)
│  │
│  ├─ Elasticsearch (log storage)
│  │  ├─ Index: drs-logs-YYYY.MM.dd (daily rollover)
│  │  ├─ ILM Policy:
│  │  │  ├─ Hot: 1 day (active writes)
│  │  │  ├─ Warm: 7 days (read-heavy)
│  │  │  ├─ Cold: 30 days (archival)
│  │  │  └─ Delete: 30+ days (cleanup)
│  │  │
│  │  ├─ Retention: 30 days (configurable)
│  │  └─ Compression: Enabled (50% reduction)
│  │
│  └─ Kibana (log search UI)
│     ├─ Data view: drs-logs-*
│     ├─ Search: KQL syntax
│     ├─ Visualizations: Charts, tables, maps
│     └─ Dashboards: Pre-built views (Kong, Incidents, Errors)
│
├─ Distributed Tracing
│  ├─ Jaeger Agent (port 6831/UDP)
│  │  ├─ Receives spans from applications
│  │  ├─ Batching: 1000 spans or 1s
│  │  └─ Forward to Jaeger Collector
│  │
│  ├─ Jaeger Collector
│  │  ├─ Span processing
│  │  ├─ Sampling: Head-based (10% traces)
│  │  └─ Storage: In-memory (7 day retention)
│  │
│  └─ Jaeger UI (port 16686)
│     ├─ Service dependency graph
│     ├─ Trace visualization (timeline view)
│     ├─ Latency analysis (p50/p95/p99)
│     └─ Error detection & correlation
│
└─ Security & Compliance
   ├─ TLS/HTTPS
   │  ├─ Kong → Services (encrypted)
   │  ├─ Clients → Kong (encrypted)
   │  └─ Certificates: Self-signed for dev, CA-signed for prod
   │
   ├─ Network Security
   │  ├─ Docker network isolation
   │  ├─ Port exposure: Only Kong (8000) public
   │  ├─ Internal services: Container-to-container (DNS)
   │  └─ Firewall rules: TBD (production)
   │
   ├─ Data Encryption
   │  ├─ At-rest: Database encryption (optional)
   │  ├─ In-transit: TLS 1.3
   │  └─ Secrets management: .env file (Docker secrets in prod)
   │
   └─ Audit Trail
      ├─ PostgreSQL audit_log table
      ├─ All API calls logged
      ├─ Incident changes tracked
      ├─ User actions recorded
      └─ Blockchain audit (immutable)

TIER 5: Blockchain Audit (J4 Specialized)
├─ Hardhat Local Node (port 8545)
│  ├─ Ethereum-compatible blockchain
│  ├─ Used for: Development & testing
│  └─ Deployer: deploy-audit-contract service
│
├─ Smart Contract: IncidentAuditLog
│  ├─ Stores immutable incident records
│  ├─ Methods:
│  │  ├─ addIncidentRecord(incident_id, hash)
│  │  ├─ getIncidentRecord(incident_id)
│  │  └─ getAuditTrail(incident_id)
│  │
│  └─ Events: IncidentRecorded, IncidentVerified
│
└─ J4 Audit API (port 8084)
   ├─ REST endpoints:
   │  ├─ [POST] /api/audit/record – Store on blockchain
   │  ├─ [GET] /api/audit/verify – Verify incident authenticity
   │  └─ [GET] /api/audit/trail – Get full audit history
   │
   └─ Integration: Called by J3 DMS on incident resolution
```

**Technology Stack:**
- **API Gateway:** Kong 3.7 (Lua + Go)
- **Identity Provider:** Keycloak 24.0 (Java/JavaEE)
- **Database:** PostgreSQL 16 (C)
- **Blockchain:** Hardhat + Solidity
- **Metrics:** Prometheus, Alertmanager, Exporters (Go)
- **Dashboards:** Grafana (Go/React)
- **Logs:** Elasticsearch, Logstash, Kibana, Filebeat (Java/Go)
- **Tracing:** Jaeger (Go)

**Key Features:**
- **High Availability:** Multi-replica setup, connection pooling
- **Performance:** Caching (Redis, Nginx cache), CDN-ready
- **Security:** OAuth 2.0/OIDC, TLS, audit logging, blockchain verification
- **Scalability:** Horizontal scaling (Kong, Logstash, J3), database sharding ready

**Integration Points:**
1. **Receives from:** All services (metrics, logs, traces)
2. **Sends to:** All services (authentication tokens, configuration)
3. **Stores in:** PostgreSQL (all data), Elasticsearch (logs), Prometheus (metrics)

---

## Infrastructure & Supporting Services

### PostgreSQL Database

**Role:** Central data repository for all persistent application state

**Databases Created:**

| Database | Purpose | Tables | User |
|----------|---------|--------|------|
| **j3db** | J3 DMS business data | incidents, devices, predictions, audit_log, chat_messages, users | postgres |
| **kong** | Kong configuration | routes, services, plugins, consumers | kong |
| **keycloak** | Keycloak identity store | users, roles, credentials, tokens | keycloak |

**Key Tables:**

```sql
-- Incidents (J3)
CREATE TABLE incidents (
  id VARCHAR PRIMARY KEY,
  type VARCHAR,              -- "flood", "landslide", "weather"
  severity VARCHAR,          -- "low", "medium", "high", "critical"
  location GEOMETRY,         -- PostGIS point
  description TEXT,
  device_id VARCHAR,
  created_at TIMESTAMP,
  resolved_at TIMESTAMP,
  assigned_responders TEXT[],
  status VARCHAR,            -- "open", "acknowledged", "in_progress", "resolved"
  risk_score NUMERIC(3,0),   -- 0–100
  created_by VARCHAR,
  updated_by VARCHAR
);

-- Devices (J3)
CREATE TABLE devices (
  id VARCHAR PRIMARY KEY,
  type VARCHAR,              -- "flood_monitor", "landslide_monitor", "weather_station"
  location GEOMETRY,
  last_heartbeat TIMESTAMP,
  battery_level NUMERIC(3,0),
  last_reading JSONB,
  status VARCHAR,            -- "active", "inactive", "error"
  created_at TIMESTAMP
);

-- Predictions (J2)
CREATE TABLE predictions (
  id SERIAL PRIMARY KEY,
  region_id VARCHAR,
  risk_type VARCHAR,         -- "flood", "landslide", "weather"
  score NUMERIC(3,0),        -- 0–100
  confidence NUMERIC(3,2),   -- 0–1
  affected_area GEOMETRY,
  time_horizon VARCHAR,      -- "1h", "6h", "24h"
  factors JSONB,             -- array of contributing factors
  timestamp TIMESTAMP
);

-- Audit Log (J4)
CREATE TABLE audit_log (
  id SERIAL PRIMARY KEY,
  actor VARCHAR,
  action VARCHAR,
  resource_type VARCHAR,
  resource_id VARCHAR,
  changes JSONB,             -- before/after values
  timestamp TIMESTAMP,
  ip_address INET
);
```

**Performance Optimization:**
- **Indexes:** On device_id, created_at, location, status
- **Partitioning:** Incidents partitioned by month (incidents_2026_05, etc.)
- **Connection Pooling:** PgBouncer (100 connections)
- **Replication:** Primary + 1 read replica

---

### Apache Kafka

**Role:** Event streaming backbone for asynchronous communication and log ingestion

**Topics & Partitions:**

| Topic | Partitions | Replicas | Retention | Producers | Consumers |
|-------|-----------|----------|-----------|-----------|-----------|
| **devices.reports** | 10 (by device_id) | 1 | 24h | J1 Edge API | J3 DMS, J2 Data Intelligence |
| **predictions.results** | 5 (by region) | 1 | 7d | J2 Model Server | J3 Event Bridge, PostgreSQL |
| **incidents.updated** | 3 | 1 | 24h | J3 DMS | Elasticsearch (via Logstash) |
| **alerts.fired** | 2 | 1 | 24h | Prometheus (via custom pusher) | J3 Event Bridge, Elasticsearch |

**Event Schemas:**

```json
// Topic: devices.reports
{
  "device_id": "dev_flood_001",
  "type": "flood",
  "severity": "high",
  "location": { "lat": 6.9271, "lon": 80.7744 },
  "readings": {
    "water_level": 45.2,
    "flow_rate": 120.5
  },
  "timestamp": "2026-05-08T14:32:15Z",
  "battery_level": 85
}

// Topic: predictions.results
{
  "region_id": "rg_north",
  "risk_type": "flood",
  "score": 78,
  "confidence": 0.85,
  "affected_area": {
    "type": "Polygon",
    "coordinates": [...]
  },
  "time_horizon": "6h",
  "timestamp": "2026-05-08T14:32:15Z",
  "model_version": "v2.1.0"
}

// Topic: incidents.updated
{
  "incident_id": "INC-2026-0508-001",
  "action": "status_changed",
  "previous_status": "open",
  "new_status": "in_progress",
  "assigned_responder": "resp_001",
  "timestamp": "2026-05-08T14:35:00Z"
}
```

---

### Event Bridge (J3)

**Role:** Kafka consumer that translates events into real-time dashboard updates

**Architecture:**

```javascript
// Node.js + Socket.IO
const kafka = new Kafka({ brokers: ["kafka:29092"] });
const consumer = kafka.consumer({ groupId: "j3-event-bridge" });

await consumer.subscribe({
  topics: ["predictions.results", "incidents.updated", "alerts.fired"],
  fromBeginning: false
});

await consumer.run({
  eachMessage: async ({ topic, message }) => {
    const event = JSON.parse(message.value);
    
    switch(topic) {
      case "predictions.results":
        io.emit("risk_score:updated", event);  // Broadcast to all connected clients
        break;
      case "incidents.updated":
        io.emit("incident:updated", event);
        break;
      case "alerts.fired":
        io.emit("alert:fired", event);
        break;
    }
  }
});
```

---

## Data Flow & Integration

### Complete Request Flow Example: "Device SOS Report"

```
STEP 1: Device Sends Report
┌─────────────────────────────────────┐
│  Flood Monitor Device (J1)          │
│  Water level > critical threshold   │
│  Sends: POST /api/v1/device/sos     │
└────────────┬────────────────────────┘
             │ HTTP + Device Certificate
             ▼
┌──────────────────────────────────────┐
│  Kong API Gateway (Port 8000)        │
│  1. Check rate limit                 │
│  2. Validate device certificate      │
│  3. Route to J3 DMS                  │
│  4. Log request to Filebeat          │
└────────────┬─────────────────────────┘
             │ HTTP
             ▼
┌──────────────────────────────────────┐
│  J3 DMS Backend (Port 3000)          │
│  1. Parse request body               │
│  2. Create Incident record           │
│  3. Assign ID: INC-2026-0508-001     │
└────────────┬─────────────────────────┘
             │
    ┌────────┴────────┬────────┬────────┐
    ▼                 ▼        ▼        ▼

STEP 2: Data Persistence & Broadcast
┌──────────────────┐ ┌──────────────┐ ┌─────────────────┐
│  PostgreSQL      │ │ Kafka Topic  │ │ Socket.IO       │
│  INSERT:         │ │ "incidents   │ │ Broadcast to    │
│  incidents table │ │ .updated"    │ │ dashboard       │
│  ID: INC-...     │ │ Event sent   │ │ connections     │
└──────────────────┘ └──────────────┘ └─────────────────┘
                           │
                           │ Consumed by:
                    ┌──────┴──────┐
                    ▼             ▼
            ┌────────────────┐ ┌───────────────┐
            │ J2 Data Intel  │ │ ELK Pipeline  │
            │ Immediately    │ │ Filebeat →    │
            │ runs risk      │ │ Logstash →    │
            │ prediction     │ │ Elasticsearch │
            │ model          │ └───────────────┘
            └────────┬───────┘
                     │
                     ▼
            ┌────────────────────┐
            │ Sends prediction   │
            │ result via Kafka   │
            │ Topic: predictions │
            │ .results           │
            └────────┬───────────┘
                     │
                 Event Bridge
                 consumes &
                 broadcasts via
                 Socket.IO
                     │
                     ▼
            ┌────────────────────┐
            │ Dashboard Updates  │
            │ Map marker +       │
            │ Risk heatmap +     │
            │ Incident panel     │
            └────────────────────┘

STEP 3: Monitoring & Alerting
┌─────────────────────────────────────┐
│ Prometheus (15s scrape)             │
│ 1. Poll /api/metrics from all svcs  │
│ 2. Evaluate alert rules             │
│ 3. If condition met: Fire alert     │
└────────────┬────────────────────────┘
             │ Alert fires:
             │ "HighIncidentRate"
             ▼
┌─────────────────────────────────────┐
│ Alertmanager (port 9093)            │
│ 1. Receive alert                    │
│ 2. Group by: alertname, job         │
│ 3. Match route: Critical            │
│ 4. Deduplicate (10s wait)           │
│ 5. Send: Webhook + Email            │
└────────────┬────────────────────────┘
             │
    ┌────────┴────────┐
    ▼                 ▼
┌──────────────┐ ┌──────────────────┐
│ J3 Webhook   │ │ Email Service    │
│ Handler      │ │ (SMTP)           │
│ /api/alerts/ │ │ To: operator@... │
│ webhook      │ │ Subject: ALERT   │
└──────────────┘ └──────────────────┘
```

---

## Component Interactions

### Service Mesh Diagram

```
┌─────────────────────────────────────────────────────────┐
│ External Clients (Mobile, Web, CLI)                    │
└────────────────┬────────────────────────────────────────┘
                 │ HTTPS (TLS 1.3)
                 ▼
        ┌────────────────────┐
        │ Kong API Gateway   │
        │ - Routing          │
        │ - Rate limiting    │
        │ - Authentication   │
        └─┬──────────────────┘
          │
    ┌─────┼─────┬───────┬───────┐
    ▼     ▼     ▼       ▼       ▼
   J1    J2    J3      J4    Keycloak
  (8081) (8082) (3000) (8084) (8180)
   │      │      │       │      │
   │      │      │       │      └─────────► PostgreSQL (Keycloak DB)
   │      │      │       │
   │      │      │       └──────► Hardhat Node (Blockchain)
   │      │      │                 │
   │      │      │                 └─► Deploy-audit contract
   │      │      │
   │      │      ├──────────────► Socket.IO (3001) ──► Connected clients
   │      │      │
   │      │      ├──────────────► PostgreSQL (J3 DB)
   │      │      │
   │      │      ├──────────────► Kafka
   │      │      │                 ├─► Event Bridge consumes
   │      │      │                 └─► Broadcasts to Socket.IO
   │      │      │
   │      │      └──────────────► J3 DMS API
   │      │
   │      ├──────────────────────► PostgreSQL (Analytics)
   │      │
   │      ├──────────────────────► Kafka (predictions.results)
   │      │
   │      └──────────────────────► J2 FastAPI server
   │
   └──────────────────────────────► Kafka (devices.reports)
                                     │
                                     ├─► Consumed by J3
                                     └─► Consumed by J2

All Services ◄─────────────────────► Prometheus (metrics scraping)
                                     │
                                     ├─► Alertmanager (alerts)
                                     │
                                     ├─► Grafana (dashboards)
                                     │
                                     └─► Jaeger (traces)

All Services ───────────────────────► Filebeat (log collection)
                                     │
                                     └─► Logstash ──► Elasticsearch
                                                       │
                                                       └─► Kibana (search)
```

---

## Deployment Architecture

### Docker Compose Stack

**Service Startup Order (Dependency Graph):**

```
1. postgres ──────┐
                  │
2. kafka ─────────┼─────► kafka-exporter
                  │
3. postgres-exporter
                  │
4. keycloak ◄─────┘
   │
   ├─► kong-migration ──┐
   │                    │
   └─► kong-setup ◄─────┼─► kong
                         │
5. prometheus ◄─────────┤
   │                    │
   └─► alertmanager ────┤
                        │
6. elasticsearch ──────┤
   │                  │
   ├─► elasticsearch-setup
   │   │
   │   ├─► kibana-setup ──► kibana
   │   │
   │   └─► logstash ◄──────────┐
   │                           │
   ├─► filebeat ───────────────┘
   │
   └─► grafana ◄───── Prometheus datasource
                     + Elasticsearch datasource

7. j3-dms ◄─── all above (depends on)
   │
   ├─► j3-event-bridge
   │   │
   │   └─► Kafka consumer
   │
   └─► PostgreSQL connection

8. j2-data-intelligence ◄─── all databases + kafka
   │
   └─► Kafka producer (predictions.results)

9. hardhat-node ──────────────► deploy-audit-contract
   │
   └─► j4-audit-api
```

**Volumes & Data Persistence:**

| Service | Volume | Mount Point | Purpose |
|---------|--------|-------------|---------|
| postgres | postgres_data | /var/lib/postgresql/data | Database storage |
| elasticsearch | elasticsearch_data | /usr/share/elasticsearch/data | Log index storage |
| prometheus | prometheus_data | /prometheus | Metrics TSDB storage |
| grafana | grafana_data | /var/lib/grafana | Dashboard config |
| alertmanager | alertmanager_data | /alertmanager | Alert history |
| hardhat | hardhat-node-modules | /app/node_modules | Node dependencies |
| audit deployment | audit-deployment | /deployment | Contract addresses |

---

## Security & Access Control

### Authentication & Authorization Flow

```
USER LOGIN
    │
    ▼
┌──────────────────────┐
│ Keycloak Login Page  │
│ Realm: disaster-     │
│ response             │
└────────┬─────────────┘
         │ User credentials
         ▼
┌──────────────────────────────┐
│ Keycloak Verifies            │
│ - Check user exists          │
│ - Validate password          │
│ - Check user status          │
└────────┬─────────────────────┘
         │
         ▼
┌──────────────────────────────┐
│ Generate JWT Token           │
│ - sub: user_id              │
│ - email: user@example.com   │
│ - roles: [responder]        │
│ - exp: +24 hours            │
│ - iat: now                  │
└────────┬─────────────────────┘
         │ Redirect to J3 with code
         ▼
┌──────────────────────────────┐
│ J3 Frontend                  │
│ Store token in localStorage │
│ Add to Authorization header │
└────────┬─────────────────────┘
         │ HTTP: Authorization: Bearer <JWT>
         ▼
┌──────────────────────────────┐
│ Kong API Gateway             │
│ 1. Extract JWT               │
│ 2. Validate signature        │
│ 3. Check expiration          │
│ 4. Verify roles              │
└────────┬─────────────────────┘
         │
    ┌────┴──────────┐
    │ Valid         │ Invalid
    ▼               ▼
┌─────────────┐ ┌──────────────┐
│ Forward to  │ │ Return 401   │
│ microservice│ │ Unauthorized │
└─────────────┘ └──────────────┘
```

**Role-Based Access Control (RBAC):**

| Role | Permissions | Services | API Access |
|------|-------------|----------|-----------|
| **admin** | Full system access | All | All endpoints |
| **responder** | Create/update incidents, view all | J1, J3 | /api/incidents/* (POST, PUT), /api/devices/* (GET) |
| **analyst** | View & analyze data | J2, J3 | /api/predictions/* (GET), /api/incidents/* (GET) |
| **viewer** | Read-only access | J3 | /api/incidents/* (GET), /api/devices/* (GET) |
| **device** | Report sensor data | J1 | /api/device/report, /api/device/sos |

---

## System Metrics & Observability

### Key Performance Indicators (KPIs)

| KPI | Target | Monitoring | Alert Threshold |
|-----|--------|-----------|-----------------|
| **Service Availability** | 99.5% | Prometheus up metric | < 99% |
| **API Latency (p95)** | < 500ms | Prometheus histogram | > 1000ms |
| **Error Rate** | < 0.1% | Rate calculation | > 1% |
| **Database Connection Pool** | < 80% | PostgreSQL exporter | > 90% |
| **Log Ingestion Rate** | 1000+ events/sec | Logstash metrics | Backlog > 10k |
| **Incident Response Time** | < 30 minutes | Custom metric | None (SLA only) |
| **System Memory Usage** | < 80% | Node exporter | > 85% |
| **Disk Space** | < 80% | Node exporter | > 90% |

### Alert Rules

```yaml
# prometheus/alert_rules.yml (15 rules)

groups:
  - name: disaster_response_alerts
    interval: 15s
    rules:
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        annotations:
          summary: "{{ $labels.job }} is down"
          
      - alert: HighErrorRate
        expr: rate(request_errors_total[5m]) > 0.01
        for: 3m
        annotations:
          summary: "{{ $labels.job }} error rate > 1%"
          
      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(request_duration_seconds_bucket[5m])) > 1
        for: 5m
        annotations:
          summary: "{{ $labels.job }} p95 latency > 1s"
          
      # ... 12 more rules
```

---

## Quick Start Guide

### Prerequisites

- Docker & Docker Compose
- 8GB RAM minimum (16GB recommended)
- 10GB disk space
- Git

### Startup (5 minutes)

```bash
# Clone repository
git clone https://github.com/Disaster-Response-System-Group-J/disaster-response-system.git
cd disaster-response-system

# Copy environment template
cp .env.example .env

# Start all services
docker-compose up -d

# Verify all services running
docker-compose ps

# View logs
docker-compose logs -f j3-dms
```

### Access Services

| Service | URL |
|---------|-----|
| J3 Dashboard | http://localhost:3000 |
| Grafana | http://localhost:3030 |
| Kibana | http://localhost:5601 |
| Jaeger | http://localhost:16686 |
| Prometheus | http://localhost:9090 |
| Keycloak | http://localhost:8180 |
| Kong Admin | http://localhost:8001 |

---

## Conclusion

The Disaster Response System integrates four specialized subsystems coordinated through a robust platform layer, providing real-time incident management, AI-driven predictions, secure access control, and comprehensive observability. The architecture is designed for scalability, reliability, and rapid incident response.

For detailed documentation on specific subsystems, see:
- [Monitoring & Observability README](./MONITORING_OBSERVABILITY_README.md)
- [System Algorithms](./MONITORING_OBSERVABILITY_ALGORITHM.md)
- [Load Testing Guide](./MONITORING_OBSERVABILITY_LOAD_TESTING.md)
- [Operational Runbook](./MONITORING_OBSERVABILITY_OPERATIONAL_RUNBOOK.md)
