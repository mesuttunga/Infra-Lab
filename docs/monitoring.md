# Monitoring

Observability stack based on kube-prometheus-stack, providing metrics collection, alerting and visualisation for the entire cluster.

## Stack Overview

| Component | Version | Purpose |
|-----------|---------|---------|
| Prometheus | kube-prometheus-stack | Metrics collection and alert evaluation |
| Grafana | kube-prometheus-stack | Metrics visualisation and dashboards |
| Alertmanager | kube-prometheus-stack | Alert routing and Slack notifications |
| kube-state-metrics | kube-prometheus-stack | Kubernetes object metrics |
| node-exporter | kube-prometheus-stack | Node-level OS metrics (CPU, memory, disk, network) |

All components are deployed via Helm into the `monitoring` namespace using the `kube-prometheus-stack` chart.

## Accessing the Stack

### Quick Aliases

Add to `~/.zshrc` or `~/.bashrc` for convenience:

```bash
alias grafana="kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80"
alias prometheus="kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090"
alias prometheus-alert="kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-alertmanager 9093:9093"
```

### Grafana

```bash
grafana
# Browser: http://localhost:3000
# Username: admin
# Password: defined in vault_grafana_admin_password
```

### Prometheus

```bash
prometheus
# Browser: http://localhost:9090
```

### Alertmanager

```bash
alertmanager
# Browser: http://localhost:9093
```

## Configuration

Monitoring is configured via an Ansible Jinja2 template:

```
ansible/roles/monitoring/templates/values.yml.j2
```

This file is rendered at deploy time and passed to Helm via `-f`. Sensitive values (Grafana password, Slack webhook URL) are stored in Ansible Vault and injected at render time.

### Deploying or Updating

```bash
ansible-playbook -i inventory.ini deploy_monitoring.yml --ask-vault-pass
```

### Adding Secrets to Vault

```bash
ansible-vault edit ansible/group_vars/all/vault.yml
```

Required vault variables:

```yaml
vault_grafana_admin_password: "YourPassword"
vault_slack_webhook_url: "https://hooks.slack.com/services/..."
```

## Alert Rules

Custom alert rules are defined in `values.yml.j2` under `additionalPrometheusRulesMap`. Rules are grouped by category.

### Node Rules

| Alert | Expression | For | Severity | Description |
|-------|-----------|-----|----------|-------------|
| NodeDown | `up{job="node-exporter"} == 0` | 1m | critical | Node exporter unreachable |
| NodeHighCPU | CPU usage > 85% (5m avg) | 5m | warning | Sustained high CPU usage |
| NodeHighMemory | Memory usage > 90% | 5m | warning | Sustained high memory usage |
| NodeDiskSpaceLow | Disk usage > 85% | 5m | warning | Disk space running low |

### Pod Rules

| Alert | Expression | For | Severity | Description |
|-------|-----------|-----|----------|-------------|
| PodCrashLooping | > 3 restarts in 15 minutes | 5m | critical | Pod is crash looping |
| PodOOMKilled | Last termination reason is OOMKilled | 0m | critical | Pod killed due to memory limit |
| PodNotReady | Pod ready condition false | 5m | warning | Pod stuck in non-ready state |

### K3s False Positives

The following alerts are fired by kube-prometheus-stack but do not indicate real issues in K3s. They are routed to a `null` receiver and suppressed from Slack notifications:

- **Watchdog** — intentional always-firing heartbeat alert
- **KubeSchedulerDown** — K3s embeds the scheduler, does not expose metrics endpoint
- **KubeControllerManagerDown** — same as above
- **KubeProxyDown** — K3s does not use kube-proxy
- **KubeAggregatedAPIDown** — K3s aggregated API behaviour differs from upstream
- **AlertmanagerClusterFailed** — Alertmanager runs as single replica (no cluster mode)
- **AlertmanagerFailedToSendAlerts** — historical send failures before Slack was configured

## Slack Alerting

Alerts are routed to Slack via Alertmanager. The webhook URL is stored in Ansible Vault.

### Routing Logic

```
Firing alert
    ↓
Alertmanager
    ↓
Match noisy K3s alerts? → null receiver (suppressed)
    ↓
All other alerts → Slack #home-lab-alert
```

### Alert Message Format

Each Slack notification includes:

- Alert status (FIRING / RESOLVED)
- Alert name
- Instance (IP and port)
- Summary
- Description
- Severity

### Repeat Interval

Alerts that remain firing are re-notified every 12 hours. Resolved alerts send a RESOLVED notification automatically.

## Grafana Dashboards

### Built-in Dashboards (kube-prometheus-stack)

| Dashboard | Description |
|-----------|-------------|
| Kubernetes / Compute Resources / Cluster | Cluster-wide CPU and memory overview |
| Kubernetes / Compute Resources / Node | Per-node resource breakdown |
| Kubernetes / Compute Resources / Pod | Per-pod CPU, memory and network |
| Node Exporter / Nodes | OS-level metrics per node |
| Alertmanager / Overview | Alert status and routing |

### Custom Dashboards

| Dashboard | File | Description |
|-----------|------|-------------|
| Home Lab Overview | `grafana/dashboards/home-lab-overview.json` | Single-pane cluster health view |

#### Home Lab Overview Panels

- **Node Status** — UP/DOWN indicator for each node (green/red)
- **Cluster CPU** — gauge showing average CPU across all nodes
- **Cluster Memory** — gauge showing average memory across all nodes
- **Firing Alerts** — count of active alerts (excludes K3s false positives)
- **Pod Restarts (15m)** — pod restart count in the last 15 minutes
- **CPU Usage per Node** — time series with mean and max per node
- **Memory Usage per Node** — time series with mean and max per node
- **Disk I/O per Node** — read/write bytes per node
- **Network Traffic per Node** — RX/TX bytes per node (Calico interfaces excluded)

#### Importing the Dashboard

```
Grafana → Dashboards → Import → Upload JSON file
Select: grafana/dashboards/home-lab-overview.json
```

## UFW Firewall Requirements

The following ports must be open on all cluster nodes for monitoring to function:

| Port | Protocol | Source | Purpose |
|------|----------|--------|---------|
| 9100 | TCP | 192.168.0.0/24 | node-exporter metrics scraping |
| any | any | 10.42.0.0/16 | pod network (Prometheus scrapes via pod IP) |

These rules are managed by the Ansible `security` role.

**Important:** If node-exporter metrics are missing for some nodes, verify UFW allows traffic from the pod network CIDR. Prometheus scrapes via its pod IP (`10.42.x.x`), not the node IP.

## Troubleshooting

### node-exporter metrics missing for some nodes

Check if Prometheus can reach the node-exporter:

```bash
# Query which node-exporters are UP
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090
# Run: up{job="node-exporter"}
# Expected: value 1 for all 3 nodes
```

If any node shows `0`, check UFW on that node:

```bash
ssh tunga-worker1
sudo ufw status numbered
sudo tail -f /var/log/ufw.log
```

Look for BLOCK entries with `DPT=9100`. If present, add the missing rule:

```bash
sudo ufw allow from 192.168.0.0/24 to any port 9100 proto tcp
sudo ufw allow from 10.42.0.0/16
```

Then update the Ansible security role to make it permanent.

### Grafana shows "No data"

Verify the Prometheus datasource is configured:

```
Grafana → Connections → Data Sources → Prometheus
URL: http://prometheus-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090
```

If the datasource is missing, redeploy monitoring:

```bash
ansible-playbook -i inventory.ini deploy_monitoring.yml --ask-vault-pass
```

### Alertmanager not sending to Slack

Check Alertmanager config is correct:

```bash
alertmanager
# Browser: http://localhost:9093/#/status
# Verify: receivers section shows slack with api_url: <secret>
```

Check Alertmanager logs:

```bash
kubectl logs -n monitoring alertmanager-prometheus-kube-prometheus-alertmanager-0 -c alertmanager
```

Common causes:
- Wrong Slack channel name (must include `#` prefix)
- Expired or revoked webhook URL
- Slack workspace permissions changed

### Port-forward drops during Helm upgrade

This is expected behaviour. Helm restarts pods during upgrade which terminates active port-forwards. Re-run the alias after deployment completes.

## Known Issues

### Prometheus scheduled on worker2

The Prometheus pod is currently scheduled on `tunga-worker2`. If worker2 is unavailable, Prometheus and Grafana become inaccessible. This is acceptable for a home lab but in production Prometheus would have persistent storage and node affinity rules to ensure stable placement.

### Single Alertmanager replica

Alertmanager runs as a single replica (no cluster mode). This triggers `AlertmanagerClusterFailed` alerts which are suppressed via the null receiver route. High availability would require 3+ replicas with gossip protocol enabled.