# Server Folder - Monitoring Stack

This folder contains the complete configuration for a centralized monitoring and logging stack built with Grafana, Prometheus, and Loki. It provides metrics collection, log aggregation, visualization, and long-term storage capabilities.

## Overview

The server stack is composed of several microservices orchestrated via Docker Compose:

```
┌──────────────────────────────────────────────────────────┐
│                     Grafana (Port 443)                   │
│              Dashboards & Visualization                  │
└─────────────────────┬──────────────────────┬─────────────┘
                      │                      │
          ┌───────────▼──────┐      ┌───────▼────────────┐
          │    Prometheus    │      │      Loki Proxy    │
          │   (Metrics DB)   │      │   (Log Router)     │
          │   (Port 9090)    │      │   (Port 3100)      │
          └──────────────────┘      └─┬──────────────┬───┘
                                      │              │
                          ┌───────────▼─┐    ┌───────▼────────┐
                          │ Loki Write  │    │  Loki Read     │
                          │(Log Ingestion)  (Log Retrieval)
                          └──────────────┘    └────────────────┘
                                      │              │
                          ┌───────────▼──────────────▼─┐
                          │     Garage S3 Storage     │
                          │    (Log & Data Store)     │
                          └────────────────────────────┘
```

## File Structure

| File | Purpose |
|------|---------|
| `docker-compose.yaml` | Orchestration file defining all services |
| `prometheus.yml` | Prometheus configuration for metrics scraping |
| `grafana.ini` | Grafana server configuration |
| `grafana-datasources.yaml` | Grafana data source definitions (Prometheus & Loki) |
| `grafana-nginx.conf` | Nginx reverse proxy for Grafana with HTTPS |
| `loki-config.yaml` | Loki distributed logging configuration |
| `loki-nginx.conf` | Nginx router for Loki read/write endpoints |
| `alloy.blackbox` | Alloy config for blackbox monitoring (endpoints/sites) |
| `alloy.proxmox` | Alloy config for Proxmox OpenTelemetry integration |
| `nginx_certs/` | SSL/TLS certificates for HTTPS |

## Services Overview

### Grafana
**Container**: `grafana`  
**Port**: 443 (HTTPS via Nginx)  
**Image**: `grafana/grafana`

The main visualization and dashboard platform. Grafana connects to Prometheus for metrics and Loki for logs.

**Key Features**:
- Interactive dashboards
- Alert management
- User authentication
- Multi-datasource support
- Templating and variables

**Configuration**: `grafana.ini`
- Domain: `grafana.domain.com`
- Root URL: `https://grafana.domain.com`
- HTTPS enabled with certificates from `nginx_certs/`

### Prometheus
**Container**: `prometheus`  
**Port**: 9090 (Internal)  
**Image**: `prom/prometheus`

Time-series database for metrics storage and querying.

**Key Features**:
- Metrics scraping and storage
- PromQL query language
- Data retention and aggregation
- Alert rule evaluation

**Configuration**: `prometheus.yml`
- Global scrape interval: 15 seconds
- Scrape configs: Empty (populated from agents via service discovery)

**Storage**: Named volume `prometheus-storage`

### Loki - Backend
**Container**: `loki_backend`  
**Image**: `grafana/loki`  
**Ports**: 3100, 7946

Handles log compaction, retention, and cluster coordination.

**Responsibilities**:
- Compacting logs (every 10 minutes)
- Managing retention policies
- Cluster membership management
- Ruler storage

**Resource Limits**: 1 CPU, 256MB RAM

### Loki - Write
**Container**: `loki_write`  
**Port**: 3102 → 3100 (Internal)  
**Image**: `grafana/loki`

Dedicated service for log ingestion.

**Responsibilities**:
- Receiving logs from agents
- Writing to S3 storage (Garage)
- Replication across cluster

**Configuration**: Multi-target Loki instance  
**Resource Limits**: 1 CPU, 512MB RAM

### Loki - Read
**Container**: `loki_read`  
**Port**: 3101 → 3100 (Internal)  
**Image**: `grafana/loki`

Dedicated service for log retrieval.

**Responsibilities**:
- Querying logs
- Supporting log tailing (streaming)
- Read-side optimizations

**Configuration**: Multi-target Loki instance  
**Resource Limits**: 1 CPU, 256MB RAM

### Loki Proxy (Nginx)
**Container**: `loki_proxy`  
**Port**: 3100  
**Image**: `nginx`

Routes requests to appropriate Loki services based on API endpoints.

**Routing Rules** (`loki-nginx.conf`):
- `/api/prom/push` → `loki-write` (log ingestion)
- `/api/prom/tail` → `loki-read` (log streaming)
- `/api/prom/*` → `loki-read` (legacy API queries)
- `/loki/api/v1/push` → `loki-write` (v1 ingestion)
- `/loki/api/v1/tail` → `loki-read` (v1 streaming)
- `/loki/api/*` → `loki-read` (v1 queries)

**Resource Limits**: 0.5 CPU, 52MB RAM

### Grafana Nginx Proxy
**Container**: `grafana-nginx`  
**Ports**: 80 (redirect), 443 (HTTPS)  
**Image**: `nginx`

Reverse proxy for Grafana providing HTTPS termination and caching.

**Features** (`grafana-nginx.conf`):
- HTTP → HTTPS redirect (port 80 → 443)
- SSL/TLS termination with certificates from `nginx_certs/`
- Response caching (5GB cache, 60-minute expiration)
- Proxy headers for real IP tracking
- Cache hit for 200/302 responses (10 minutes)

### Garage (S3 Storage)
**Container**: `garage_s3`
**Image**: `dxflrs/garage:v2.3.0`

S3-compatible object storage for Loki logs and metadata.

**Endpoint**: `garage:3900` (insecure=true, path-style requests)

**Bucket**: `loki-data` - Loki data and ruler state

**Storage**: Named volume `garage-storage`, mounted at `/var/lib/garage`

**Configuration**: `garage.toml`; credentials can be supplied through `GARAGE_RPC_SECRET`, `GARAGE_DEFAULT_ACCESS_KEY`, and `GARAGE_DEFAULT_SECRET_KEY`.

The Compose setup uses Garage's single-node mode, which has no data redundancy. For a production Garage cluster, deploy multiple Garage nodes using Garage's multi-node deployment guidance.

#### Migrating from MinIO

Garage cannot use MinIO's on-disk data format. The Compose files use a separate `garage-storage` volume (`/mnt/gluster/garage` in Swarm), leaving the existing MinIO volume (`loki` or `/mnt/gluster/loki`) untouched. The old `loki-data` and `loki-ruler` buckets are not copied automatically; the new config uses one `loki-data` bucket for both, so plan an S3-level migration of both buckets into a compatible layout before cutover. Keep a backup of the original volume until the migrated data is verified.

## Configuration Files

### prometheus.yml

```yaml
global:
  scrape_interval: 15s

scrape_configs: []
```

- **scrape_interval**: Frequency of metric collection (15 seconds)
- **scrape_configs**: Currently empty - populated by agent service discovery or manual configuration

**To add Prometheus scrape targets**, add entries to `scrape_configs`:

```yaml
scrape_configs:
  - job_name: 'linux-hosts'
    static_configs:
      - targets: ['localhost:9100']
  
  - job_name: 'docker'
    static_configs:
      - targets: ['localhost:8080']
```

### grafana.ini

**Key Configuration**:

```ini
[server]
domain = grafana.domain.com
root_url = https://grafana.domain.com
```

**Customizable Settings**:
- `http_port` - Default: 3000 (overridden by Nginx to 443)
- `http_addr` - Default: all interfaces
- `enable_gzip` - Response compression
- `log_level` - Logging verbosity
- `admin_user` / `admin_password` - Initial credentials

### loki-config.yaml

**Key Sections**:

```yaml
server:
  http_listen_address: 0.0.0.0
  http_listen_port: 3100

schema_config:
  configs:
    - from: 2023-01-01
      store: tsdb
      object_store: s3
      schema: v13

common:
  path_prefix: /loki
  replication_factor: 1
  compactor_address: http://loki-backend:3100
  storage:
    s3:
      endpoint: garage:3900
      region: garage
      bucketnames: loki-data
```

**Features**:
- **Multi-target mode**: Separate read, write, and backend processes
- **S3 Storage**: Garage backend for data persistence
- **Retention**: Enabled with 2-hour delete delay
- **Compaction**: Every 10 minutes
- **Cluster Mode**: Memberlist-based gossip protocol

### grafana-datasources.yaml

Automatically provisioned Grafana data sources:

1. **Loki Datasource**
   - Type: `loki`
   - URL: `http://loki-proxy:3100`
   - Tenant ID: `${TENANT_ID}` (from environment)
   - Used for log queries and visualization

2. **Prometheus Datasource**
   - Type: `prometheus`
   - URL: `http://prometheus:9090`
   - Default datasource for metrics
   - Query method: POST
   - Cache level: High
   - Prometheus version: 3.3.0

### grafana-nginx.conf

Nginx reverse proxy configuration:

**SSL Configuration**:
```nginx
ssl_certificate /opt/certificates/server.crt;
ssl_certificate_key /opt/certificates/server.key;
```

**Caching**:
- Cache zone: 10MB keys, 5GB max size
- Cache valid: 10 minutes (200/302), 1 minute (404)
- Stale cache serving on backend errors

**Headers**:
- Preserves real client IP
- Forwards protocol scheme
- Host header passthrough

### loki-nginx.conf

Dedicated router for Loki endpoints:

**Write Endpoints** (→ `loki-write:3100`):
- `/api/prom/push`
- `/loki/api/v1/push`

**Read Endpoints** (→ `loki-read:3100`):
- `/api/prom/tail` (with WebSocket upgrade)
- `/api/prom/*` (legacy API)
- `/loki/api/v1/tail` (with WebSocket upgrade)
- `/loki/api/*` (v1 API)

**Resolver**: Uses Docker's internal DNS (127.0.0.11)

### alloy.blackbox

**Purpose**: Monitoring external endpoints and websites

```
// This is for monitoring ports and sites
```

Currently a placeholder for blackbox monitoring configuration.

**Typical use cases**:
- Endpoint availability monitoring
- SSL certificate expiration checks
- HTTP response time monitoring
- Port reachability checks

### alloy.proxmox

**Purpose**: OpenTelemetry receiver for Proxmox cluster metrics

```
otelcol.receiver.otlp "proxmox" {
  http {
    endpoint = "0.0.0.0:4318"
  }
}
```

**Features**:
- Receives OpenTelemetry metrics on port 4318
- Exports to Prometheus via remote write
- Integrates Proxmox cluster monitoring into centralized Prometheus

**Configuration**:
- Receiver: OTLP/HTTP on 0.0.0.0:4318
- Exporter: Prometheus remote write to Prometheus server

## Docker Compose Network

**Network**: `grafana`  
**Scope**: All services communicate over this internal Docker network

**Service DNS Names**:
- `grafana` - Grafana service
- `prometheus` - Prometheus service
- `loki-proxy` - Loki proxy router
- `loki-read` - Loki read instance
- `loki-write` - Loki write instance
- `loki-backend` - Loki backend instance
- `garage` - Garage S3 storage

## Volumes

| Volume | Purpose | Retention |
|--------|---------|-----------|
| `grafana-storage` | Grafana databases, plugins, dashboards | Persistent |
| `prometheus-storage` | Prometheus time-series data | Persistent |
| `garage-storage` | Garage metadata and objects | Persistent |

## Environment Variables

Configure these before running `docker-compose up`:

```bash
export TENANT_ID=my-tenant
export NODE_NAME=my-server
```

Used by:
- Grafana datasources (Loki tenant ID)
- Log labels and filtering

## Startup Instructions

### Prerequisites

- Docker and Docker Compose installed
- SSL/TLS certificates generated in `nginx_certs/` folder
- Sufficient disk space for log storage (Garage volume)

### Starting the Stack

```bash
# Navigate to server directory
cd server

# Set environment variables
export TENANT_ID=production
export NODE_NAME=server-01
export GARAGE_RPC_SECRET="$(openssl rand -hex 32)"
export GARAGE_DEFAULT_ACCESS_KEY="GK$(openssl rand -hex 16)"
export GARAGE_DEFAULT_SECRET_KEY="$(openssl rand -hex 32)"

# Start all services
docker-compose up -d

# Verify all services are running
docker-compose ps
```

### Accessing Services

| Service | URL | Purpose |
|---------|-----|---------|
| Grafana | `https://grafana.domain.com` | Dashboards & visualization |
| Prometheus | `http://localhost:9090` | Metrics queries (internal only) |
| Loki | `http://localhost:3100` | Logs API |
| Garage S3 | Internal `http://garage:3900` | S3-compatible object storage (not published to host) |

### Stopping the Stack

```bash
docker-compose down

# To also remove volumes (WARNING: deletes all data)
docker-compose down -v
```

## Health Checks

Each service includes automated health checks:

```bash
# View health status
docker-compose ps

# Check Loki health
docker-compose exec loki-backend loki -health

# Check Prometheus
curl http://localhost:9090/-/healthy

# Check Grafana
curl -k https://grafana.domain.com/api/health
```

## Integration Points

### Receiving Metrics

Prometheus scrapes metrics from configured targets. Connect agents/exporters via:

1. **Service Discovery**: Add scrape configs in `prometheus.yml`
2. **Remote Write**: Agents send metrics to Prometheus remote write endpoint
3. **Alloy Proxmox**: Send OpenTelemetry metrics to port 4318

### Receiving Logs

Loki accepts logs via:

1. **Loki Agent**: Uses Loki write endpoint (port 3100)
2. **Alloy Agent**: Sends Docker container logs
3. **Promtail**: Forward to `http://loki-proxy:3100/loki/api/v1/push`

### Querying Data

Access both metrics and logs in Grafana:

- **Metrics**: Use Prometheus datasource
- **Logs**: Use Loki datasource
- **Combined**: Log volume correlation with metrics

## Customization

### Adding More Prometheus Targets

Edit `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: 'my-app'
    static_configs:
      - targets: ['my-app:8080']
```

### Adjusting Loki Retention

Edit `loki-config.yaml`:

```yaml
compactor:
  retention_delete_delay: 24h  # Increase from 2h
  retention_enabled: true
```

### Changing Grafana Domain

Edit `grafana.ini`:

```ini
domain = mynewdomain.com
root_url = https://mynewdomain.com
```

### Increasing Resource Limits

Edit `docker-compose.yaml` service limits:

```yaml
deploy:
  resources:
    limits:
      cpus: '2.0'
      memory: 1g
```

## Troubleshooting

### Services won't start

```bash
# Check logs
docker-compose logs -f

# Check specific service
docker-compose logs grafana

# Ensure volumes exist
docker volume ls | grep grafana
```

### Grafana won't load

- Verify SSL certificates exist in `nginx_certs/`
- Check Nginx logs: `docker-compose logs grafana-nginx`
- Verify domain in `grafana.ini` matches your access URL

### No logs appearing in Loki

- Verify agents are sending to port 3100
- Check Loki proxy logs: `docker-compose logs loki-proxy`
- Verify `TENANT_ID` environment variable is set
- Check loki-write health: `docker-compose logs loki-write`

### Prometheus scraping fails

- Verify targets are reachable from Prometheus container
- Check Prometheus logs: `docker-compose logs prometheus`
- Test from container: `docker-compose exec prometheus curl http://target:port/metrics`

### Garage connectivity issues

- Verify Garage is running: `docker-compose ps garage`
- Check Garage logs: `docker-compose logs garage`
- Confirm the same Garage access key and secret are set for Garage and Loki

### High memory usage

- Check which service: `docker stats`
- Reduce cache in Nginx: Edit `grafana-nginx.conf`
- Limit Prometheus retention: Edit `prometheus.yml`

## Performance Tuning

### Optimize Prometheus

```yaml
global:
  scrape_interval: 30s        # Increase for less data
  evaluation_interval: 30s
```

### Optimize Loki

```yaml
compactor:
  compaction_interval: 30m     # Reduce frequency
  retention_delete_delay: 12h  # Increase delay
```

### Optimize Nginx Caching

```nginx
proxy_cache_valid 200 302 1h;  # Increase cache duration
```

## Security Considerations

⚠️ **Important Security Notes**:

1. **Garage Credentials**: Set unique `GARAGE_RPC_SECRET`, `GARAGE_DEFAULT_ACCESS_KEY`, and `GARAGE_DEFAULT_SECRET_KEY` values before deployment. Use the same access key and secret for Garage and Loki. The Compose defaults are for local testing only.

2. **SSL Certificates**: Replace self-signed certs with valid ones for production
   - Update `nginx_certs/` with production certificates
   - Or use Let's Encrypt for free certificates

3. **Network Access**: Restrict access to services
   - Prometheus (port 9090): Internal only - don't expose
  - Garage S3 (port 3900): Internal only - don't expose
   - Grafana (port 443): Only expose to authorized users

4. **Authentication**: Enable Grafana authentication
   - Set `admin_user` and `admin_password` in `grafana.ini`
   - Configure OAuth or LDAP if available

5. **API Keys**: Generate Grafana API keys for programmatic access
   - Avoid using admin credentials in integrations

## Backup and Recovery

### Backing Up Data

```bash
# Backup Prometheus data
docker cp prometheus:/prometheus/data prometheus-backup

# Backup Grafana data
docker volume inspect grafana | grep Mountpoint
# Copy the mountpoint directory

# Back up Garage metadata and objects (stop Garage first for a consistent copy)
docker compose stop garage
docker cp garage_s3:/var/lib/garage garage-backup
docker compose start garage
```

### Restoring Data

```bash
# Restore Prometheus
docker cp prometheus-backup prometheus:/prometheus/data

# Restore Grafana
# Copy to the volume mountpoint identified earlier

# Restore the backup into the garage-storage volume while Garage is stopped.
# Preserve the meta/ and data/ directories under /var/lib/garage.
```

## Additional Resources

- [Grafana Documentation](https://grafana.com/docs/)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Loki Documentation](https://grafana.com/docs/loki/)
- [Alloy Documentation](https://grafana.com/docs/alloy/)
- [Garage Documentation](https://garagehq.deuxfleurs.fr/documentation/)
- [Nginx Documentation](https://nginx.org/en/docs/)

## License

Configuration provided as-is for use with open-source monitoring tools.
