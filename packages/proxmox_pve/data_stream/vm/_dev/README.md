# Proxmox VE VM Integration Development Guide

## API Endpoints Used

The integration uses the Proxmox VE Cluster Resources API to collect VM metrics in a single call:

### Collect all VM metrics
```
GET /api2/json/cluster/resources?type=vm
```

This endpoint returns comprehensive metrics for all virtual machines across all nodes in the cluster, similar to how the Prometheus PVE Exporter works.

## Example API Response

```json
{
  "data": [
    {
      "id": "qemu/100",
      "vmid": 100,
      "name": "ubuntu-server-01",
      "node": "proxmox-node-01",
      "type": "qemu",
      "status": "running",
      "uptime": 604800,
      "cpu": 0.25,
      "maxcpu": 4,
      "mem": 6291456000,
      "maxmem": 8589934592,
      "disk": 53687091200,
      "maxdisk": 107374182400,
      "netin": 2147483648,
      "netout": 1073741824,
      "diskread": 1099511627776,
      "diskwrite": 549755813888,
      "template": false,
      "tags": "production,monitoring"
    }
  ]
}
```

## Metric Mapping

The `/cluster/resources` API provides these raw fields which are mapped to ECS metrics:

| API Field | ECS Metric | Unit | Type | Description |
|-----------|-----------|------|------|-------------|
| `cpu` | proxmox.vm.cpu.usage.percent | percent | gauge | CPU usage ratio (0-1) |
| `maxcpu` | proxmox.vm.cpu.cores | - | gauge | Number of CPUs allocated |
| `mem` | proxmox.vm.memory.used.bytes | bytes | gauge | Used memory |
| `maxmem` | proxmox.vm.memory.allocated.bytes | bytes | gauge | Allocated memory |
| `disk` | proxmox.vm.disk.used.bytes | bytes | gauge | Used disk space |
| `maxdisk` | proxmox.vm.disk.allocated.bytes | bytes | gauge | Allocated disk space |
| `diskread` | proxmox.vm.disk.read.bytes | bytes | counter | Bytes read from disk |
| `diskwrite` | proxmox.vm.disk.write.bytes | bytes | counter | Bytes written to disk |
| `netin` | proxmox.vm.network.in.bytes | bytes | counter | Bytes received |
| `netout` | proxmox.vm.network.out.bytes | bytes | counter | Bytes transmitted |
| `uptime` | proxmox.vm.uptime | seconds | gauge | VM uptime |
| `status` | proxmox.vm.status | - | keyword | running/stopped/paused |

## Alternative Node-specific API Endpoints

For individual node queries:

### List VMs on a specific node
```
GET /api2/json/nodes/{node}/qemu
```

### Get individual VM configuration
```
GET /api2/json/nodes/{node}/qemu/{vmid}/config
```

### Get VM status updates
```
GET /api2/json/nodes/{node}/qemu/{vmid}/status/current
```

## Development

To test the integration:

1. Configure a test Proxmox VE instance with VMs
2. Use the Proxmox Web UI to view current metrics
3. Test API authentication with `curl` or `pvesh`
4. Validate the cluster/resources endpoint returns expected metrics
5. Test data transformation and field mapping

## Example curl command to test the API

```bash
curl -u user@realm:password \
  -k \
  'https://proxmox.example.com:8006/api2/json/cluster/resources?type=vm' \
  | jq '.'
```

## Future Enhancements

- Support for additional metrics (guest CPU time, IO wait, cache metrics)
- VM configuration metadata collection from `/nodes/{node}/qemu/{vmid}/config`
- High Availability (HA) state metrics
- Lock state and migration status
- Per-interface network metrics
- Backup job status integration
- Alerting rules for common issues (high CPU, memory pressure)
