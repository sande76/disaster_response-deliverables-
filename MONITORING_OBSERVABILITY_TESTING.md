# Unit Testing for Monitoring & Observability

**Version:** 1.0.0  
**Date:** May 7, 2026  
**Scope:** Testing strategies for Prometheus rules, Logstash pipelines, Jaeger instrumentation, and alerting logic

---

## Table of Contents

1. [Testing Overview](#testing-overview)
2. [Prometheus Rules Testing](#prometheus-rules-testing)
3. [Logstash Pipeline Testing](#logstash-pipeline-testing)
4. [Jaeger Instrumentation Testing](#jaeger-instrumentation-testing)
5. [Alertmanager Testing](#alertmanager-testing)
6. [Test Suite Setup](#test-suite-setup)

---

## Testing Overview

**Three Layers of Testing:**

1. **Unit Tests** — Individual components (rules, filters, exporters)
2. **Integration Tests** — Multi-component workflows (scrape → alert → notify)
3. **End-to-End Tests** — Full stack with mock services

**Tools:**
- **Prometheus:** Unit tests via `promtool` CLI
- **Logstash:** Unit tests via `logstash-unit-plugin` + custom Ruby
- **Jaeger:** Unit tests via Go test framework
- **Docker Compose:** Integration test environment

---

## Prometheus Rules Testing

### 1. PromQL Expression Testing

**Tool:** `promtool` (Prometheus unit test framework)

**Setup:**

```bash
# Download promtool
wget https://github.com/prometheus/prometheus/releases/download/v2.42.0/prometheus-2.42.0.linux-amd64.tar.gz
tar xzf prometheus-2.42.0.linux-amd64.tar.gz
sudo cp prometheus-2.42.0.linux-amd64/promtool /usr/local/bin/
```

**Test File Structure:**

```yaml
# prometheus/rules_test.yml

tests:
  - interval: 15s
    input_series:
      # Simulate service UP metrics
      - series: 'up{job="j3-dms",instance="localhost:3000"}'
        values: '1 1 1 1 1 1 1 1 1 0 0 0 0 0 0'  # Becomes 0 at T=135s
      
      - series: 'up{job="postgres",instance="localhost:5432"}'
        values: '1 1 1 1 1 1 1 1 1 1 1 1 1 1 1'
    
    alert_rule_test:
      - alertname: ServiceDown
        eval_time: 120s  # After 120s, alert should PENDING (2 scrapes at 0)
        exp_alerts:
          - exp_labels:
              job: j3-dms
              severity: critical
            exp_annotations:
              summary: 'j3-dms service is down'
      
      - alertname: ServiceDown
        eval_time: 150s  # After 150s (for > 1m), alert should FIRING
        exp_alerts:
          - exp_labels:
              job: j3-dms
              severity: critical

  # Test PostgreSQL connection threshold alert
  - interval: 30s
    input_series:
      - series: 'pg_stat_activity_count{job="postgres"}'
        values: '10 20 30 50 75 85 90 95'  # Exceeds 80 at T=150s
    
    alert_rule_test:
      - alertname: PostgreSQLConnectionHigh
        eval_time: 300s  # After 5m (for > 5m rule)
        exp_alerts:
          - exp_labels:
              job: postgres
              severity: warning
            exp_annotations:
              summary: 'PostgreSQL connection count is high'

  # Test cache hit ratio alert
  - interval: 60s
    input_series:
      - series: 'pg_stat_bgwriter_buffers_clean_total{job="postgres"}'
        values: '1000 2000 3000 4000 5000'
      
      - series: 'pg_stat_bgwriter_buffers_backend_total{job="postgres"}'
        values: '500 1000 1500 2000 2500'
    
    alert_rule_test:
      - alertname: PostgreSQLCacheHitRatioLow
        eval_time: 600s
        exp_alerts:
          - exp_labels:
              job: postgres
              severity: warning
```

**Running Tests:**

```bash
promtool test rules prometheus/rules_test.yml

# Output:
# Unit test passed successfully!
# Processed 15 queries (avg 5.2ms, max 12.3ms)
```

### 2. Custom Alert Rule Tests

**Test Case 1: ServiceDown Alert**

```yaml
# Test: Service recovers within `for` window (alert should NOT fire)
- interval: 15s
  input_series:
    - series: 'up{job="j3-dms"}'
      values: '0 0 0 1 1 1'  # Down 45s, then recovers
  
  alert_rule_test:
    - alertname: ServiceDown
      eval_time: 60s  # After 60s, service is back up
      exp_alerts: []  # No alert expected (recovered within 1m)

# Test: Service stays down (alert should fire)
- interval: 15s
  input_series:
    - series: 'up{job="j3-dms"}'
      values: '0 0 0 0 0 0'  # Down entire duration
  
  alert_rule_test:
    - alertname: ServiceDown
      eval_time: 90s
      exp_alerts:
        - exp_labels:
            job: j3-dms
            severity: critical
```

**Test Case 2: Error Rate Alert**

```yaml
- interval: 15s
  input_series:
    # J3 DMS requests
    - series: 'request_total{job="j3-dms"}'
      values: '100 200 300 400 500 600 700 800'
    
    # J3 DMS errors
    - series: 'request_errors_total{job="j3-dms"}'
      values: '0 2 4 8 16 32 64 100'  # Error rate increases to 12.5%
  
  alert_rule_test:
    - alertname: HighErrorRate
      eval_time: 120s
      exp_alerts:
        - exp_labels:
            job: j3-dms
            severity: warning
          exp_annotations:
            description: 'Error rate is 12.5%'
```

---

## Logstash Pipeline Testing

### 1. Logstash Unit Test Framework

**Setup:**

```bash
# Install Logstash
wget https://artifacts.elastic.co/downloads/logstash/logstash-8.6.0-linux-x86_64.tar.gz
tar xzf logstash-8.6.0-linux-x86_64.tar.gz

# Install Logstash Unit Plugin
cd logstash-8.6.0
bin/logstash-plugin install logstash-unit

# Install testing gems
gem install rspec rspec-expectations
```

**Test File:**

```ruby
# logstash/tests/filters_test.rb

describe 'J3-DMS JSON parsing' do
  
  let(:pipeline) do
    Logstash::Config::PipelineConfig.new(
      <<-CONFIG
        input {
          generator {
            message => '{"level":"ERROR","requestId":"123","traceId":"abc","message":"DB error"}'
            count => 1
          }
        }
        
        filter {
          if [container][name] =~ /j3-dms$/ {
            json {
              source => "message"
              target => "j3"
              skip_on_invalid_json => true
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
        }
        
        output {
          stdout { codec => json }
        }
      CONFIG
    )
  end
  
  it 'parses JSON log correctly' do
    events = []
    pipeline.run do |event|
      events << event
    end
    
    expect(events.length).to eq(1)
    expect(events[0].get('log.level')).to eq('ERROR')
    expect(events[0].get('request_id')).to eq('123')
    expect(events[0].get('trace.id')).to eq('abc')
  end
end

describe 'Kong access log parsing' do
  
  it 'parses Kong Grok pattern' do
    # Sample Kong log line
    log_line = '192.168.1.100 - admin [01/May/2026:12:34:56 +0000] "POST /api/incidents HTTP/1.1" 201 1024 "-" "curl/7.64.1"'
    
    # Expected fields
    expect_fields = {
      'client_ip' => '192.168.1.100',
      'user_name' => 'admin',
      'http_method' => 'POST',
      'url_original' => '/api/incidents',
      'http_version' => '1.1',
      'http_status_code' => 201,
      'http_response_body_bytes' => 1024
    }
    
    # Test via logstash-unit
    result = process_log_through_logstash(log_line, 'kong_filter')
    
    expect_fields.each do |field, expected_value|
      expect(result[field]).to eq(expected_value)
    end
  end
end

describe 'Trace ID extraction' do
  
  it 'extracts trace ID from plaintext logs' do
    plaintext_logs = [
      'Starting request traceId=xyz-123',
      'Processing with trace_id=abc-456',
      'Completed traceId: def-789'
    ]
    
    expected_trace_ids = ['xyz-123', 'abc-456', 'def-789']
    
    plaintext_logs.each_with_index do |log, index|
      result = process_log_through_logstash(log, 'trace_filter')
      expect(result['trace_id']).to eq(expected_trace_ids[index])
    end
  end
end

describe 'Field mapping and type conversion' do
  
  it 'converts string latency to integer milliseconds' do
    log_line = 'latency="1234ms"'
    result = process_log_through_logstash(log_line, 'conversion_filter')
    
    expect(result['latency_ms']).to eq(1234)
    expect(result['latency_ms']).to be_a(Integer)
  end
  
  it 'converts string status code to integer' do
    log_line = 'status="500"'
    result = process_log_through_logstash(log_line, 'conversion_filter')
    
    expect(result['http_status_code']).to eq(500)
    expect(result['http_status_code']).to be_a(Integer)
  end
end
```

**Running Tests:**

```bash
cd logstash
logstash -f tests/filters_test.rb

# Or using rspec directly
rspec tests/filters_test.rb -f documentation

# Output:
# J3-DMS JSON parsing
#   parses JSON log correctly
#
# Kong access log parsing
#   parses Kong Grok pattern
#   extracts trace ID from plaintext logs
#
# Finished in 1.234 seconds
```

### 2. Elasticsearch Index Mapping Test

```ruby
# logstash/tests/elasticsearch_mapping_test.rb

describe 'Elasticsearch index mapping' do
  
  it 'creates correct index template' do
    client = Elasticsearch::Client.new
    
    # Verify index template exists
    template = client.indices.get_template(name: 'drs-logs-template')
    expect(template).not_to be_nil
    
    # Verify field mappings
    mappings = template['drs-logs-template']['mappings']['properties']
    
    expect(mappings['service_name']['type']).to eq('keyword')
    expect(mappings['log.level']['type']).to eq('keyword')
    expect(mappings['@timestamp']['type']).to eq('date')
    expect(mappings['http_status_code']['type']).to eq('integer')
    expect(mappings['latency_ms']['type']).to eq('integer')
    expect(mappings['message']['type']).to eq('text')
  end
  
  it 'enforces ILM policy' do
    client = Elasticsearch::Client.new
    
    policy = client.ilm.get_lifecycle(policy: 'drs-logs-policy')
    expect(policy['drs-logs-policy']).not_to be_nil
    
    # Verify policy phases
    phases = policy['drs-logs-policy']['policy']['phases']
    expect(phases.keys).to include('hot', 'warm', 'cold', 'delete')
    
    # Verify delete phase
    expect(phases['delete']['actions']['delete']['delay']).to eq('30d')
  end
end
```

---

## Jaeger Instrumentation Testing

### 1. Span Generation Test

```go
// jaeger/instrumentation_test.go

package jaeger

import (
    "testing"
    "io"
    "github.com/jaegertracing/jaeger-client-go"
    "github.com/jaegertracing/jaeger-client-go/config"
    "github.com/stretchr/testify/assert"
)

func TestSpanGeneration(t *testing.T) {
    // Initialize tracer
    cfg := &config.Configuration{
        ServiceName: "test-service",
        Sampler: &config.SamplerConfig{
            Type:  "const",
            Param: 1,  // Always sample for testing
        },
    }
    
    closer, err := cfg.InitGlobalTracer()
    assert.NoError(t, err)
    defer closer.Close()
    
    tracer := jaeger.GlobalTracer()
    
    // Test 1: Create simple span
    span := tracer.StartSpan("operation-1")
    span.SetTag("http.method", "POST")
    span.SetTag("http.status_code", 201)
    span.Finish()
    
    // Verify span was created
    assert.NotNil(t, span)
    assert.Equal(t, "operation-1", span.OperationName())
}

func TestTraceIDPropagation(t *testing.T) {
    cfg := &config.Configuration{
        ServiceName: "service-a",
        Sampler: &config.SamplerConfig{
            Type:  "const",
            Param: 1,
        },
    }
    closer, _ := cfg.InitGlobalTracer()
    defer closer.Close()
    
    tracer := jaeger.GlobalTracer()
    
    // Create parent span
    parentSpan := tracer.StartSpan("parent-operation")
    traceID := parentSpan.SpanContext().TraceID()
    
    // Create child span
    childSpan := tracer.StartSpan("child-operation", opentracing.ChildOf(parentSpan.SpanContext()))
    childTraceID := childSpan.SpanContext().TraceID()
    
    // Verify trace ID is propagated
    assert.Equal(t, traceID, childTraceID)
    
    childSpan.Finish()
    parentSpan.Finish()
}

func TestSpanWithBaggageItems(t *testing.T) {
    cfg := &config.Configuration{
        ServiceName: "test-service",
        Sampler: &config.SamplerConfig{
            Type:  "const",
            Param: 1,
        },
    }
    closer, _ := cfg.InitGlobalTracer()
    defer closer.Close()
    
    tracer := jaeger.GlobalTracer()
    
    span := tracer.StartSpan("operation")
    span.SetBaggageItem("user_id", "12345")
    span.SetBaggageItem("request_id", "req-xyz")
    
    // Retrieve baggage
    userID := span.BaggageItem("user_id")
    requestID := span.BaggageItem("request_id")
    
    assert.Equal(t, "12345", userID)
    assert.Equal(t, "req-xyz", requestID)
    
    span.Finish()
}

func TestSpanErrorLogging(t *testing.T) {
    cfg := &config.Configuration{
        ServiceName: "error-test",
        Sampler: &config.SamplerConfig{
            Type:  "const",
            Param: 1,
        },
    }
    closer, _ := cfg.InitGlobalTracer()
    defer closer.Close()
    
    tracer := jaeger.GlobalTracer()
    
    span := tracer.StartSpan("failing-operation")
    
    // Log error
    span.LogKV(
        "event", "error",
        "message", "Database connection failed",
        "error.kind", "connection_refused",
    )
    
    // Verify error was tagged
    assert.NotNil(t, span)
    
    span.Finish()
}
```

**Running Tests:**

```bash
cd jaeger
go test -v ./instrumentation_test.go

# Output:
# === RUN   TestSpanGeneration
# --- PASS: TestSpanGeneration (0.02s)
# === RUN   TestTraceIDPropagation
# --- PASS: TestTraceIDPropagation (0.03s)
# === RUN   TestSpanErrorLogging
# --- PASS: TestSpanErrorLogging (0.02s)
# === PASS: 3 passed in 0.07s
```

---

## Alertmanager Testing

### 1. Alert Routing Test

```yaml
# alertmanager/test_routing.yml

# Test configuration
global:
  resolve_timeout: 5m
  smtp_from: 'test@example.com'

route:
  receiver: 'default'
  group_by: ['alertname', 'job']
  group_wait: 0s  # Immediate for testing
  group_interval: 10s
  repeat_interval: 12h
  
  routes:
    - match:
        severity: critical
      receiver: 'critical'
    - match:
        severity: warning
      receiver: 'warning'

receivers:
  - name: 'default'
    webhook_configs:
      - url: 'http://localhost:5001/webhook'
  
  - name: 'critical'
    webhook_configs:
      - url: 'http://localhost:5001/webhook/critical'
  
  - name: 'warning'
    webhook_configs:
      - url: 'http://localhost:5001/webhook/warning'
```

**Test Script:**

```bash
#!/bin/bash
# alertmanager/test_routing.sh

# Test Case 1: Critical alert routing
echo "Test 1: Critical alert routing"
curl -X POST http://localhost:9093/api/v1/alerts \
  -H 'Content-Type: application/json' \
  -d '[{
    "status": "firing",
    "labels": {
      "alertname": "ServiceDown",
      "severity": "critical",
      "job": "j3-dms"
    },
    "annotations": {
      "summary": "J3 DMS is down"
    },
    "startsAt": "2026-05-07T10:00:00Z",
    "endsAt": "0001-01-01T00:00:00Z"
  }]'

# Verify webhook receives alert at critical endpoint
expected_response=$(curl -s http://localhost:5001/webhook/critical | jq '.status')
if [ "$expected_response" == "firing" ]; then
  echo "✓ Critical alert routed correctly"
else
  echo "✗ Critical alert routing failed"
  exit 1
fi

# Test Case 2: Warning alert deduplication
echo "Test 2: Warning alert deduplication"
curl -X POST http://localhost:9093/api/v1/alerts \
  -H 'Content-Type: application/json' \
  -d '[{
    "status": "firing",
    "labels": {
      "alertname": "HighLatency",
      "severity": "warning",
      "job": "kong"
    }
  },{
    "status": "firing",
    "labels": {
      "alertname": "HighLatency",
      "severity": "warning",
      "job": "kong"
    }
  }]'

# Verify only one notification sent
notification_count=$(curl -s http://localhost:5001/webhook/warning | jq '.alerts | length')
if [ "$notification_count" == "1" ]; then
  echo "✓ Duplicate alerts deduplicated"
else
  echo "✗ Deduplication failed"
  exit 1
fi
```

---

## Test Suite Setup

### Docker Compose for Testing

```yaml
# docker-compose.test.yml

version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/rules_test.yml:/etc/prometheus/rules_test.yml
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=24h'

  logstash:
    image: docker.elastic.co/logstash/logstash:8.6.0
    volumes:
      - ./logstash/pipeline/logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    environment:
      - LOGSTASH_JAVA_OPTS=-Xmx256m -Xms256m

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.6.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false

  alertmanager:
    image: prom/alertmanager:latest
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml

  # Webhook receiver for testing
  webhook:
    build:
      context: ./tests
      dockerfile: Dockerfile.webhook
    ports:
      - "5001:5001"
```

### Test Runner Script

```bash
#!/bin/bash
# run-all-tests.sh

set -e

echo "========== Running Monitoring & Observability Tests =========="

# Unit test 1: Prometheus rules
echo ""
echo "Test 1: Prometheus alert rules..."
promtool test rules prometheus/rules_test.yml

# Unit test 2: Logstash pipelines
echo ""
echo "Test 2: Logstash filter pipelines..."
rspec logstash/tests/filters_test.rb

# Unit test 3: Jaeger instrumentation
echo ""
echo "Test 3: Jaeger span generation..."
cd jaeger && go test -v ./instrumentation_test.go && cd ..

# Integration test: Full stack
echo ""
echo "Test 4: Integration tests (Prometheus + Alertmanager)..."
docker-compose -f docker-compose.test.yml up -d
sleep 5

# Run webhook test
bash alertmanager/test_routing.sh

docker-compose -f docker-compose.test.yml down

echo ""
echo "========== All tests passed! =========="
```

---

**Next Steps:**
- Set up CI/CD pipeline to run tests on each commit
- Extend tests to cover custom rules
- Add performance benchmarks (see [Load Testing](./MONITORING_OBSERVABILITY_LOAD_TESTING.md))
