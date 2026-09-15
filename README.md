# End-to-End Infrastructure & Application Monitoring with Prometheus, Grafana, and Loki

An enterprise-ready, production-grade observability stack designed to collect, scrape, visualize, and alert on system metrics, container health, custom Node.js application telemetry, and centralized log files using **Prometheus**, **Grafana**, **Node Exporter**, **cAdvisor**, and **Loki**.

---

Below is the complete set of configuration and code files required to run the entire monitoring stack.

 1. `docker-compose.yml`

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:v2.47.0
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/alert.rules.yml:/etc/prometheus/alert.rules.yml
      - ./prometheus/recording.rules.yml:/etc/prometheus/recording.rules.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.enable-lifecycle'

  node-exporter:
    image: prom/node-exporter:v1.6.1
    container_name: node-exporter
    restart: unless-stopped
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.47.2
    container_name: cadvisor
    restart: unless-stopped
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    privileged: true
    devices:
      - /dev/kmsg

  grafana:
    image: grafana/grafana:10.1.2
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana_data:/var/lib/grafana

  loki:
    image: grafana/loki:2.9.0
    container_name: loki
    restart: unless-stopped
    ports:
      - "3100:3100"
    volumes:
      - ./loki/loki-config.yml:/etc/loki/loki-config.yml
    command: -config.file=/etc/loki/loki-config.yml

  promtail:
    image: grafana/promtail:2.9.0
    container_name: promtail
    restart: unless-stopped
    volumes:
      - ./promtail/promtail-config.yml:/etc/promtail/promtail-config.yml
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    command: -config.file=/etc/promtail/promtail-config.yml

  nodejs-app:
    build:
      context: ./app
      dockerfile: Dockerfile
    container_name: nodejs-app
    restart: unless-stopped
    ports:
      - "3001:3000"

volumes:
  prometheus_data:
  grafana_data:

 2. Prometheus Configuration 
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

 3. prometheus/recording.rules.yml
      groups:
  - name: infrastructure_rules
    rules:
      - record: job:node_cpu_utilization:avg5m
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

      - record: job:node_memory_usage_bytes
        expr: node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes

    4. Log Collector Configurations
      auth_enabled: false

server:
  http_listen_port: 3100

ingester:
  lifecycler:
    address: 127.0.0.1
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
    final_sleep: 0s
  chunk_idle_period: 5m
  chunk_retain_period: 30s

schema_config:
  configs:
    - from: 2020-05-15
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /tmp/loki/index
    cache_location: /tmp/loki/cache
    shared_store: filesystem
  filesystem:
    directory: /tmp/loki/chunks

limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h

5. promtail/promtail-config.yml
    server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          __path__: /var/log/*log

  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        target_label: 'container'

6. Application Source Code (app/)
 {
  "name": "monitored-nodejs-app",
  "version": "1.0.0",
  "description": "Sample application instrumented with Prometheus client metrics",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "prom-client": "^14.2.0"
  }
}

7. app/index.js
const express = require('express');
const client = require('prom-client');

const app = express();

// Enable collection of default system metrics
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

// Middleware to track latency and request counts
app.use((req, res, next) => {
  const end = httpRequestDurationMicroseconds.startTimer();
  res.on('finish', () => {
    httpRequestsTotal.inc({ method: req.method, route: req.path, status: res.statusCode });
    end({ method: req.method, route: req.path, code: res.statusCode });
  });
  next();
});

// Sample Routes
app.get('/', (req, res) => {
  res.send('Application is running.');
});

app.get('/api/data', (req, res) => {
  const delay = Math.random() * 500;
  setTimeout(() => {
    res.json({ message: 'Success', latency: `${delay.toFixed(2)}ms` });
  }, delay);
});

// Prometheus Metrics Endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics());
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Server listening on port ${PORT}`);
});

8. app/Dockerfile
FROM node:18-alpine

WORKDIR /usr/src/app

COPY package*.json ./
RUN npm install --production

COPY . .

EXPOSE 3000

CMD ["npm", "start"]

9. Directory Tree Setup
Ensure your local directory structure is created as follows before running:

Plaintext
.
├── docker-compose.yml
├── app/
│   ├── Dockerfile
│   ├── index.js
│   └── package.json
├── loki/
│   └── loki-config.yml
├── prometheus/
│   ├── alert.rules.yml
│   ├── prometheus.yml
│   └── recording.rules.yml
└── promtail/
    └── promtail-config.yml
Deployment & Operation Commands
Start the Stack:

Bash
docker compose up -d --build
Verify Running Containers:

Bash
docker compose ps
Access Services:

Grafana UI: http://<SERVER_IP>:3000 (User: admin, Password: admin)

Prometheus UI: http://<SERVER_IP>:9090

Application Metrics: http://<SERVER_IP>:3001/metrics

cAdvisor UI: http://<SERVER_IP>:8080

Grafana Data Source Configuration:

Navigate to Connections > Data Sources in Grafana.

Add Prometheus with URL: http://prometheus:9090.

Add Loki with URL: http://loki:3100.

Stop Stack:

Bash
docker compose down -v
