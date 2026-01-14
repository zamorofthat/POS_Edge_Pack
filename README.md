# Criblcoffee-POS-Edge Pack

A Cribl Stream Pack designed to process, enrich, and route telemetry data from Point-of-Sale (POS) systems running in Kubernetes environments. This pack handles logs, metrics, and events from Odoo-based POS applications with intelligent filtering and noise reduction.

## Version

- **Pack Version:** 0.0.4
- **Display Name:** Criblcoffee-POS-Edge

## Overview

This pack provides comprehensive observability for POS systems by:

- Parsing and structuring application logs from Odoo/uvicorn services
- Processing Kubernetes events and container metrics
- Enriching data with Kubernetes context (namespace, pod, node)
- Reducing noise through intelligent sampling
- Standardizing fields for downstream analytics platforms

## Data Sources

The pack is designed to ingest data from the following sources:

| Source | Input ID | Description |
|--------|----------|-------------|
| Kubernetes Logs | `kube_logs:in_kube_logs` | Container logs from POS services |
| Kubernetes Events | `kube_events:in_kube_events` | K8s cluster events |
| Kubernetes Metrics | `kube_metrics:in_kube_metrics` | Container and pod metrics |
| System Metrics | `system_metrics:in_system_metrics` | Host-level metrics |
| Prometheus | `edge_prometheus:node_exporter` | Node exporter metrics |

## Pipelines

### pos-logs
Processes application logs from POS services including:
- **Uvicorn access logs** - Extracts client IP, HTTP method, path, status
- **JSON events** - Parses structured JSON log entries
- **HTTP client logs** - Captures inter-service communication

Features:
- Automatic log format detection
- Service name enrichment from environment
- 20% sampling for inter-service calls to reduce volume

### pos_metrics
Handles metrics from multiple sources:
- Prometheus metrics
- Kubernetes metrics
- System metrics

Features:
- Service name extraction from pod identifiers
- Namespace-based classification (POS vs Cribl services)
- 10% sampling for non-POS metrics

### odoo_cleanup
Comprehensive Odoo application log parser with specialized handling for:

- **Werkzeug (Web Server) logs** - Extracts source IP, method, endpoint, protocol, status, response time
- **Cron job events** - Parses job status, name, and duration
- **POS order events** - Extracts transaction details including sync_id, order_list, transaction_id, order_id

### Kube_Event_Logs
Extracts and standardizes Kubernetes events:
- Timestamp normalization
- Namespace and pod extraction
- Event reason and message capture

### process_parse
Parses process metrics with memory calculations:
- JSON extraction from raw data
- Memory percentage calculation: `(heap / rss * 100)`
- Field reduction to essential metrics

### cribl_metrics_rollup
Aggregates Cribl internal metrics:
- 30-second rollup windows
- Last value aggregation for gauges
- Metric normalization

### Groupnameadd
Enriches host field with group metadata:
- Format: `{host}-{group}-pos`

### passthru
Pass-through pipeline for CMDB and metadata routing.

## Routes

Routes are evaluated in the following order:

| Priority | Route Name | Filter | Pipeline |
|----------|------------|--------|----------|
| 1 | K8_events | Kubernetes events input | Kube_Event_Logs |
| 2 | pod_cmdb | Pod metadata with image info | passthru |
| 3 | cmdb_data | Helm deployment metadata | passthru |
| 4 | odoo_data | Kubernetes container logs | pos-logs |
| 5 | k8_ProcessMetrics | System/K8s/Prometheus metrics | pos_metrics |
| 6 | System Metrics | System metrics with host | Groupnameadd |
| 7 | default | All unmatched events | main |

## Sampling Rates

The pack implements intelligent sampling to reduce data volume:

| Data Type | Sample Rate | Keep Percentage |
|-----------|-------------|-----------------|
| Inter-service HTTP calls | 1:5 | 20% |
| Non-POS metrics | 1:10 | 10% |
| Health check endpoints | 1:5 | 20% |

## Monitored Namespaces

- `pos` - Point-of-Sale application services
- `cribl` - Cribl Stream services
- Other namespaces are sampled at reduced rates

## Installation

1. Download the pack or clone the repository
2. In Cribl Stream, navigate to **Processing > Packs**
3. Click **Add Pack** and select **Import from file**
4. Upload the pack or point to the repository
5. Configure your data sources to use the appropriate input IDs

## Configuration

### Required Inputs

Ensure the following inputs are configured in your Cribl Stream instance:

- `kube_logs:in_kube_logs` - Kubernetes log collector
- `kube_events:in_kube_events` - Kubernetes event watcher
- `kube_metrics:in_kube_metrics` - Kubernetes metrics collector
- `system_metrics:in_system_metrics` - System metrics collector
- `edge_prometheus:node_exporter` - Prometheus node exporter (optional)

### Expected Log Formats

**Uvicorn Access Logs:**
```
INFO: 192.168.1.1:5000 - "GET /api/endpoint HTTP/1.1" 200
```

**Werkzeug Logs:**
```
INFO:werkzeug:192.168.1.10 - - [14/Jan/2024 10:30:45] "GET / HTTP/1.1" 200 - 0.5 0.6
```

**Odoo Cron Logs:**
```
2024-01-14 10:30:45,123 1234 INFO module submodule: Starting job `export_sync` (1.2s).
```

## Directory Structure

```
POS_Edge_Pack/
├── package.json                    # Pack metadata
├── README.md                       # This file
└── default/
    ├── pack.yml                    # Pack configuration
    └── pipelines/
        ├── route.yml               # Route definitions
        ├── pos-logs/conf.yml       # POS logs pipeline
        ├── pos_metrics/conf.yml    # Metrics pipeline
        ├── odoo_cleanup/conf.yml   # Odoo log parser
        ├── Kube_Event_Logs/conf.yml
        ├── process_parse/conf.yml
        ├── passthru/conf.yml
        ├── Groupnameadd/conf.yml
        └── cribl_metrics_rollup/conf.yml
```

## Key Fields

| Field | Description |
|-------|-------------|
| `__inputId` | Source identification for routing |
| `dataSource` | Classified data source type |
| `kube_namespace` | Kubernetes namespace |
| `kube_pod` | Pod name |
| `service_name` | Extracted service identifier |
| `_time` | Normalized timestamp |

## License

Proprietary - Criblcoffee
