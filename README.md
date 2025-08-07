# Vispyr Backend

The main purpose of this entity is to ingest, store and present the telemetry data produced by the user's application with the Vispyr agent attached. Grafana Alloy - which itself wraps around OTel Collector - routes the flux of data from its ingestion point to each database. Prometheus, Tempo and Pyroscope store metrics, traces and profiling data, respectively. The `docker-compose.yml` file orchastrates the spinning up of all these services.

## Data Flow

1. **Telemetry Ingestion:** Applications send telemetry data to Grafana Alloy via OTLP endpoints
2. **Data Routing:** Alloy processes and routes data to appropriate backends:
   - Metrics → Prometheus
   - Traces → Tempo  
   - Profiles → Pyroscope
3. **Visualization:** Grafana queries all backends to provide unified dashboards

## Architecture

```
                              ┌─────────────┐
                              │     App     │
                              │(Telemetry)  │
                              └─────────────┘
                                     │
                              ┌─────────────┐
                              │Grafana Alloy│
                              │ (Collector) │
                              └─────────────┘
                                /     |     \
                               /      |      \
                    ┌─────────────┐   |   ┌─────────────┐
                    │ Prometheus  │   |   │   Tempo     │
                    │ (Metrics)   │---|---│ (Traces)    │
                    └─────────────┘   |   └─────────────┘
                           │          |          │
                           │    ┌─────────────┐  │
                           │    │ Pyroscope   │  │
                           │    │(Profiling)  │  │
                           │    └─────────────┘  │
                           │          │          │
                           └──────────┼──────────┘
                                      │
                              ┌─────────────┐
                              │   Grafana   │
                              │ (Dashboard) │
                              └─────────────┘
```

## Port Mapping

| Service | Port |
|---------|------|
| Alloy | 4317 |
| Alloy | 9090 |
| Alloy | 9999 |
| Alloy | 12345 |
| Grafana | 3000 |
| Pyroscope | 4040 |
| Tempo | 3200 |

## Service Details

### Grafana Alloy (Gateway Collector)

It's the front-door of the observability pipeline receiving telemetry data from all distributed applications that were instrumented via the Vispyr agent. It batches the metrics and traces sent through NodeJS instrumentation and further processes these metrics into Prometheus format. There are 3 flows of telemetry data passing through it:


```
FLOW 1: OpenTelemetry
┌─────────────────┐    OTLP     ┌───────────┐    Batch    ┌──────────────────┐
│ OpenTelemetry   │ ──────────► │   Alloy   │ ──────────► │   OTLP to        │
│ SDKs            │  gRPC:4317  │           │             │     OpenMetrics  │
│ (Node.js)       │             │           │             │  ┌─────────────┐ │
└─────────────────┘             └───────────┘             │  │ Metrics ────┼─┼──► Prometheus
                                                          └──│─────────────│─┘    :9090/write
                                                             │ Traces ─────┼───► Tempo
                                                             │             │     :4317 OTLP
                                                             └─────────────┘ 

FLOW 2: Prometheus
┌─────────────────┐   HTTP:9090  ┌───────────┐ remote_write ┌──────────────┐
│ Prometheus      │ ──────────►  │   Alloy   │ ───────────► │ Prometheus   │
│ Node Exporter   │   (push)     │           │              │ :9090/write  │
└─────────────────┘              └───────────┘              └──────────────┘

FLOW 3: Pyroscope
┌─────────────────┐   HTTP:9999  ┌───────────┐   forward    ┌──────────────┐
│ Pyroscope       │ ──────────►  │   Alloy   │ ───────────► │ Pyroscope    │
│ SDK             │   (profiles) │           │              │ :4040        │
└─────────────────┘              └───────────┘              └──────────────┘

```

### Prometheus

Prometheus receives metrics that's being pushed via its default remote write endpoint: `/api/v1/write` on port 9090. It's acting purely as a passive storage TSDB, i.e. it doesn't scrape data from anywhere. Vispyr's custom PromQL queries in Grafana are used to build the dashboards from the data stored here.

### Tempo

Tempo receives OTLP traces from Alloy via gRPC on port 4317 and generates metrics from these traces by the enabling of its `metrics_genetor` feature. An S3 bucket is used for object retention of 30 days. Tempo spins up an HTTP server on port 3200 that is used by Grafana to query traces.

```
Applications ──► Alloy ──► Tempo ──────► S3 Storage
    (traces)      (OTLP)    │              (long-term)
                            │
                            ▼
                    Generated Metrics ──► Prometheus
                    (RED metrics)         (via remote_write)
```

### Pyroscope

Similar to Prometheus, Pyroscope is not configured much beyond its default options: local filesystem storage, HTTP server on port 4040, retention period of 30 days and default API routes. The only overriding configuration is so that Pyroscope won't profile itself. 

### Grafana

Grafana is provisioned with Vispyr's dashboard in its homepage. To build each panel from this dashboard, the above databases are queried in their respective language: PromQL for Prometheus, TraceQL for Tempo and FlameQL for Pyroscope.

# Learn more

For a more detailed and comprehensive description of Vispyr, along the motivation for its creation, please read our [case study](https://vispyr.com "Go to Case Study").

