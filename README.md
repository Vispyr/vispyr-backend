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

| Service | Port | Purpose |
|---------|------|---------|
| Grafana | 3000 | Dashboard UI |
| Tempo | 3200 | Tempo API |
| Pyroscope | 4040 | Pyroscope UI |
| Alloy | 4317 | OTLP gRPC |
| Alloy | 9090 | Metrics scraping |
| Alloy | 9999 | Pyroscope ingestion |
| Alloy | 12345 | Debug UI |

## Local Deployment

The deployment of this stack is done automatically by [Vispyr's CLI](), but you can also deploy it locally by following these steps:
* Clone this repository.
* Make sure you have Docker daemon (`dockerd`) running.
* Run `docker compose up`.
* Send OTLP data in gRPC to `localhost:4317`.
