# 📊 Observability Knowledge Base

## 🔎 What is Observability?
Observability is the ability to understand the internal state of a system by examining its outputs.  
It goes beyond monitoring by answering **why** something is happening, not just **what** is happening.  

Observability is built on three pillars: **[Metrics](ca://s?q=Metrics_in_observability)**, **[Logs](ca://s?q=Logs_in_observability)**, and **[Traces](ca://s?q=Traces_in_observability)**.  

---

##  Visualization
```
+----------------------------------+
|   Observability                  |
+----------------+-----------------+
           |
+-----------+---------+-------------
|   Metrics | Logs  | Traces       | 
|-----------+--------+---------------
| Numbers   | Events| Request flows |
| CPU, Mem  | Errors| Latency, spans|
+-----------+-------+----------------

```


---

---

## 📌 Observability Stacks (Industry Popular Pairs)

| Pillar | Purpose | Core Tools | Popular Stack (Industry) |
|--------|---------|------------|---------------------------|
| **[Metrics](ca://s?q=Metrics_stack)** | Numeric measurements (CPU, latency, error rates) | Prometheus, InfluxDB, Grafana | **Prometheus + Grafana** |
| **[Logging](ca://s?q=Logging_stack)** | Centralized logs for debugging & auditing | Fluentd, Logstash, Elasticsearch, Kibana, Loki | **ELK**, **EFK**, **Loki** |
| **[Tracing](ca://s?q=Tracing_stack)** | Request flows across services, latency analysis | OpenTelemetry, Jaeger, Zipkin, Tempo | **Jaeger**, **Zipkin** |

---

## 📌 Metrics Stack
- **Collector** → Prometheus, Telegraf  
- **Storage** → Prometheus TSDB, InfluxDB, VictoriaMetrics  
- **Visualization** → Grafana  
- **Industry Pair** → [Prometheus + Grafana](ca://s?q=Prometheus_Grafana_stack)  

---

## 📌 Logging Stack
- **Collector/Processor** → Fluentd, Logstash, Filebeat  
- **Storage** → Elasticsearch, Loki, Splunk  
- **Visualization** → Kibana, Grafana  
- **Industry Pairs** → [ELK](ca://s?q=ELK_logging_stack), [EFK](ca://s?q=EFK_logging_stack), [Loki](ca://s?q=Loki_logging_stack)  

---

## 📌 Tracing Stack
- **Collector** → OpenTelemetry SDKs, Jaeger agents  
- **Storage** → Jaeger, Zipkin, Tempo  
- **Visualization** → Jaeger UI, Grafana Tempo  
- **Industry Pairs** → [Jaeger](ca://s?q=Jaeger_tracing_stack), [Zipkin](ca://s?q=Zipkin_tracing_stack)  

---

## 📜 Standards
- **[Structured Logging](ca://s?q=Structured_logging_in_observability)** → JSON format  
- **[Log Levels](ca://s?q=Log_levels_best_practices)** → ERROR, WARN, INFO, DEBUG  
- **[Correlation IDs](ca://s?q=Correlation_IDs_in_logging)** → trace_id, span_id  
- **[Timestamps](ca://s?q=Logging_timestamp_standards)** → ISO 8601  
- **[Sensitive Data Handling](ca://s?q=Sensitive_data_in_logs)** → Mask/redact secrets  

---

## ✅ Conclusion
This file documents observability through **metrics, logging, and tracing stacks**, highlighting the **popular industry pairs**:  
- **Prometheus + Grafana** for metrics  
- **ELK / EFK / Loki** for logging  
- **Jaeger / Zipkin** for tracing  

Together, these stacks form the backbone of modern observability in cloud‑native and enterprise systems.

