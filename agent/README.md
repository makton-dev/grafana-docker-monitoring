# Agent Folder - Grafana Alloy Monitoring Setup

This folder contains the configuration for running **Grafana Alloy**, a telemetry collection and forwarding agent that gathers metrics from your Linux host and Docker containers, then sends them to centralized Prometheus and Loki backends.

## Overview

The agent setup consists of two files:

- **`alloy.conf`** - Grafana Alloy configuration file defining what metrics and logs to collect
- **`docker-compose.yaml`** - Docker Compose file to run the Alloy agent container

Together, they provide an integrated monitoring solution that collects system metrics, Docker container metrics, and container logs from your host.

## Architecture

```
┌─────────────────────────────────────────┐
│         Linux Host Metrics              │
│  (CPU, Memory, Disk, Network, etc.)     │
└────────────┬────────────────────────────┘
             │
┌────────────▼────────────────────────────┐
│      Grafana Alloy Container            │
│  ┌─────────────────────────────────┐   │
│  │ • Node Metrics Exporter         │   │
│  │ • Docker Metrics (cAdvisor)     │   │
│  │ • Docker Logs Collector         │   │
│  │ • Log Filtering & Relabeling    │   │
│  └─────────────────────────────────┘   │
└────────────┬────────────────────────────┘
             │
      ┌──────┴──────┐
      │             │
    Prometheus    Loki
    (Metrics)   (Logs)
```

## Alloy Configuration (alloy.conf)

### Node Metrics Collection

```
prometheus.exporter.unix "node" {
  set_collectors = ["cpu", "meminfo", "diskstats", "filesystem", "netdev"]
}
```

**Purpose**: Collects Linux host-level metrics

**Metrics Collected**:
- **CPU** - CPU usage and timing information
- **Memory** - RAM usage, free memory, buffers, cache
- **Disk Stats** - Disk I/O operations and throughput
- **Filesystem** - Disk usage per mount point
- **Network** - Network interface statistics

**Scrape Interval**: 10 seconds

**Labels Applied**:
- `job = "integrations/linux"`
- `node = NODE_NAME` (from environment variable)

### Docker Metrics Collection

```
prometheus.exporter.cadvisor "docker" {
  docker_host      = "unix:///var/run/docker.sock"
  docker_only      = true
  enabled_metrics  = ["cpu", "memory", "disk", "network", "oom_event"]
}
```

**Purpose**: Collects container-level metrics using cAdvisor

**Metrics Collected**:
- **CPU** - Per-container CPU usage
- **Memory** - Per-container memory usage
- **Disk** - Per-container disk I/O
- **Network** - Per-container network traffic
- **OOM Events** - Out-of-memory events

**Scrape Interval**: 10 seconds

**Labels Applied**:
- `job = "integrations/docker"`
- `node = NODE_NAME` (from environment variable)

### Docker Logs Collection

```
loki.source.docker "default" {
  host = "unix:///var/run/docker.sock"
  targets = discovery.docker.logs.targets
}
```

**Purpose**: Collects logs from all running Docker containers

**Features**:
- Discovers all running containers
- Extracts container name from Docker metadata
- Adds `node` label to identify the source

### Log Filtering

The configuration includes two filtering rules:

1. **Drop Info Logs**: Removes all log entries with "info" level
   ```
   rule {
     action = "drop"
     regex  = "(?i).*info.*"
   }
   ```

2. **Drop Healthcheck Logs**: Removes logs containing "healthcheck"
   ```
   rule {
     action = "drop"
     regex  = "(?i).*healthcheck.*"
   }
   ```

**Purpose**: Reduces log volume by filtering out verbose, low-value logs

### Remote Endpoints

The configuration sends collected data to remote backends:

- **Prometheus Endpoint**: `PROM_ENDPOINT` environment variable
  - Default format: `http://{NODE_NAME}:9090/api/v1/write`
  - Receives metrics from both node and Docker exporters

- **Loki Endpoint**: `LOKI_ENDPOINT` environment variable
  - Default format: `http://{NODE_NAME}:3100/loki/api/v1/push`
  - Receives logs from Docker containers
  - Includes `TENANT_ID` for multi-tenant scenarios

## Docker Compose Configuration (docker-compose.yaml)

### Service: Alloy

```yaml
services:
  alloy:
    container_name: alloy
    image: grafana/alloy
    platform: linux/amd64
    privileged: true
    mem_limit: 500m
```

**Configuration Details**:

| Setting | Value | Purpose |
|---------|-------|---------|
| **Container Name** | `alloy` | Consistent container identification |
| **Image** | `grafana/alloy` | Official Grafana Alloy image |
| **Platform** | `linux/amd64` | Ensures x86_64 architecture |
| **Privileged** | `true` | Required for system-level metric access |
| **Memory Limit** | `500m` | Max 500MB RAM allocation |

### Environment Variables

```yaml
environment:
  - PROM_ENDPOINT=http://${NODE_NAME}:9090/api/v1/write
  - LOKI_ENDPOINT=http://${NODE_NAME}:3100/loki/api/v1/push
  - NODE_NAME=${NODE_NAME}
  - TENANT_ID=${TENANT_ID}
```

| Variable | Purpose |
|----------|---------|
| `PROM_ENDPOINT` | Prometheus remote write endpoint URL |
| `LOKI_ENDPOINT` | Loki log ingestion endpoint URL |
| `NODE_NAME` | Identifier for this host (used in labels) |
| `TENANT_ID` | Tenant identifier for multi-tenant Loki |

**Note**: These variables must be set in your environment or `.env` file before running the container.

### Ports

```yaml
ports:
  - 12345:12345
```

**Purpose**: Exposes Alloy's HTTP server for health checks, debugging, and metrics exposure

**Access**: `http://localhost:12345`

### Volumes

| Volume | Mount Point | Permissions | Purpose |
|--------|------------|-------------|---------|
| `alloy.conf` | `/etc/alloy/config.alloy` | Read-only | Configuration file |
| Docker socket | `/var/run/docker.sock` | Read-only | Access to Docker daemon |
| `/sys` | `/sys` | Read-only | System metrics |
| Root filesystem | `/rootfs` | Read-only | Host filesystem access |
| Disk devices | `/dev/disk` | Read-only | Disk information |
| Docker libraries | `/var/lib/docker` | Read-only | Docker container info |

### Command

```bash
run --server.http.listen-addr=0.0.0.0:12345 --storage.path=/var/lib/alloy/data /etc/alloy/config.alloy
```

**Flags**:
- `--server.http.listen-addr=0.0.0.0:12345` - Listen on all interfaces, port 12345
- `--storage.path=/var/lib/alloy/data` - Storage directory for Alloy data
- `/etc/alloy/config.alloy` - Path to the configuration file

## Prerequisites

- Docker and Docker Compose installed
- Network access to Prometheus instance (at `PROM_ENDPOINT`)
- Network access to Loki instance (at `LOKI_ENDPOINT`)
- Docker daemon running on the host

## Setup Instructions

### 1. Environment Variables

Create a `.env` file in the agent folder or set these environment variables:

```bash
export NODE_NAME=my-server-1
export TENANT_ID=my-tenant
```

**Example .env file**:
```
NODE_NAME=production-server-01
TENANT_ID=production
```

### 2. Running the Container

```bash
# Navigate to the agent folder
cd agent

# Start the Alloy container
docker-compose up -d

# Verify it's running
docker-compose ps
```

### 3. Checking Status

```bash
# View logs
docker-compose logs -f alloy

# Check Alloy health/metrics endpoint
curl http://localhost:12345/metrics
```

### 4. Stopping the Container

```bash
docker-compose down
```

## Integration with Server Stack

This agent is designed to work with the server stack defined in `/server/docker-compose.yaml`, which typically includes:

- **Prometheus** (port 9090) - Metrics storage and querying
- **Loki** (port 3100) - Log storage and querying
- **Grafana** (port 443) - Visualization dashboard

The Alloy agent sends data to these services, enabling:

- Real-time system monitoring dashboards
- Container performance visualization
- Log aggregation and analysis
- Alerting based on metrics and logs

## Troubleshooting

### Container won't start

**Error**: `connect: permission denied`

**Solution**: Ensure the container has proper permissions to access Docker socket:

```bash
# Run with proper permissions
docker-compose up -d

# Or check Docker socket permissions
ls -la /var/run/docker.sock
```

### No metrics appearing

**Problem**: Metrics not showing in Prometheus

**Debugging**:
1. Verify endpoints are reachable:
   ```bash
   docker-compose exec alloy curl http://${NODE_NAME}:9090/api/v1/write
   ```

2. Check Alloy logs for errors:
   ```bash
   docker-compose logs alloy
   ```

3. Verify environment variables are set:
   ```bash
   docker-compose exec alloy env | grep PROM
   ```

### Docker metrics not collected

**Problem**: Container metrics missing

**Solution**:
1. Ensure Docker socket is properly mounted: `docker-compose logs alloy`
2. Verify the container runs with `privileged: true`
3. Check Docker daemon is running: `docker ps`

### Memory issues

**Problem**: Container using too much memory

**Solution**: The memory limit is set to 500MB. Adjust if needed:

```yaml
mem_limit: 1g  # Increase to 1GB
```

## Configuration Customization

### Changing Scrape Intervals

To collect metrics more or less frequently, edit `alloy.conf`:

```
prometheus.scrape "node" {
  scrape_interval = "30s"  # Change from 10s to 30s
}
```

### Adjusting Log Filters

To keep info-level logs, comment out or remove that rule in `alloy.conf`:

```
// rule {
//   action = "drop"
//   regex  = "(?i).*info.*"
// }
```

### Adding More Metrics

Extend the collectors in `alloy.conf`:

```
set_collectors = ["cpu", "meminfo", "diskstats", "filesystem", "netdev", "thermal"]
```

## Additional Resources

- [Grafana Alloy Documentation](https://grafana.com/docs/alloy/)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Loki Documentation](https://grafana.com/docs/loki/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)

## Health Checks

Monitor Alloy health at:

```
http://localhost:12345/-/healthy
```

View internal metrics at:

```
http://localhost:12345/metrics
```

View Alloy configuration status at:

```
http://localhost:12345/api/v1/logs/ready
```

## License

Configuration provided as-is for use with Grafana Alloy.
