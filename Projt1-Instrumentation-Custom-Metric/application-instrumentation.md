# Application Instrumentation Guide (Metrics, Logs & Traces)

This guide dives into the core concepts of **Application Instrumentation**—the process of embedding telemetry hooks inside your source code—and features a hands-on architectural blueprint and implementation demo.

---

## 1. The Full-Stack Observability Flow

Before writing code, it is important to understand where application telemetry sits in the grand scheme of your infrastructure. Telemetry flows from individual runtimes up through your cluster layers into specialized storage backends.
```
┌───────────────────────────────────────────────────────────────┐
│                       APPLICATION LAYER                       │
│  • Metrics (prom-client)   • Logs (Pino JSON)  • Traces (OTel)│
└───────────────────────────────┬───────────────────────────────┘
                                │ Runs inside
                                ▼
┌───────────────────────────────────────────────────────────────┐
│                    KUBERNETES CLUSTER LAYER                   │
│  • Pod/Service States (kube-state-metrics)  • Container Logs  │
└───────────────────────────────┬───────────────────────────────┘
                                │ Hosted on
                                ▼
┌───────────────────────────────────────────────────────────────┐
│                       HOST / NODE LAYER                       │
│  • Machine Hardware Stats (Node Exporter)   • System Syslogs  │
└───────────────────────────────┬───────────────────────────────┘
                                │ Collected & Processed via:
                                ▼
┌───────────────────┐    ┌───────────────────┐    ┌───────────────────┐
│  PROMETHEUS TSDB  │    │   GRAFANA LOKI    │    │  JAEGER / TEMPO   │
│ (Metrics Engine)  │    │   (Logs Engine)   │    │  (Traces Engine)  │
└─────────┬─────────┘    └─────────┬─────────┘    └─────────┬─────────┘
          │                        │                        │
          └────────────────────────┼────────────────────────┘
                                   ▼
                       ┌───────────────────────┐
                       │  GRAFANA DASHBOARDS   │
                       │ (Unified Correlation) │
                       └───────────────────────┘
```
---

## 2. What is Application Instrumentation?

Instrumentation is the mechanism that makes a software application observable. Instead of guessing how your code behaves from the outside (Black-Box), instrumentation exposes its exact execution flow from the inside (White-Box).

### Auto-Instrumentation vs. Manual Instrumentation
*   **Auto-Instrumentation:** Uses runtime agents or module-wrapping to patch common framework libraries automatically. It gives you instant HTTP request rates, standard database latency numbers, and foundational connection traces without changing a line of your code.
*   **Manual Instrumentation:** Involves importing a specific SDK directly into your code to record business-specific rules. Use this when you need to calculate domain-level values (e.g., tracking checkouts by payment type) or timing custom data-processing algorithms.

---

## 3. Hands-On Demo: Implementing the Triad

Below is a practical application setup showcasing how to instrument a service using a Prometheus metric client (`prom-client`), structured logging (`pino`), and OpenTelemetry tracing dependencies.

### Step 1: Install Required Packages
```bash
npm install express prom-client pino pino-pretty @opentelemetry/api
```
## Step 2: The Core Application Server (server.js)

> 💡 **Note on Instrumentation Libraries:** This hybrid demo uses `prom-client` to format raw Prometheus metrics on the `/metrics` endpoint, alongside the standard `@opentelemetry/api` package to extract active context Trace IDs.

```javascript
const express = require('express');
const client = require('prom-client');
const pino = require('pino');
const api = require('@opentelemetry/api');

const app = express();
const PORT = 8080;

// 1. INSTRUMENTATION: Structured Logging (Pino JSON output)
const logger = pino({
  transport: { target: 'pino-pretty' } // Remove pino-pretty in production environments
});

// 2. INSTRUMENTATION: Setup Prometheus Metrics Ingestion
const registry = new client.Registry();
client.collectDefaultMetrics({ register: registry }); // Captures CPU, memory heap, event loop lag

// Define a Custom Metric (Counter)
const checkoutCounter = new client.Counter({
  name: 'app_checkout_total',
  help: 'Total number of successfully processed user orders',
  labelNames: ['payment_method', 'status'],
});
registry.registerMetric(checkoutCounter);

// Define a Custom Metric (Histogram for Latency)
const routeLatency = new client.Histogram({
  name: 'app_route_duration_seconds',
  help: 'Duration of HTTP processing requests in seconds',
  labelNames: ['route'],
  buckets: [0.1, 0.5, 1, 2, 5]
});
registry.registerMetric(routeLatency);


// Exposed Scraping Endpoint for Prometheus Server
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', registry.contentType);
  res.end(await registry.metrics());
});

// Business Logic Route
app.post('/checkout', async (req, res) => {
  const timer = routeLatency.startTimer({ route: '/checkout' });
  
  // 3. INSTRUMENTATION: Capture the active Distributed Trace Context safely
  const activeSpan = api.trace.getActiveSpan();
  const currentTraceId = activeSpan?.spanContext().traceId || 'simulated-trace-id-12345';

  try {
    // Simulate payment sequence logic...
    
    // Increment our custom counter metric
    checkoutCounter.inc({ payment_method: 'credit_card', status: 'success' });

    // Emit a highly structured JSON log attached directly to our Trace Context
    logger.info({
      message: "Order placed successfully",
      trace_id: currentTraceId,
      checkout_details: { items: 3, currency: "USD" }
    });

    res.status(200).send({ status: "Success", traceId: currentTraceId });
  } catch (error) {
    logger.error({ message: "Checkout breakdown encountered", error: error.message, trace_id: currentTraceId });
    res.status(500).send("Processing Error");
  } finally {
    timer(); // Stop the latency gauge recording
  }
});

app.listen(PORT, () => {
  logger.info(`Instrumented microservice is live on port ${PORT}`);
});
```
# Project 1: Application Instrumentation (Custom Metrics, Logs & Traces)

This directory demonstrates hands-on **Application Instrumentation** using a microservice architecture (`service-a` and `service-b`) deployed to Kubernetes. The application code is explicitly instrumented to emit the full Telemetry Triad: Metrics, Logs, and Traces.

---

## 🏗️ Instrumentation Architecture

Instead of treating our services as a black box, the application code is instrumented from the inside out to handle three distinct telemetry pipelines:
```
                  ┌────────────────────────────────────┐
                  │        APPLICATION RUNTIME         │
                  │            (service-a)             │
                  └────┬────────────┬────────────┬─────┘
                       │            │            │
        Pulls Metrics  │            │ Streams    │ Ships Traces
       (HTTP /metrics) ▼            │ Logs       ▼ (OTLP/gRPC)
         ┌─────────────────┐        │      ┌─────────────────┐
         │   prom-client   │        ▼      │   OpenTelemetry │
         │  (Prometheus)   │    ┌───────┐  │   (SDK/Jaeger)  │
         └─────────────────┘    │ Pino  │  └─────────────────┘
                                │(JSON) │
                                └───────┘
```
### 1. Metrics Instrumentation (`prom-client`)
The backend exposes a dedicated standard `/metrics` endpoint. We utilize specific Prometheus metric types depending on the behavior we want to track:
*   **Counter (`http_requests_total`)**: Cumulative metric that only goes up. Used to track request volumes and detect client/server error spikes.
*   **Gauge (`node_gauge_example`)**: A metric that fluctuates up and down. Used here to track asynchronous task durations or temporary queue backups.
*   **Histogram & Summary (`http_request_duration_seconds`)**: Samples observations in configurable buckets to monitor latency and evaluate high percentiles (like p95 or p99 slowness).

### 2. Structured Logging (`pino` & `morgan`)
*   Instead of raw unformatted text, the application outputs highly structured **JSON logs** via Pino.
*   This structure allows log aggregators (like Fluent Bit/EFK stack) to index key-value pairs effortlessly instead of relying on expensive regex expressions.

### 3. Distributed Tracing (`Jaeger`)
*   The application is instrumented to construct an end-to-end trace lifecycle. 
*   When a request hits `/call-service-b`, a unique **Trace ID** is generated and propagated across network boundaries down to `service-b`, highlighting exactly where cross-service network latencies reside.

---

## 🎯 Project Objectives

1.  **Implement Custom Metrics**: Use the `prom-client` library to write and expose custom application hooks.
2.  **Set Up Alerts**: Configure Alertmanager to trigger critical email alerts if a container experiences a crash loop (`kube_pod_container_status_restarts_total > 2`).
3.  **Centralize Logs**: Stream structured container outputs to an **EFK Stack** (Elasticsearch, Fluent Bit, Kibana).
4.  **Distributed Tracing**: Map service dependencies and performance profiles visually inside **Jaeger**.

---

## 🚀 Step-by-Step Implementation

### 1. Dockerize & Push to Registry
Build and tag your microservice containers from the project root:
```bash
# Dockerize microservice-a
docker build -t <YOUR_REGISTRY>/demoservice-a:v ./application/service-a/

# Dockerize microservice-b
docker build -t <YOUR_REGISTRY>/demoservice-b:v ./application/service-b/
```
### 2. Deploy to Kubernetes
Apply the configuration manifests directly into your dedicated namespace:
```
kubectl create ns dev
kubectl apply -k kubernetes-manifest/
```

3. Expose & Test Endpoints
Locate your external LoadBalancer DNS name and trigger traffic to explore application behaviors:

- `/ & /healthy` — Baseline service verification.

- `/metrics` — View the raw Prometheus text format output.

- `/logs` — Generates a sample block of structured JSON entries.

- `/serverError` — Simulates an explicit HTTP 500 failure status.

- `/call-service-b` — Executes a cross-service transaction to populate a Distributed Trace.
You can also use the automated load-generation script to simulate continuous traffic:
```bash
./test.sh <LOAD_BALANCER_DNS_NAME>
```
4. Configure Alerts & Alertmanager
Before deploying rules, update alerts-alertmanager-servicemonitor-manifest/alertmanagerconfig.yml with your SMTP configurations and recipient email address. Then apply the resources:

```bash
kubectl apply -k alerts-alertmanager-servicemonitor-manifest/
```
