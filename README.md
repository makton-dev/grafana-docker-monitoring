# grafana-docker-monitoring
The purpose of this repo is to maintain a central location for managing the docker configs for a fully operational On-Prem Grafana Log and Metric NMS/APM.

## How this came to be
I been running an 8 node K3S Rasberry Pie 4 cluster for years and one of the biggest issues was the about the ram and CPU being used when no apps were even installed. I've also started working with Amazon ECS as it's cheaper to run than EKS. However, I need a way to monitor all the apps I'm running and ensure all 8 nodes are healthy while having a small footprint.

To make this really flexible, I create a "server" config and an "agent" config, as well as a swarm config. The agent is installed on every docker host as a stand alone service and will have access to host for host metrics.

## Included with server
The server includes the following:
* Grafana Dashboard
* Loki Logging data source
* Prometheus Metric data source

The setup is based on the docker compose from Grafana with modification for standardization and ease of use as I found many issues using the defaut conpose file from Grafana.

## Included with client
The following is included with the client:
* Alloy

Alloy pulls from the logs and the metric exporters and sends to the Loki and Prometheus data sources. Yes, this is a "push-only" setup.

## Swarm deployment
The `swarm-compose.yaml` file uses the configs from the single node server install in `/server/` and deploys across the swarm. The agent remains seperate as it need elevated privilieges for the node metrics which is not alowed in a docker swarm service.
# Centralized Monitoring and Logging Stack

A complete, production-ready monitoring and observability solution that aggregates metrics and logs from multiple hosts and containerized environments into a centralized Grafana dashboard.

## Project Overview

This project implements a distributed monitoring architecture consisting of two main components:

1. **Agent Nodes** - Lightweight collectors deployed on each host to gather metrics and logs
2. **Server Stack** - Centralized backend for storage, aggregation, and visualization

The architecture enables:
- **Real-time monitoring** of system health across multiple nodes
- **Centralized logging** from all Docker containers
- **Unified dashboards** across your infrastructure
- **Horizontal scalability** to monitor hundreds of hosts
- **Long-term retention** of metrics and logs with S3-compatible storage


## Folder Structure

```
/
├── README.md                    # This file - Project overview
├── LICENSE                      # Project license
├── env.sh                       # Environment variable configuration
├── swarm-compose.yaml           # Docker Swarm orchestration (multi-host)
│
├── agent/                       # Agent component (deploy on each host)
│   ├── README.md                # Agent documentation
│   ├── docker-compose.yaml      # Standalone deployment config
│   ├── alloy.conf               # Grafana Alloy configuration
│   └── ...
│
└── server/                      # Server stack (deploy once, centralized)
    ├── README.md                # Server documentation
    ├── docker-compose.yaml      # Standalone deployment config
    ├── prometheus.yml           # Prometheus configuration
    ├── grafana.ini              # Grafana settings
    ├── grafana-datasources.yaml # Data source provisioning
    ├── loki-config.yaml         # Loki distributed config
    ├── grafana-nginx.conf       # Grafana reverse proxy
    ├── loki-nginx.conf          # Loki routing proxy
    ├── alloy.blackbox           # Blackbox monitoring config
    ├── alloy.proxmox            # Proxmox telemetry config
    └── nginx_certs/             # SSL/TLS certificates
        └── README.md            # Certificate generation guide
```

## Quick Start

### Prerequisites

- Docker and Docker Compose installed
- Network connectivity between agent nodes and server
- SSL/TLS certificates for HTTPS (see `server/nginx_certs/README.md`)

### 1. Configure Environment Variables

Edit `env.sh` with your infrastructure details:

```bash
nano env.sh
```

Required variables:

```bash
export PROM_ENDPOINT=http://monitoring-server:9090/api/v1/write
export LOKI_ENDPOINT=http://monitoring-server:3100/loki/api/v1/push
export NODE_NAME=my-server-1
export TENANT_ID=production
```

**Source the environment**:

```bash
source env.sh
```

### 2. Deploy Server Stack (On Central Monitoring Server)

```bash
cd server

# Start all services
docker-compose up -d

# Verify services are running
docker-compose ps

# Access Grafana
https://grafana.domain.com
```

**See**: [server/README.md](server/README.md) for detailed server configuration

### 3. Deploy Agent (On Each Host to Monitor)

```bash
cd agent

# Update NODE_NAME for this host
export NODE_NAME=production-server-01

# Start the agent
docker-compose up -d

# Verify agent is running
docker-compose ps
```

**See**: [agent/README.md](agent/README.md) for detailed agent configuration

### 4. Verify Connectivity

```bash
# Check if metrics are appearing in Prometheus
curl http://prometheus-server:9090/api/v1/query?query=node_cpu_seconds_total

# Check if logs are appearing in Loki
curl http://loki-server:3100/api/prom/query?query={job=\"integrations/docker\"}
```

## Deployment Options

### Option 1: Standalone (Docker Compose) - Development/Testing

**Server**:
```bash
cd server
docker-compose up -d
```

**Agents** (run on each host):
```bash
cd agent
docker-compose up -d
```

**Pros**: Simple, lightweight, easy testing  
**Cons**: Requires manual coordination across hosts

### Option 2: Docker Swarm (Production Recommended)

**Initialize Swarm** (on first manager node):
```bash
docker swarm init
```

**Join other nodes** (on worker nodes):
```bash
docker swarm join --token <token> <manager-ip>:2377
```

**Deploy Stack**:
```bash
docker stack deploy -c swarm-compose.yaml monitoring
```

**See logs**:
```bash
docker service logs monitoring_loki-backend
```

**Pros**: Automated orchestration, high availability, distributed  
**Cons**: More complex, requires Swarm knowledge

## Component Details

### Agent (Deployed on Each Host)

**Purpose**: Collect and forward metrics and logs from individual hosts

**What it monitors**:
- Linux system metrics (CPU, memory, disk, network)
- Docker container metrics and logs
- Custom application metrics (via OpenTelemetry)
- Blackbox/endpoint monitoring (optional)
- Proxmox cluster metrics (optional)

**How it works**:
1. Discovers all running Docker containers
2. Collects system and container metrics every 10 seconds
3. Filters out verbose logs (info-level, healthchecks)
4. Sends metrics to Prometheus remote write API
5. Sends logs to Loki API

**Configuration**: See [agent/README.md](agent/README.md)

### Server Stack (Deployed Once, Centralized)

**Components**:

| Service | Purpose | Port |
|---------|---------|------|
| **Grafana** | Dashboards & visualization | 443 (HTTPS) |
| **Prometheus** | Metrics database | 9090 |
| **Loki Write** | Log ingestion | 3102 |
| **Loki Read** | Log querying | 3101 |
| **Loki Backend** | Log compaction & retention | 3100 |
| **Loki Proxy** | Route read/write requests | 3100 |
| **MinIO** | S3-compatible storage | 9000 |
| **Nginx** | Reverse proxy for Grafana | 80/443 |

**How it works**:
1. Prometheus scrapes or receives metrics from agents
2. Loki receives logs from agents and stores in MinIO S3
3. Grafana connects to both Prometheus and Loki
4. Nginx provides HTTPS access to Grafana
5. Data is persisted in volumes (can be backed by GlusterFS)

**Configuration**: See [server/README.md](server/README.md)

## Environment Variables

Set these in `env.sh` before deployment:

```bash
# Prometheus endpoint - where Prometheus server is accessible
PROM_ENDPOINT=http://[prometheus-host]:9090/api/v1/write

# Loki endpoint - where Loki write service is accessible
LOKI_ENDPOINT=http://[loki-host]:3100/loki/api/v1/push

# Node name - unique identifier for this host
NODE_NAME=my-server-1

# Tenant ID - for multi-tenant scenarios (Grafana Loki)
TENANT_ID=production
```

## Integration Points

### Receiving Metrics

Agents send metrics via Prometheus remote write API:

```
Agent → Prometheus (http://PROM_ENDPOINT:9090/api/v1/write)
```

Metrics collected:
- Node (Linux) metrics: CPU, memory, disk, network
- Docker metrics: Container CPU, memory, disk, network
- Custom metrics: Via OpenTelemetry (Proxmox, applications)

### Receiving Logs

Agents send logs via Loki API:

```
Agent → Loki (http://LOKI_ENDPOINT:3100/loki/api/v1/push)
```

Logs collected:
- Docker container logs from all running containers
- Filtered to remove low-value logs (info-level, healthchecks)
- Labeled with container name, host, and tenant ID

### Querying in Grafana

Both metrics and logs available in Grafana:

**Metrics queries** (PromQL):
```
rate(node_cpu_seconds_total[5m])
container_memory_usage_bytes
```

**Log queries** (LogQL):
```
{job="integrations/docker", node="my-server-1"}
{container_name=~"app-.*"} | json
```

## Storage Architecture

### Volumes

| Volume | Service | Path | Purpose |
|--------|---------|------|---------|
| `grafana-storage` | Grafana | `/var/lib/grafana` | Dashboards, plugins, users |
| `prometheus-storage` | Prometheus | `/prometheus` | Time-series metrics data |
| `loki-s3` | MinIO | `/data` | Logs and metadata (S3) |

### Persistence Options

**Development**: Docker named volumes (stored on local disk)

**Production**: Mount to persistent storage:
- GlusterFS (as shown in `swarm-compose.yaml`)
- NFS volumes
- Cloud block storage (EBS, GCP Persistent Disks, etc.)

### Data Retention

**Prometheus**: 
- Default: 15 days (configurable in `prometheus.yml`)
- Adjust `--storage.tsdb.retention.time` for longer retention

**Loki**:
- Default: Indefinite (controlled by compactor)
- Set retention in `loki-config.yaml` under `compactor.retention_enabled`
- Recommendations: 30-90 days for production

## Scaling Considerations

### Horizontal Scaling

**Adding more agents**:
1. Update Node names to be unique
2. Run agent docker-compose on each new host
3. Agents automatically register with centralized services

**Expected capacity**:
- Single Prometheus instance: ~100-200 hosts
- Single Loki instance: Depends on log volume (typically 500+ hosts)

### Vertical Scaling

Increase resource limits in docker-compose files:

**Server resources** (`server/docker-compose.yaml`):
```yaml
deploy:
  resources:
    limits:
      cpus: '2.0'        # Increase cores
      memory: 1g         # Increase memory
```

**Agent resources** (`agent/docker-compose.yaml`):
```yaml
mem_limit: 1g           # Increase from 500m
```

## Monitoring the Monitoring Stack

The stack self-monitors! View internal health:

- **Grafana health**: `https://grafana.domain.com/api/health`
- **Prometheus health**: `http://prometheus:9090/-/healthy`
- **Loki health**: `docker-compose exec loki-backend loki -health`
- **MinIO health**: `docker-compose exec minio_s3 mc admin info local`

## Troubleshooting

### No metrics appearing

1. **Verify agent is running**: `docker ps | grep alloy`
2. **Check connectivity**: `curl $PROM_ENDPOINT` from agent host
3. **Check agent logs**: `docker logs alloy`
4. **Verify environment variables**: `env | grep PROM`

### No logs appearing

1. **Verify Loki endpoint is reachable**: `curl $LOKI_ENDPOINT/api/prom/push`
2. **Check Loki logs**: `docker-compose logs loki-write`
3. **Verify containers are running**: `docker ps`
4. **Check log filtering rules** in `agent/alloy.conf`

### High memory usage

1. **Check which service**: `docker stats`
2. **Reduce Prometheus retention**: Edit `server/prometheus.yml`
3. **Increase allocated memory**: Edit resource limits
4. **Optimize Nginx cache**: Edit `server/grafana-nginx.conf`

### Certificate errors

1. **Generate certificates**: See [server/nginx_certs/README.md](server/nginx_certs/README.md)
2. **Place in correct path**: `server/nginx_certs/server.{crt,key}`
3. **Update Nginx config**: Edit `server/grafana-nginx.conf`

See individual component READMEs for detailed troubleshooting:
- [agent/README.md](agent/README.md)
- [server/README.md](server/README.md)
- [server/nginx_certs/README.md](server/nginx_certs/README.md)

## Configuration Customization

### Change Scrape Interval

Edit `server/prometheus.yml`:
```yaml
global:
  scrape_interval: 30s        # Default: 15s
```

And `agent/alloy.conf`:
```
prometheus.scrape "node" {
  scrape_interval = "30s"     # Default: 10s
}
```

### Change Log Retention

Edit `server/loki-config.yaml`:
```yaml
compactor:
  retention_delete_delay: 24h  # Increase retention
  retention_enabled: true
```

### Add Custom Prometheus Targets

Edit `server/prometheus.yml`:
```yaml
scrape_configs:
  - job_name: 'my-application'
    static_configs:
      - targets: ['app-server:8080']
```

### Enable/Disable Log Filtering

Edit `agent/alloy.conf` - comment out rules in the `loki.relabel` section:

```
// rule {
//   action = "drop"
//   regex  = "(?i).*healthcheck.*"
// }
```

## Future Enhancements

### Planned Features

- [ ] **Garage replacement**: Replace MinIO with Garage for better distributed storage
- [ ] **Alerting**: Configure alert rules and notifications
- [ ] **Service discovery**: Automatic agent registration
- [ ] **Multi-region**: Support for distributed deployments across regions
- [ ] **HA Prometheus**: High-availability setup with Thanos
- [ ] **Kubernetes integration**: Native K8s monitoring
- [ ] **Custom dashboards**: Pre-built dashboard templates
- [ ] **LDAP/OAuth**: Enterprise authentication

### Known Limitations

- MinIO community edition has limited support (planned Garage replacement)
- Single Grafana instance (can be HA with external database)
- Manual certificate management (could use cert-manager)
- No built-in alerting (can integrate with AlertManager)

## Best Practices

### Security

- ✅ Use HTTPS for all external access (certificates in `nginx_certs/`)
- ✅ Change default MinIO credentials in `server/loki-config.yaml`
- ✅ Enable Grafana authentication in `server/grafana.ini`
- ✅ Restrict network access to services
- ✅ Rotate credentials and API keys regularly
- ❌ Don't expose Prometheus or MinIO ports to public internet

### Operations

- ✅ Monitor the monitoring stack itself
- ✅ Set up log rotation for docker logs
- ✅ Implement backup procedures for volumes
- ✅ Use consistent Node names across deploys
- ✅ Version your configuration files in Git
- ✅ Test disaster recovery procedures
- ❌ Don't run in privileged mode unless necessary

### Performance

- ✅ Use persistent volumes backed by fast storage
- ✅ Tune retention policies based on requirements
- ✅ Monitor resource usage and adjust limits
- ✅ Use the cache in Nginx for Grafana
- ✅ Consider load balancing for high traffic
- ❌ Don't set unreasonably long retention periods

## Support and Resources

### Documentation

- [Grafana Official Docs](https://grafana.com/docs/)
- [Prometheus Official Docs](https://prometheus.io/docs/)
- [Loki Official Docs](https://grafana.com/docs/loki/)
- [Alloy Official Docs](https://grafana.com/docs/alloy/)
- [Docker Compose Docs](https://docs.docker.com/compose/)
- [Docker Swarm Docs](https://docs.docker.com/engine/swarm/)

### Community

- [Grafana Community](https://community.grafana.com/)
- [Prometheus Community](https://prometheus.io/community/)
- GitHub Issues and Discussions

## License

This project is provided as-is for use with open-source monitoring tools. See LICENSE file for details.

## Contributing

Contributions are welcome! Please:

1. Test changes thoroughly
2. Update documentation
3. Follow existing configuration patterns
4. Submit pull requests with clear descriptions

## Project Status

✅ **Production Ready** - Used in production environments  
✅ **Active Development** - Regularly updated and improved  
✅ **Stable API** - Configuration format stable  
⚠️ **Breaking Changes** - Check CHANGELOG before upgrades  

## Quick Reference

### Common Commands

```bash
# Source environment
source env.sh

# Start server stack
cd server && docker-compose up -d

# Start agent
cd agent && docker-compose up -d

# View logs
docker-compose logs -f [service]

# View running services
docker-compose ps

# Stop services
docker-compose down

# Restart services
docker-compose restart

# View health status
docker-compose ps

# Access Grafana
https://grafana.domain.com
```

### Important Files

- **Configuration**: `env.sh`, `server/prometheus.yml`, `agent/alloy.conf`
- **Credentials**: `server/loki-config.yaml` (MinIO keys)
- **Certificates**: `server/nginx_certs/server.{crt,key}`
- **Compose files**: `docker-compose.yaml` (standalone), `swarm-compose.yaml` (swarm)

---

**For component-specific documentation, see**:
- 🔧 [Agent Configuration](agent/README.md)
- 📊 [Server Stack Setup](server/README.md)
- 🔐 [Certificate Generation](server/nginx_certs/README.md)
