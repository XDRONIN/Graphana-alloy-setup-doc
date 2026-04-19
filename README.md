# Grafana Alloy — Observability Setup Guide

A comprehensive guide to collecting, processing, and exporting **Metrics**, **Logs**, and **Traces** using Grafana Alloy.

---

## Table of Contents

- [What is Grafana Alloy?](#what-is-grafana-alloy)
- [Core Concepts](#core-concepts)
- [Prerequisites](#prerequisites)
- [1. Collecting Metrics (Prometheus)](#1-collecting-metrics-prometheus)
- [2. Collecting Logs (Loki)](#2-collecting-logs-loki)
- [3. Collecting Traces (OpenTelemetry)](#3-collecting-traces-opentelemetry)
- [4. Storage & Visualization in Grafana](#4-storage--visualization-in-grafana)
- [5. Correlating Data — The Golden Path](#5-correlating-data--the-golden-path)
- [Setup Checklist](#setup-checklist)
- [Troubleshooting](#troubleshooting)

---

## What is Grafana Alloy?

**Grafana Alloy** is the official successor to the Grafana Agent. It is a vendor-neutral distribution of the [OpenTelemetry (OTel) Collector](https://opentelemetry.io/docs/collector/), purpose-built for collecting and routing observability data.

Alloy uses a declarative configuration language with the `.alloy` file extension. Inside these files, you define **components** (called *blocks*) that are chained together into **pipelines**.

### Why use Alloy?

| Feature | Benefit |
|---|---|
| Single binary | Replaces multiple agents (Prometheus, Promtail, OTel Collector) |
| Pipeline-based config | Readable, composable, and easy to debug |
| OTel-native | First-class support for traces, metrics, and logs |
| Vendor-neutral | Works with Prometheus, Loki, Tempo, Jaeger, Datadog, and more |
| Environment variable support | Configuration values can be injected via `sys.env()` — no secrets in config files |

---

## Core Concepts

Every pipeline in Alloy follows a three-stage pattern:

```
+----------------+     +----------------+     +----------------+
|    RECEIVER    | --> |   PROCESSOR    | --> |    EXPORTER    |
|                |     |                |     |                |
| Collects raw   |     | Transforms,    |     | Sends data to  |
| data           |     | enriches, or   |     | storage        |
|                |     | filters data   |     | backends       |
+----------------+     +----------------+     +----------------+
```

- **Receiver** — Pulls or listens for incoming data (e.g., scraping a `/metrics` endpoint, tailing a log file, listening on a gRPC/HTTP port).
- **Processor** — Optionally transforms or enriches the data (e.g., injecting environment labels, filtering noise, parsing log lines).
- **Exporter** — Forwards the processed data to a storage backend (e.g., Prometheus, Loki, Tempo).

> **Note:** Not every pipeline requires a processor. A receiver can be wired directly to an exporter when no transformation is needed.

---

## Prerequisites

Before you begin, ensure the following are in place:

- [ ] **Grafana Alloy** is installed on your application server ([Installation Guide](https://grafana.com/docs/alloy/latest/set-up/install/))
- [ ] A running **Prometheus** instance for metrics storage
- [ ] A running **Loki** instance for log storage
- [ ] A running **Tempo** instance for trace storage
- [ ] A running **Grafana** instance for visualization
- [ ] Network connectivity between Alloy and all backend services
- [ ] Your `.alloy` config file created (e.g., `/etc/alloy/config.alloy`)
- [ ] Environment variables set for all backend URLs and sensitive values (see the [Environment Variables](#environment-variables) section)

---

## Environment Variables

Alloy supports reading values from the host environment via `sys.env("VAR_NAME")`. All backend URLs, endpoints, and environment identifiers should be provided this way — never hardcoded in the config file.

| Variable | Description |
|---|---|
| `ENV_NAME` | Logical environment name (e.g., `production`, `staging`) — stamped on every metric, log, and trace |
| `MONITORING_SERVER_PROMETHEUS_URL` | Remote write URL for Prometheus |
| `MONITORING_SERVER_LOKI_URL` | Push URL for Loki |
| `MONITORING_SERVER_TEMPO_ENDPOINT` | gRPC endpoint for Tempo (no `http://` scheme) |
| `POSTGRES_DSN` | PostgreSQL connection string for `postgres_exporter` |

---

## 1. Collecting Metrics (Prometheus)

Alloy scrapes metrics from any HTTP endpoint that exposes data in Prometheus format (the `/metrics` path).

### How it works

```
Your App (/metrics) --> prometheus.scrape --> prometheus.remote_write --> Prometheus DB
```

### Step 1 — Define a Scrape Target

The `prometheus.scrape` component polls your application or exporters at a regular interval.

```hcl
prometheus.scrape "app_metrics" {
  targets = [
    { "__address__" = "<APP_HOST>:<PORT>", "env" = "production", "app" = "my-service" }
  ]

  scrape_interval = "15s"
  job_name        = "my-service"

  forward_to = [prometheus.remote_write.backend.receiver]
}
```

> **Tip:** You can add multiple entries in `targets` to scrape several services at once. Custom labels like `env` and `app` will be attached to every metric from that target, which is useful for filtering in Grafana.

### Step 2 — Forward Metrics to Prometheus

The `prometheus.remote_write` component sends scraped metrics to a Prometheus-compatible storage backend.

```hcl
prometheus.remote_write "backend" {
  endpoint {
    url = sys.env("MONITORING_SERVER_PROMETHEUS_URL")
  }
}
```

> **Note:** The `/api/v1/write` path is the standard Prometheus Remote Write endpoint. Ensure `--web.enable-remote-write-receiver` is enabled on your Prometheus server.

### Exporters

This setup supports scraping the following Prometheus exporters in addition to application metrics:

| Exporter | Default Port | Metrics Provided |
|---|---|---|
| `node_exporter` | `9100` | Host CPU, memory, disk, network |
| `cadvisor` | `8080` | Docker container resource usage |
| `nginx-prometheus-exporter` | `9113` | Nginx connections and request totals |
| `postgres_exporter` | `9187` | PostgreSQL query stats, connections, replication |

Each exporter is scraped as a separate `prometheus.scrape` block sharing the same `prometheus.remote_write` exporter, with a distinct `job_name` for easy filtering.

> **Note for `nginx-prometheus-exporter`:** Nginx must have `stub_status` enabled. The exporter exposes metrics including `nginx_connections_active`, `nginx_connections_reading`, `nginx_connections_writing`, `nginx_connections_waiting`, and `nginx_http_requests_total`.

> **Note for `postgres_exporter`:** The exporter requires a valid connection string supplied via the `POSTGRES_DSN` environment variable (e.g., `postgresql://user:pass@host:5432/db?sslmode=disable`). It surfaces `pg_stat_statements`, `pg_stat_bgwriter`, `pg_stat_replication`, and general database metrics.

---

## 2. Collecting Logs (Loki)

Alloy supports multiple log sources. This setup covers three: **Docker container logs**, **Nginx host logs**, and **PostgreSQL host logs**.

### Option A — Docker Container Logs

This approach automatically discovers all running containers via the Docker socket and streams their logs to Loki.

```
Docker Socket --> discovery.docker --> discovery.relabel --> loki.source.docker --> loki.write
```

**Step 1 — Discover containers:**

```hcl
discovery.docker "containers" {
  host             = "unix:///var/run/docker.sock"
  refresh_interval = "5s"
}
```

**Step 2 — Relabel discovered targets:**

This step normalises the container name label (Docker names start with a `/`, e.g., `/my-app`) and attaches additional labels for filtering.

```hcl
discovery.relabel "docker_meta" {
  targets = discovery.docker.containers.targets

  rule {
    source_labels = ["__meta_docker_container_name"]
    regex         = "/(.*)"
    target_label  = "container"
  }

  rule {
    source_labels = ["__meta_docker_container_image"]
    target_label  = "image"
  }

  rule {
    source_labels = ["__meta_docker_container_id"]
    target_label  = "container_id"
  }

  rule {
    target_label = "env"
    replacement  = sys.env("ENV_NAME")
  }
}
```

**Step 3 — Stream logs:**

```hcl
loki.source.docker "docker_logs" {
  host             = "unix:///var/run/docker.sock"
  targets          = discovery.relabel.docker_meta.output
  forward_to       = [loki.write.loki_backend.receiver]
  refresh_interval = "5s"
}
```

> **Note:** The relabeling step is important. Without it, the `container` label retains a leading `/`, which makes LogQL queries awkward.

---

### Option B — Local File Tailing

Use this for services running directly on the host (outside of Docker), such as Nginx or PostgreSQL. The log files must be mounted into the Alloy container as read-only volumes.

```
Log Files on Disk --> local.file_match --> loki.source.file --> loki.write
```

**Step 1 — Match log file paths:**

```hcl
local.file_match "system_logs" {
  path_targets = [
    { "__path__" = "/var/log/nginx/access.log", "job" = "nginx", "log_type" = "access" },
    { "__path__" = "/var/log/nginx/error.log",  "job" = "nginx", "log_type" = "error"  },
    { "__path__" = "/var/log/postgresql/*.log",  "job" = "postgresql"                   }
  ]
}
```

**Step 2 — Tail the files:**

```hcl
loki.source.file "files" {
  targets    = local.file_match.system_logs.targets
  forward_to = [loki.write.loki_backend.receiver]
}
```

> **Note:** Alloy tracks the last read position of each file, so it will not re-send old log lines after a restart.

---

### PostgreSQL Log Processing Pipeline

Raw PostgreSQL log lines can be enriched before being sent to Loki using `loki.process`. The pipeline below:

1. Extracts the PostgreSQL severity level (`LOG`, `WARNING`, `ERROR`, etc.) and maps it to a structured `level` label.
2. Tags any log line containing `duration:` (emitted when `log_min_duration_statement` is set) with `query_type=slow_query`.

```
loki.source.file --> loki.process (parse + label) --> loki.write
```

```hcl
loki.process "postgresql_log_pipeline" {
  forward_to = [loki.write.loki_backend.receiver]

  stage.regex {
    expression = `(?P<pg_level>LOG|WARNING|ERROR|FATAL|PANIC|DETAIL|HINT|CONTEXT|STATEMENT|NOTICE|DEBUG\d?):`
  }

  stage.labels {
    values = { level = "pg_level" }
  }

  stage.match {
    selector = `{job="postgresql"} |~ "duration:"`

    stage.static_labels {
      values = { query_type = "slow_query" }
    }
  }
}
```

This enables targeted alerting in Loki with queries such as `{job="postgresql", query_type="slow_query"}`.

---

### Forwarding Logs to Loki

All log sources — Docker containers, Nginx, and PostgreSQL — share the same `loki.write` exporter:

```hcl
loki.write "loki_backend" {
  endpoint {
    url = sys.env("MONITORING_SERVER_LOKI_URL")
  }
}
```

---

## 3. Collecting Traces (OpenTelemetry)

Alloy acts as an **OTLP (OpenTelemetry Protocol) receiver**, accepting distributed traces sent by your application's instrumentation libraries.

```
Your App (OTLP SDK) --> otelcol.receiver.otlp --> otelcol.processor.transform --> otelcol.exporter.otlp --> Tempo
```

### Step 1 — Listen for Incoming Traces

Configure Alloy to accept trace data over gRPC or HTTP:

```hcl
otelcol.receiver.otlp "app_traces" {
  grpc {
    endpoint = "0.0.0.0:<GRPC_PORT>"
  }

  http {
    endpoint = "0.0.0.0:<HTTP_PORT>"
  }

  output {
    traces = [otelcol.processor.transform.add_env.input]
  }
}
```

> **Standard OTel ports:** The default gRPC port is `4317` and the default HTTP port is `4318`. Align these with your application SDK configuration and firewall rules.

### Step 2 — Enrich Spans with Metadata

Before forwarding traces to storage, inject resource-level attributes — such as the environment name — into every span. This greatly improves filtering and correlation in Grafana.

```hcl
otelcol.processor.transform "add_env" {
  error_mode = "ignore"

  trace_statements {
    context = "resource"
    statements = [
      format(`set(attributes["service.environment"], "%s") where attributes["service.environment"] == nil`, sys.env("ENV_NAME")),
    ]
  }

  output {
    traces = [otelcol.exporter.otlp.tempo.input]
  }
}
```

> **Implementation note:** `otelcol.processor.transform` with `context = "resource"` is the correct Alloy component for setting resource-level attributes. Resource attributes are what Tempo's `metrics_generator` reads when building `spanmetrics` dimensions, which causes it to emit the `resource_service_environment` label on all `traces_spanmetrics_*` metrics.

> **Tip:** Additional attributes such as `service.region` or `service.version` can be injected using the same pattern by adding further `statements`.

### Step 3 — Export Traces to Tempo

```hcl
otelcol.exporter.otlp "tempo" {
  client {
    endpoint = sys.env("MONITORING_SERVER_TEMPO_ENDPOINT")  // gRPC — no http:// scheme

    tls {
      insecure = true
    }
  }
}
```

> **Security note:** `insecure = true` disables TLS verification. This is acceptable for private internal networks, but proper TLS should be configured for any environment accessible over the internet.

### Live Debugging

Alloy supports live debugging of pipeline components. To enable it, add the following block to your config:

```hcl
livedebugging {
  enabled = true
}
```

This allows you to inspect data flowing through components in real time using the Alloy UI.

---

## 4. Storage & Visualization in Grafana

Once data is flowing through Alloy into your backends, connect Grafana to each storage layer.

### Adding Data Sources

Navigate to **Grafana → Connections → Data Sources → Add new data source**.

| Data Type | Backend | Grafana Data Source | Query Language |
|---|---|---|---|
| Metrics | Prometheus | Prometheus | PromQL |
| Logs | Loki | Loki | LogQL |
| Traces | Tempo | Tempo | TraceQL |

Configure each data source by pointing it at the corresponding backend URL.

### Building Dashboards

| Use Case | Tool | Example Query |
|---|---|---|
| CPU / memory / request rate over time | PromQL | `rate(http_requests_total[5m])` |
| Error log counts | LogQL | `count_over_time({job="nginx-error"}[1m])` |
| Slow query detection | LogQL | `{job="postgresql", query_type="slow_query"}` |
| PostgreSQL slow query duration | LogQL | `{job="postgresql"} \|~ "duration:"` |
| Request latency breakdown | Trace View | Search by service name + time range |

---

## 5. Correlating Data — The Golden Path

The real power of the Grafana stack is the ability to **navigate between all three data types** without leaving the UI.

```
Metrics Dashboard
      |
      |  (spike detected -> "View Logs" data link)
      v
  Loki Logs
      |
      |  (log line contains trace_id -> "View Trace" link)
      v
  Tempo Trace
      |
      |  (trace contains service tags -> "Search Logs" link)
      +-------------------------------------------------> Back to Loki
```

### Setting Up Correlations

**Metrics → Logs** via Data Links:
In your Grafana panel, add a **Data Link** that passes the time range and labels to a Loki query. This lets you click a spike on a graph and jump directly to the relevant logs.

**Logs → Traces** via Derived Fields:
In your **Loki data source settings**, add a **Derived Field** matching `trace_id=(\w+)` with a regex. Link it to your Tempo data source. Any log line containing a `trace_id` will display a clickable button that opens the full trace.

**Traces → Logs** via Tempo–Loki Integration:
In your **Tempo data source settings**, enable **Loki Search**. Tempo will automatically query Loki for logs sharing the same `trace_id` or service labels, surfacing them alongside the trace waterfall view.

> **Prerequisite:** For all three correlations to work end-to-end, your application must propagate `trace_id` into its structured logs. Most OpenTelemetry SDKs support this automatically.

---

## Setup Checklist

Use this checklist when deploying for the first time:

- [ ] Install Grafana Alloy on the application server
- [ ] Set all required environment variables (`ENV_NAME`, backend URLs, `POSTGRES_DSN`)
- [ ] Create and populate your `.alloy` config file
- [ ] Mount host log directories into the Alloy container as read-only volumes
- [ ] Verify Alloy can reach Prometheus, Loki, and Tempo (check firewall rules)
- [ ] Confirm metrics are appearing in Prometheus (`/api/v1/query`)
- [ ] Confirm logs are appearing in Loki (use `logcli` or Grafana Explore)
- [ ] Confirm traces are appearing in Tempo (use Grafana Explore → Tempo)
- [ ] Add Prometheus, Loki, and Tempo as data sources in Grafana
- [ ] Import community dashboards or build custom ones
- [ ] Configure Metrics → Logs → Traces correlations
- [ ] (Production) Replace `insecure = true` with a proper TLS configuration
- [ ] (Production) Add authentication to all remote write and push endpoints

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| No metrics in Prometheus | Remote write not enabled | Add `--web.enable-remote-write-receiver` flag to Prometheus |
| No logs in Loki | Docker socket permission denied | Add the Alloy user to the `docker` group |
| No traces in Tempo | Wrong port or TLS mismatch | Verify the endpoint and `insecure` setting |
| Duplicate metrics | Multiple Alloy instances scraping the same target | Set consistent `external_labels` in Prometheus to deduplicate |
| Log → Trace link not working | `trace_id` missing from log lines | Ensure the application's OTel SDK injects trace context into structured logs |
| `service.environment` missing from traces | `ENV_NAME` not set | Ensure the `ENV_NAME` environment variable is available to the Alloy process |

---

## Resources

- [Grafana Alloy Documentation](https://grafana.com/docs/alloy/latest/)
- [Alloy Component Reference](https://grafana.com/docs/alloy/latest/reference/components/)
- [OpenTelemetry SDK Setup](https://opentelemetry.io/docs/instrumentation/)
- [Grafana Community Dashboards](https://grafana.com/grafana/dashboards/)
