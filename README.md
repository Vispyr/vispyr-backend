<div align="center">
  <a href="https://vispyr.com">
    <img src="./assets/vispyr-banner.png" alt="Vispyr Banner" width="400">
  </a>
</div>

# Backend

The purpose of Vispyr's backend is to ingest, store, and present the telemetry data produced by the user's application with the Vispyr agent attached. Grafana Alloy - which itself wraps around the OTel Collector - routes the flux of data from its ingestion point to each database. Prometheus, Tempo, and Pyroscope store metrics, traces, and profiling data, respectively. A Grafana instance queries the databases and displays the result on panels for visualization of the telemetry data. The `docker-compose.yml` file orchestrates the spinning up of all these services.

## Architecture

<div align="center">
  <img src="assets/backend_architecture3.svg" alt="Collector Overview" width="600">
</div>


## Port Mapping

| Service | Port | Purpose |
|---------|------|--------|
| Gateway Collector (Alloy) | 4317 | OTLP traces and metrics via gRPC |
| Gateway Collector (Alloy) | 9090 | System metrics HTTP entry point |
| Gateway Collector (Alloy) | 9999 | Pyroscope profiles ingestion |
| Grafana | 3000 | Vispyr dashboard |
| Pyroscope | 4040 | Pyroscope UI |
| Tempo | 3200 | Tempo API |

## Service Details

### Gateway Collector (Grafana Alloy)

The Gateway Collector is the front door of the observability pipeline receiving telemetry data from all distributed applications that were instrumented by the Vispyr Agent. It batches the metrics and traces sent by the user application via Vipyr's Agent Collector in OTLP format via gRPC, and further processes these metrics into Prometheus format. From its HTTP ingestion points, the Gateway Collector also forwards system metrics received in OpenMetrics format, and profiles sent via the Agent's Pyroscope SDK instrumentation. The following diagram illustrates this process:

<div align="center">
  <img src="assets/gateway_collector4.svg" alt="Collector Overview" width="600">
</div>

### Prometheus

Prometheus receives metrics that are being pushed via its default remote write endpoint: `/api/v1/write` on port 9090. It's acting purely as a passive storage TSDB, i.e., it doesn't scrape data from anywhere. Vispyr uses Prometheus default configurations for all of its features. 

### Tempo

Tempo receives OTLP traces from the Gateway Collector via gRPC on port 4317 and generates metrics from these traces by enabling its `metrics_generator` feature. An S3 bucket is used for object retention of 30 days. Tempo spins up an HTTP server on port 3200 that is used by Grafana to query traces.

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

Grafana is provisioned with Vispyr's dashboard on its homepage. To build each panel from this dashboard, the above databases are queried in their respective languages: PromQL for Prometheus, TraceQL for Tempo, and FlameQL for Pyroscope. Vispyr's custom queries are used to build the dashboards from the data stored there. Nginx sits as reverse proxy between Grafana and the user accessing Vispyr.

# Learn more

Please refer to the [CLI documentation](https://github.com/Vispyr/vispyr-cli "Go to CLI page") for deployment.

For a more detailed and comprehensive description of Vispyr, along with the motivation for its creation, please read our [case study](https://vispyr.com "Go to Case Study").
