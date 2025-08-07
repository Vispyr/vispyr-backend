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
│ (Node.js)       │  HTTP:4318  │           │             │  ┌─────────────┐ │
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

### Grafana
**Container:** `grafana`  
**Image:** `grafana/grafana`  
**Purpose:** Unified observability dashboard and visualization

**Features:**
- Pre-configured dashboards for metrics, traces, and profiles
- Data source integrations with Prometheus, Tempo, and Pyroscope
- Custom dashboard provisioning

**Configuration Files:**
- `./grafana/grafana.ini` - Main Grafana configuration
- `./grafana/dashboards/` - Dashboard definitions
- `./grafana/provisioning/` - Data source and dashboard provisioning

### Prometheus
**Container:** `prometheus`  
**Image:** `prom/prometheus`  
**Purpose:** Metrics storage and querying engine

**Features:**
- Remote write receiver enabled for external metric ingestion
- Time-series data storage and aggregation
- PromQL query interface

**Configuration:** `./prometheus/prom-config.yaml`

### Tempo
**Container:** `tempo`  
**Image:** `grafana/tempo`  
**Purpose:** Distributed tracing backend

**Features:**
- OTLP trace ingestion
- Trace storage and querying
- Distributed trace visualization support

**Configuration:** `./tempo/tempo.yaml`  
**Data Storage:** `./tempo/data`

### Pyroscope
**Container:** `pyroscope`  
**Image:** `grafana/pyroscope`  
**Purpose:** Continuous profiling platform

**Features:**
- Application performance profiling
- Integration with NodeJS profile emitter app

**Configuration:** `./pyroscope/pyro-config.yaml`

# Learn more

For a more detailed and comprehensive description of Vispyr, along the motivation for its creation, please read our [case study](https://vispyr.com "Go to Case Study").

