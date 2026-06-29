# Proxmox VE Integration - Developer Summary

## Overview

This integration collects performance and operational metrics from Proxmox VE environments, including both virtual machines and LXC containers. Metrics are collected via the Proxmox REST API and shipped to Elasticsearch for analysis and visualization.

The implementation is based on the [Prometheus PVE Exporter](https://github.com/prometheus-pve/prometheus-pve-exporter) architecture, which provides a well-tested reference implementation for collecting Proxmox metrics.

## Directory Structure

```
proxmox_pve/
├── manifest.yml                    # Main integration configuration
├── changelog.yml                   # Version history and changes
├── docs/
│   ├── README.md                  # User documentation
│   └── DEVELOPMENT.md             # This file
├── kibana/
│   └── dashboard.ndjson           # Dashboards and visualizations
├── img/
│   ├── proxmox.svg                # Proxmox logo/icon
│   ├── proxmox_pve_vm_dashboard.png          # VM dashboard screenshot
│   └── proxmox_pve_lxc_dashboard.png         # LXC dashboard screenshot
└── data_stream/
    ├── vm/                         # Virtual Machine metrics datastream
    │   ├── manifest.yml
    │   ├── fields/
    │   │   └── fields.yml         # Field definitions
    │   ├── elasticsearch/
    │   │   └── ingest_pipeline/
    │   │       └── default.yml    # Data processing pipeline
    │   ├── agent/
    │   │   └── input.yml          # Agent input configuration
    │   ├── sample_event.json       # Example metric event
    │   └── _dev/
    │       └── README.md          # Development guide
    └── lxc/                        # LXC Container metrics datastream
        ├── manifest.yml
        ├── fields/
        │   └── fields.yml
        ├── elasticsearch/
        │   └── ingest_pipeline/
        │       └── default.yml
        ├── agent/
        │   └── input.yml
        ├── sample_event.json
        └── _dev/
            └── README.md
```

## API Architecture

### Primary Endpoint
```
GET /api2/json/cluster/resources?type={vm|lxc}
```

This is the main API endpoint used by both this integration and the Prometheus PVE Exporter. It provides comprehensive metrics for all resources of a given type across the entire cluster in a single request.

**Advantages:**
- Single API call collects metrics for all VMs/containers
- Works across cluster (no need to query individual nodes)
- Consistent metric format for all resources
- Efficient - minimizes API calls

### Response Format

The API returns metrics including:
- **Dimensions (identifying info)**: id, vmid, name, node, type, template, tags
- **Status**: status (running/stopped/paused)
- **CPU**: cpu (ratio 0-1), maxcpu (cores)
- **Memory**: mem, maxmem (bytes), swap (for containers)
- **Disk**: disk (used), maxdisk (allocated), diskread, diskwrite (bytes)
- **Network**: netin, netout (bytes)
- **Uptime**: uptime (seconds)

### Alternative Node-specific Endpoints

For advanced use cases:
- `/api2/json/nodes/{node}/qemu` - List VMs on specific node
- `/api2/json/nodes/{node}/qemu/{vmid}/config` - VM configuration
- `/api2/json/nodes/{node}/qemu/{vmid}/status/current` - VM status
- `/api2/json/nodes/{node}/lxc` - List LXC containers on specific node
- `/api2/json/nodes/{node}/lxc/{vmid}/config` - Container configuration
- `/api2/json/nodes/{node}/replication` - Replication jobs
- `/api2/json/nodes/{node}/subscription` - Subscription info

## Data Processing Pipeline

### 1. Data Collection (Agent/Input)
The httpjson input in `data_stream/{vm|lxc}/agent/input.yml`:
- Makes authenticated GET request to `/cluster/resources`
- Filters by `type=vm` or `type=lxc` via query parameters
- Collects metrics at configurable interval (default 30s)
- Authenticates with username and password/token

### 2. Metric Transformation
Raw API fields are transformed to standardized metrics:
- CPU ratio (0-1) → percentage (0-100) in percent
- Byte values preserved as-is with proper units
- Status values mapped to keywords
- Uptime converted to seconds if needed

### 3. Field Mapping
The `fields/fields.yml` defines:
- Field names and types (gauge, counter, keyword)
- Metric units (bytes, percent, seconds)
- Dimension fields for data aggregation
- Descriptions for Kibana

### 4. Ingest Pipeline
The `elasticsearch/ingest_pipeline/default.yml`:
- Adds ECS (Elastic Common Schema) version
- Sets event module and dataset
- Adds service type
- Preserves original message

## Key Features

✅ **Agent-based Collection** - Uses Elastic Agent with httpjson input  
✅ **Cluster-wide Metrics** - Single API call collects all resources  
✅ **API Authentication** - Supports username/password or API tokens  
✅ **SSL Support** - Includes certificate verification options  
✅ **Time Series Index** - Optimized for Elasticsearch time series storage  
✅ **Pre-built Dashboards** - Ready-to-use Kibana visualizations  
✅ **ECS Compliance** - Follows Elastic Common Schema standards  

## Metrics Overview

### Virtual Machine Metrics

| Metric | Source Field | Type |
|--------|-------------|------|
| `proxmox.vm.cpu.usage.percent` | cpu (ratio) | gauge |
| `proxmox.vm.memory.usage.percent` | mem/maxmem | gauge |
| `proxmox.vm.disk.read.bytes` | diskread | counter |
| `proxmox.vm.disk.write.bytes` | diskwrite | counter |
| `proxmox.vm.network.in.bytes` | netin | counter |
| `proxmox.vm.network.out.bytes` | netout | counter |

### LXC Container Metrics

| Metric | Source Field | Type |
|--------|-------------|------|
| `proxmox.lxc.cpu.usage.percent` | cpu (ratio) | gauge |
| `proxmox.lxc.memory.usage.percent` | mem/maxmem | gauge |
| `proxmox.lxc.memory.swap.bytes` | swap | gauge |
| `proxmox.lxc.rootfs.used.bytes` | disk | gauge |
| `proxmox.lxc.network.in.bytes` | netin | counter |
| `proxmox.lxc.network.out.bytes` | netout | counter |

## Testing

### Sample Events
- `data_stream/vm/sample_event.json` - Example VM metric
- `data_stream/lxc/sample_event.json` - Example LXC metric

These can be used to:
- Test dashboard visualizations without a live Proxmox instance
- Validate field mappings
- Test Elasticsearch ingest pipelines

### Manual API Testing

```bash
# List all VMs
curl -u user@realm:password -k \
  'https://proxmox.example.com:8006/api2/json/cluster/resources?type=vm' | jq '.'

# List all LXC containers
curl -u user@realm:password -k \
  'https://proxmox.example.com:8006/api2/json/cluster/resources?type=lxc' | jq '.'
```

## Future Enhancements

- **Advanced Metrics**: Cache metrics, guest CPU time, IO wait times
- **Configuration Metadata**: OnBoot status, HA state, lock status
- **Per-interface Metrics**: Individual network interface statistics
- **Node Metrics**: Cluster-level resource aggregation
- **Alerting Rules**: Pre-defined alerts for common issues
- **Extended Data Streams**: Cluster status, replication, backup info
- **Custom Processors**: Advanced field extraction and enrichment

## Troubleshooting

### Common Issues

**No metrics collected**
- Verify API connectivity: `curl -u user:pass -k 'https://host:8006/api2/json/cluster/resources' | jq '.data | length'`
- Check agent logs for authentication errors
- Ensure user has `Sys.Audit` and `VM.Audit` permissions

**Incomplete metrics**
- Verify VMs/containers are running (stopped resources don't report all metrics)
- Check if collection interval is appropriate
- Inspect sample API response for expected fields

**High cardinality**
- Increase collection interval to reduce metric volume
- Use Elasticsearch ILM for data retention policies
- Consider downsampling with rollup jobs

## Related Resources

- [Proxmox VE API Docs](https://pve.proxmox.com/pve-docs/api-viewer/)
- [Prometheus PVE Exporter](https://github.com/prometheus-pve/prometheus-pve-exporter)
- [Elastic Common Schema](https://www.elastic.co/guide/en/ecs/current/)
- [Elastic Fleet Integration Development](https://www.elastic.co/guide/en/fleet/current/developer-guide.html)
