# Grafana Performance Monitoring Dashboard

This directory contains a pre-configured Grafana dashboard for monitoring API performance and JMeter test metrics.

## Dashboard Features

The dashboard includes the following panels:

1. **API Request Rate by Endpoint** - Shows requests per second for each API endpoint
2. **95th Percentile Response Time** - Displays response time percentiles by route
3. **Error Rate** - Overall error percentage across all requests (4xx, 5xx status codes)
4. **Active Threads (JMeter)** - Number of active JMeter threads during load tests *
5. **Request Count by Status Code** - Breakdown of HTTP status codes over time
6. **HTTP Error Responses** - Detailed view of 4xx and 5xx errors by endpoint
7. **Request Success Rate** - Percentage of successful (2xx) requests
8. **Total Requests per Minute** - Total request volume

*Note: JMeter Active Threads panel requires the Prometheus listener plugin to be properly installed.

## How to Import the Dashboard

1. **Access Grafana**: Navigate to http://localhost:3001 (admin/admin)

2. **Add Prometheus Data Source**:
   - Go to Configuration → Data Sources
   - Click "Add data source"
   - Select "Prometheus"
   - Set URL to: `http://prometheus:9090`
   - Click "Save & Test"

3. **Import Dashboard**:
   - Go to Dashboards → Import
   - Click "Upload JSON file"
   - Select `grafana-dashboard.json` from this directory
   - Click "Import"

## Metrics Overview

### Application Metrics (from Node.js app)
- `http_request_duration_ms_count` - Request counter with labels: method, route, status_code
- `http_request_duration_ms_bucket` - Request duration histogram buckets

### JMeter Metrics (when Prometheus listener is configured)
- `jmeter_threads` - Active thread count
- `jmeter_sample_duration_ms` - Sample response times
- `jmeter_errors_total` - Error counts

## PromQL Queries Used

```promql
# API Request Rate
sum(rate(http_request_duration_ms_count[5m])) by (route)

# 95th Percentile Response Time
histogram_quantile(0.95, sum(rate(http_request_duration_ms_bucket[5m])) by (le, route))

# Error Rate
sum(rate(http_request_duration_ms_count{status_code!~"2.."}[1m])) / sum(rate(http_request_duration_ms_count[1m]))
```

## Dashboard Configuration

- **Refresh Rate**: 5 seconds
- **Time Range**: Last 15 minutes
- **UID**: `perf-monitoring`

The dashboard will automatically refresh and show real-time metrics during JMeter test runs.

## Troubleshooting

### JMeter Test Issues

If JMeter tests are failing with 100% error rate, check these common issues:

1. **Variable Resolution Error**: `URISyntaxException: Illegal character in authority`
   - **Cause**: JMeter variables like `${host}` are not being resolved
   - **Solution**: Use hardcoded values in the test plan or fix variable definitions
   - **Fixed in**: The test plan now uses hardcoded `application:3000` values

2. **Network Connectivity**:
   - **Test**: Run `docker run --rm --network=jenkins_net alpine/curl:latest curl http://application:3000/health`
   - **Expected**: Should return `{"ok":true}`

3. **Missing Prometheus Listener**:
   - **Symptom**: No JMeter metrics in Prometheus/Grafana
   - **Cause**: JMeter Prometheus listener plugin not installed
   - **Impact**: Application metrics work fine, only JMeter-specific metrics are missing

### Dashboard Issues

1. **No Error Metrics Showing**:
   - **Check**: Verify JMeter tests are actually making HTTP requests (not connection failures)
   - **Verify**: Look for HTTP status codes in application metrics: `http_request_duration_ms_count{status_code="400"}`

2. **Empty Panels**:
   - **Check**: Prometheus data source configuration (http://prometheus:9090)
   - **Verify**: Time range and refresh settings

## Current Status

✅ **Working**: Application HTTP metrics (request rates, response times, error rates)
✅ **Working**: JMeter connectivity and HTTP requests
✅ **Working**: Error tracking for HTTP 4xx/5xx responses
⚠️ **Partial**: JMeter-specific metrics (requires plugin installation)

The dashboard will show real-time error rates and performance metrics from successful JMeter test runs.