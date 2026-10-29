# Monitoring Stack with Prometheus, Grafana & Alertmanager

A ready-to-run monitoring stack deployed with Docker Compose. It collects host and container metrics (CPU, memory, disk, network), visualizes them in Grafana dashboards, and fires alerts through Alertmanager.

## Architecture

```
Node Exporter (host metrics) ──┐
                               ├──> Prometheus ──> Grafana (dashboards)
cAdvisor (container metrics) ──┘        │
                                        └──> Alertmanager (alerts)
```

| Service | Purpose | Port |
|---|---|---|
| Prometheus | Scrapes and stores metrics, evaluates alert rules | 9090 |
| Node Exporter | Host metrics (CPU, memory, disk, network) | 9100 |
| cAdvisor | Per-container metrics | 8082 |
| Alertmanager | Groups and routes alerts | 9093 |
| Grafana | Dashboards and visualization | 3000 |

## Quick start

```bash
git clone https://github.com/manal333/monitoring-stack.git
cd monitoring-stack
cp .env.example .env        # set your Grafana password
docker compose up -d
docker compose ps
```

- Grafana: http://localhost:3000 (user/password from `.env`)
- Prometheus: http://localhost:9090 (check **Status > Targets**, all should be UP)
- Alertmanager: http://localhost:9093

The Prometheus data source is provisioned automatically.

## Dashboards

Import these community dashboards in Grafana (**Dashboards > New > Import**, enter the ID, choose the Prometheus data source):

- **1860** - Node Exporter Full (host metrics)
- **14282** - cAdvisor / Docker containers

Exported dashboard JSON files live in `grafana/dashboards/` and are loaded automatically on startup.

## Alert rules

Defined in `prometheus/alert.rules.yml`:

| Alert | Condition | Severity |
|---|---|---|
| InstanceDown | Target unreachable for 1 min | critical |
| HighCPUUsage | Host CPU > 80% for 2 min | warning |
| HighMemoryUsage | Host memory > 85% for 2 min | warning |
| LowDiskSpace | Free disk < 15% for 2 min | warning |
| ContainerHighCPU | Container CPU > 80% for 2 min | warning |
| ContainerDown | Container not seen for 60 s | critical |

Notifications (Slack / email) can be enabled in `alertmanager/alertmanager.yml`.

## Testing the alerts

```bash
# Service down: stop a target, then watch Prometheus > Alerts and Alertmanager
docker compose stop node-exporter     # InstanceDown fires after ~1 min
docker compose start node-exporter

# High CPU: generate load for 5 minutes
docker run --rm alpine sh -c "apk add --no-cache stress-ng && stress-ng --cpu 0 --timeout 300s"
```

## Useful PromQL

```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100)   # CPU %
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100            # Memory %
rate(node_network_receive_bytes_total[2m])                                         # Network in
sum by (name) (rate(container_cpu_usage_seconds_total{name!=""}[2m]))              # CPU per container
```

## Screenshots

Add screenshots of your dashboards and alerts to `docs/` and link them here.

## Stop / cleanup

```bash
docker compose down        # keep data
docker compose down -v     # remove data volumes too
```

## Notes

- On Docker Desktop (Windows/macOS) host metrics describe the Docker VM, not the physical machine. For accurate results run it on a Linux host or VM (e.g. Ubuntu / AWS EC2).
