<div align="center">
  <a href="https://vispyr.com">
    <img src="./assets/vispyr-banner.png" alt="Vispyr Banner" width="400">
  </a>
</div>

# Backend

The purpose of Vispyr's backend is to ingest, store, and present the telemetry data produced by the user's application with the Vispyr agent attached. Grafana Alloy - which itself wraps around the OTel Collector - routes the flux of data from its ingestion point to each database. Prometheus, Tempo, and Pyroscope store metrics, traces, and profiling data, respectively. A Grafana instance queries the databases and displays the result on panels for visualization of the telemetry data. The `docker-compose.yml` file orchestrates the spinning up of all these services.

## Architecture

<div align="center">
  <img src="assets/backend_architecture.svg" alt="Collector Overview" width="600">
</div>


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

It's the front door of the observability pipeline receiving telemetry data from all distributed applications that were instrumented by the Vispyr agent. It batches the metrics and traces sent by OpenTelemetry SDK instrumentation in OTLP format via gRPC, and further processes these metrics into Prometheus format. From its HTTP ingestion points, Alloy also forwards Prometheus Node Exporter data received in OpenMetrics format, and profiles sent via the Pyroscope SDK instrumentation. The following diagram illustrates:

<div align="center">
  <img src="assets/gateway_collector3.svg" alt="Collector Overview" width="600">
</div>

### Prometheus

Prometheus receives metrics that are being pushed via its default remote write endpoint: `/api/v1/write` on port 9090. It's acting purely as a passive storage TSDB, i.e., it doesn't scrape data from anywhere. Vispyr uses Prometheus default configurations for all of its features. 

### Tempo

Tempo receives OTLP traces from Alloy via gRPC on port 4317 and generates metrics from these traces by enabling its `metrics_generator` feature. An S3 bucket is used for object retention of 30 days. Tempo spins up an HTTP server on port 3200 that is used by Grafana to query traces.

```
Applications ──► Alloy ──► Tempo ──────► S3 Storage
    (traces)      (OTLP)    │              (long-term)
                            │
                            ▼
                    Generated Metrics ──► Prometheus
                    (RED metrics)         (via remote_write)
```

### Pyroscope

Pyroscope stores data for profiles produced by the Pyroscope SDK. Profiles are received over HTTP and stored on disk within Pyroscope’s internal database. Pyroscope’s retention is based on capacity, so a specific retention period is not specified.

### Grafana

Grafana is provisioned with Vispyr's dashboard on its homepage. To build each panel from this dashboard, the above databases are queried in their respective languages: PromQL for Prometheus, TraceQL for Tempo, and FlameQL for Pyroscope. Vispyr's custom queries are used to build the dashboards from the data stored there.

# Learn more

For a more detailed and comprehensive description of Vispyr, along with the motivation for its creation, please read our [case study](https://vispyr.com "Go to Case Study").
