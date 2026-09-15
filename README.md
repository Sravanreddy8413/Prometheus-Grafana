

End-to-End Infrastructure & Application Monitoring with Prometheus, Grafana, and Loki
An enterprise-ready, production-grade observability stack designed to collect, scrape, visualize, and alert on system metrics, container health, custom Node.js application telemetry, and centralized log files using Prometheus, Grafana, Node Exporter, cAdvisor, and Loki.

Architecture Overview
+-----------------------------------------------------------------------------------+
|                                  SERVER HOST                                      |
|                                                                                   |
|  +--------------------+   +--------------------+   +---------------------------+  |
|  |   Node Exporter    |   |     cAdvisor       |   |    Node.js Application    |  |
|  |  (Host System Ops) |   | (Docker Metrics)   |   |   (prom-client /metrics)  |  |
|  +---------+----------+   +---------+----------+   +-------------+-------------+  |
|            |                        |                            |                |
|            | :9100                  | :8080                      | :3000          |
|            +------------------------+----------------------------+                |
|                                     |                                             |
|                                     v Scraping                                    |
|                       +---------------------------+                               |
|                       |     PROMETHEUS SERVER     | :9090                         |
|                       +-------------+-------------+                               |
|                                     |                                             |
|                                     v PromQL Queries                              |
|                       +---------------------------+                               |
|                       |      GRAFANA DASHBOARD    | :3000 (Host) / UI             |
|                       +-------------+-------------+                               |
|                                     ^                                             |
|                                     | Logql Queries                               |
|  +--------------------+   +---------+-------------+                               |
|  | Promtail (Logs)    |-->|   Grafana Loki Stack  | :3100                         |
|  +--------------------+   +-----------------------+                               |
+-----------------------------------------------------------------------------------+
Features
Host & System Monitoring: Real-time tracking of host CPU, memory, disk I/O, and network throughput via Node Exporter.

Container Telemetry: Per-container CPU, RAM usage, and network traffic via Google's cAdvisor.

Application Instrumentation: Prometheus custom metrics integrated into a sample Node.js service using prom-client.

Centralized Logging: Log collection and aggregation using Grafana Loki and Promtail, linked directly with metrics.

Alerting Pipeline: Automated Prometheus alert rules routed to Grafana notification channels (Slack / Email).

Performance Optimization: Prometheus recording rules for heavy time-series aggregations.

Repository Structure
Plaintext
├── docker-compose.yml          # Container orchestra for observability stack
├── prometheus/
│   ├── prometheus.yml          # Main Prometheus configuration & scrape targets
│   ├── alert.rules.yml         # Prometheus alerting rules
│   └── recording.rules.yml     # Pre-calculated metric aggregation rules
├── loki/
│   └── loki-config.yml         # Loki log store configuration
├── promtail/
│   └── promtail-config.yml     # Log scraping & processing configuration
├── app/
│   ├── index.js                # Node.js app instrumented with custom metrics
│   └── package.json
└── README.md
Quickstart Guide
Prerequisites
Linux host machine (Ubuntu 22.04 LTS recommended) or Cloud instance (DigitalOcean, AWS EC2, GCP)

Docker engine (>= 24.0.0) and Docker Compose (>= v2.20.0)

Ports 3000, 9090, 9100, 8080, 3100, and 9080 open in host firewall.

Installation & Setup
Clone the Repository

Bash
git clone https://github.com/your-username/prometheus-grafana-monitoring.git
cd prometheus-grafana-monitoring
Deploy the Observability Stack

Bash
docker compose up -d
Verify Service Health

Bash
docker compose ps
Configuration Details
1. Prometheus Scraping (prometheus/prometheus.yml)
YAML
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert.rules.yml"
  - "recording.rules.yml"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'nodejs-app'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['nodejs-app:3000']
2. Custom Application Instrumentation (app/index.js)
Expose custom application metrics using prom-client:

JavaScript
const express = require('express');
const client = require('prom-client');

const app = express();
const collectDefaultMetrics = client.collectDefaultMetrics;
collectDefaultMetrics({ timeout: 5000 });

// Custom Counter Metric
const httpRequestsTotal = new client.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status']
});

// Custom Histogram Metric for Latency
const httpRequestDurationMicroseconds = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'code'],
  buckets: [0.1, 0.5, 1, 1.5, 2, 5]
});

app.use((req, res, next) => {
  const end = httpRequestDurationMicroseconds.startTimer();
  res.on('finish', () => {
    httpRequestsTotal.inc({ method: req.method, route: req.path, status: res.statusCode });
    end({ method: req.method, route: req.path, code: res.statusCode });
  });
  next();
});

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics());
});

app.listen(3000, () => console.log('App listening on port 3000'));
Key PromQL Queries & Recording Rules
Essential PromQL Queries
CPU Utilization Rate (Host):

Code snippet
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
Memory Usage Percentage:

Code snippet
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100
Application Request Latency (95th Percentile):

Code snippet
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
Recording Rules (prometheus/recording.rules.yml)
YAML
groups:
  - name: infrastructure_rules
    rules:
      - record: job:node_cpu_utilization:avg5m
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
Alerting Strategy
Alerting Rules (prometheus/alert.rules.yml)
YAML
groups:
  - name: system_alerts
    rules:
      - alert: HighCpuLoad
        expr: job:node_cpu_utilization:avg5m > 85
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High CPU usage detected on {{ $labels.instance }}"
          description: "CPU load has been above 85% for more than 5 minutes."

      - alert: InstanceDown
        expr: up == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Target {{ $labels.instance }} down"
          description: "Scrape target has been unreachable for more than 2 minutes."
Grafana Dashboards & Access
Log into Grafana at http://<your-server-ip>:3000 (Default credentials: admin / admin).

Go to Configuration > Data Sources and add:

Prometheus: URL http://prometheus:9090

Loki: URL http://loki:3100

Import community dashboards for quick system visibility:

Node Exporter Full: ID 1860

cAdvisor Docker Insights: ID 14282

Verification & Testing
Prometheus UI: http://<your-server-ip>:9090/targets (Verify all endpoints display state UP).

Metrics Endpoint: http://<your-server-ip>:3000/metrics (Verify JSON metric output from Node.js app).

Simulate High CPU Load (To trigger Prometheus alerts):

Bash
stress --cpu 4 --timeout 300s
License
Distributed under the MIT License. See LICENSE for details.
