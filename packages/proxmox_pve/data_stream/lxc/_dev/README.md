# Proxmox VE LXC Integration Development Guide

## API Endpoints Used

The integration uses the Proxmox VE Cluster Resources API to collect container metrics in a single call:

### Collect all LXC container metrics
```
GET /api2/json/cluster/resources?type=lxc
```

This endpoint returns comprehensive metrics for all LXC containers across all nodes in the cluster, similar to how the Prometheus PVE Exporter works.

## Example API Response

```json
{
  "data": [
    {
      "id": "lxc/200",
      "vmid": 200,
      "name": "debian-container-01",
      "hostname": "debian-web-01",
      "node": "proxmox-node-01",
      "type": "lxc",
      "status": "running",
      "uptime": 86400,
      "cpu": 0.12,
      "maxcpu": 2,
      "mem": 1610612736,
      "maxmem": 2147483648,
      "swap": 268435456,
      "disk": 21474836480,
      "maxdisk": 53687091200,
      "netin": 536870912,
      "netout": 268435456,
      "template": false,
      "tags": "web,production"
    }
  ]
}
```

## Metric Mapping

The `/cluster/resources` API provides these raw fields which are mapped to ECS metrics:

| API Field | ECS Metric | Unit | Type | Description |
|-----------|-----------|------|------|-------------|
| `cpu` | proxmox.lxc.cpu.usage.percent | percent | gauge | CPU usage ratio (0-1) |
| `maxcpu` | proxmox.lxc.cpu.cores | - | gauge | Number of CPUs allocated |
| `mem` | proxmox.lxc.memory.used.bytes | bytes | gauge | Used memory |
| `maxmem` | proxmox.lxc.memory.allocated.bytes | bytes | gauge | Allocated memory |
| `swap` | proxmox.lxc.memory.swap.bytes | bytes | gauge | Swap memory used |
| `disk` | proxmox.lxc.rootfs.used.bytes | bytes | gauge | Used rootfs space |
| `maxdisk` | proxmox.lxc.rootfs.quota.bytes | bytes | gauge | Rootfs quota |
| `netin` | proxmox.lxc.network.in.bytes | bytes | counter | Bytes received |
| `netout` | proxmox.lxc.network.out.bytes | bytes | counter | Bytes transmitted |
| `uptime` | proxmox.lxc.uptime | seconds | gauge | Container uptime |
| `status` | proxmox.lxc.status | - | keyword | running/stopped/paused |

## Alternative Node-specific API Endpoints

For individual node queries:

### List LXC containers on a specific node
```
GET /api2/json/nodes/{node}/lxc
```

### Get individual container configuration
```
GET /api2/json/nodes/{node}/lxc/{vmid}/config
```

### Get container status updates
```
GET /api2/json/nodes/{node}/lxc/{vmid}/status/current
```

## Development

To test the integration:

1. Configure a test Proxmox VE instance with LXC containers
2. Use the Proxmox Web UI to view current metrics
3. Test API authentication with `curl` or `pvesh`
4. Validate the cluster/resources endpoint returns expected metrics
5. Test data transformation and field mapping

## Example curl command to test the API

```bash
curl -u user@realm:password \
  -k \
  'https://proxmox.example.com:8006/api2/json/cluster/resources?type=lxc' \
  | jq '.'
```

## Container-specific considerations

- **Swap memory**: LXC containers can have swap configured separately from main memory
- **Rootfs**: The root filesystem size in containers is different from VM disk allocation
- **Network metrics**: Container network traffic is often significant and should be monitored
- **Resource limits**: Containers share kernel with host, so CPU measurements are different from VMs

## Future Enhancements

- Per-interface network metrics
- Process-level metrics inside containers
- Mount point and filesystem metrics
- CGroup memory pressure indicators
- Container configuration metadata collection
- High Availability (HA) state metrics
- Alerting rules for resource constraints
- Backup job status integration
