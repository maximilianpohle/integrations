# Proxmox VE Integration

This integration collects metrics from Proxmox VE virtual machines and LXC containers using the Proxmox API.

## Data Streams

### VM (Virtual Machines)
Collects metrics for Proxmox VE virtual machines including:
- CPU usage and allocation
- Memory usage and allocation
- Disk I/O operations and bytes
- Network traffic (in/out packets and bytes)
- Virtual machine status and uptime

### LXC (Containers)
Collects metrics for LXC containers including:
- CPU usage and allocation
- Memory usage, allocation, and swap
- Rootfs usage and quota
- Network traffic (in/out packets and bytes)
- Container status and uptime

## Requirements

- Proxmox VE 7.0 or later
- Elastic Agent 8.0 or later
- Elasticsearch 8.0 or later
- An API user account or API token for authentication

## Setup

### Create an API User (Optional)

You can either use an existing user or create a dedicated API user for monitoring:

```bash
# Create a user
pveum user add monitoring@pam --password yourpassword

# Create a role with API permissions
pveum role add Monitoring -privs "Sys.Audit VM.Audit"

# Add role to user
pveum aclmod / -user monitoring@pam -role Monitoring
```

### Create an API Token (Recommended)

```bash
# Generate an API token
pveum user token add monitoring@pam monitor_token --privsep 0

# This will output your token ID and secret
```

## Configuration

1. In Fleet, add this integration to your Proxmox VE host
2. Configure the following settings:
   - **Hosts**: URL(s) of your Proxmox VE API endpoint(s) (e.g., https://proxmox.example.com:8006)
   - **Username**: Your Proxmox VE username (e.g., monitoring@pam)
   - **Password/Token**: Your password or API token
   - **Realm**: Authentication realm (default: pam)
   - **Collection Interval**: How often to collect metrics (default: 30 seconds)
   - **Disable SSL verification** (optional): Only for self-signed certificates

## Metrics

### Virtual Machine Metrics

| Metric | Unit | Type | Description |
|--------|------|------|-------------|
| `proxmox.vm.cpu.usage.percent` | percent | gauge | CPU usage percentage |
| `proxmox.vm.memory.used.bytes` | bytes | gauge | Used memory |
| `proxmox.vm.memory.allocated.bytes` | bytes | gauge | Allocated memory |
| `proxmox.vm.memory.usage.percent` | percent | gauge | Memory usage percentage |
| `proxmox.vm.disk.used.bytes` | bytes | gauge | Used disk space |
| `proxmox.vm.disk.allocated.bytes` | bytes | gauge | Allocated disk space |
| `proxmox.vm.disk.read.bytes` | bytes | counter | Total bytes read |
| `proxmox.vm.disk.write.bytes` | bytes | counter | Total bytes written |
| `proxmox.vm.network.in.bytes` | bytes | counter | Bytes received |
| `proxmox.vm.network.out.bytes` | bytes | counter | Bytes transmitted |
| `proxmox.vm.uptime` | seconds | gauge | Virtual machine uptime |

### LXC Container Metrics

| Metric | Unit | Type | Description |
|--------|------|------|-------------|
| `proxmox.lxc.cpu.usage.percent` | percent | gauge | CPU usage percentage |
| `proxmox.lxc.memory.used.bytes` | bytes | gauge | Used memory |
| `proxmox.lxc.memory.allocated.bytes` | bytes | gauge | Allocated memory |
| `proxmox.lxc.memory.swap.bytes` | bytes | gauge | Swap memory |
| `proxmox.lxc.memory.usage.percent` | percent | gauge | Memory usage percentage |
| `proxmox.lxc.rootfs.used.bytes` | bytes | gauge | Used rootfs space |
| `proxmox.lxc.rootfs.quota.bytes` | bytes | gauge | Rootfs quota |
| `proxmox.lxc.network.in.bytes` | bytes | counter | Bytes received |
| `proxmox.lxc.network.out.bytes` | bytes | counter | Bytes transmitted |
| `proxmox.lxc.uptime` | seconds | gauge | Container uptime |

## Troubleshooting

### Authentication Errors
- Verify your username and password/token are correct
- Ensure the user has proper permissions
- Check that the API endpoint URL is correct

### SSL Certificate Errors
- If using self-signed certificates, either:
  - Enable "Disable SSL verification" (not recommended for production)
  - Provide the CA certificate fingerprint
  - Import the certificate into your system's CA store

### Missing Metrics
- Verify that the Proxmox VE API is accessible from the Elastic Agent
- Check agent logs for API connection errors
- Ensure the collection interval is not too short for your system

## Further Reading

- [Proxmox VE API Documentation](https://pve.proxmox.com/pve-docs/api-viewer/)
- [Elastic Fleet Documentation](https://www.elastic.co/guide/en/fleet/current/index.html)
