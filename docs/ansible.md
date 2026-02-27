# Cluster Automation & Disaster Recovery

Complete Infrastructure as Code implementation for reproducible cluster deployment.

## Directory Structure

```
ansible/
├── bootstrap.yml               # One-time passwordless sudo setup
├── setup_cluster.yml           # Full cluster deployment
├── deploy_gateway.yml          # Gateway API + Envoy Gateway
├── deploy_cert_manager.yml     # cert-manager installation
├── deploy_cluster_issuer.yml   # Cloudflare DNS-01 ClusterIssuer
├── deploy_monitoring.yml       # Prometheus + Grafana
├── inventory.ini               # Node definitions and groups
├── ansible.cfg                 # Ansible configuration
├── manifests/
│   └── gatewayclass.yaml       # Envoy GatewayClass manifest
├── group_vars/
│   └── all/
│       ├── main.yml            # Global variables (versions, network config)
│       └── vault.yml           # Encrypted secrets (never commit unencrypted)
└── roles/
    ├── common/                 # Base configuration for all nodes
    │   ├── defaults/main.yml   # Package lists, timeout values
    │   ├── handlers/main.yml   # Service restart triggers
    │   └── tasks/main.yml      # OS updates, packages, laptop settings
    ├── security/               # Infrastructure hardening
    │   ├── handlers/main.yml
    │   └── tasks/main.yml      # UFW firewall, SSH restrictions
    ├── k3s_master/             # Control plane initialization
    │   └── tasks/main.yml      # K3s server install, token extraction
    ├── k3s_worker/             # Worker node configuration
    │   └── tasks/main.yml      # K3s agent join
    ├── calico/                 # Calico CNI
    │   └── tasks/main.yml      # Tigera operator, Installation CR
    ├── metallb/                # Bare-metal load balancer
    │   └── tasks/main.yml      # MetalLB, IPAddressPool, L2Advertisement
    ├── gateway_api/            # Gateway API CRDs
    │   └── tasks/main.yml      # Standard channel CRDs (server-side apply)
    ├── envoy_gateway/          # Envoy Gateway controller
    │   └── tasks/main.yml      # Install, wait, GatewayClass
    ├── cert_manager/           # TLS certificate management
    │   └── tasks/main.yml      # Helm install with CRDs
    ├── cluster_issuer/         # Let's Encrypt issuer
    │   └── tasks/main.yml      # Cloudflare secret, ClusterIssuer
    └── monitoring/             # Observability stack
        └── tasks/main.yml      # Prometheus + Grafana via Helm
```

## Key Variables (group_vars/all/main.yml)

```yaml
# K3s
k3s_version: "v1.35.1+k3s1"
k8s_kubeconfig: /etc/rancher/k3s/k3s.yaml

# Calico
calico_version: "v3.29.1"
pod_cidr: "10.42.0.0/16"

# MetalLB
metallb_version: "v0.14.9"
metallb_ip_range: "192.168.0.200-192.168.0.210"
metallb_namespace: "metallb-system"

# Gateway API
gateway_api_version: "v1.4.1"

# Envoy Gateway
envoy_gateway_version: "v1.7.0"
envoy_gateway_namespace: "envoy-gateway-system"

# cert-manager
cert_manager_version: "v1.19.1"
cert_manager_namespace: "cert-manager"

# Secrets (from vault.yml)
cloudflare_api_token: "{{ vault_cloudflare_api_token }}"
letsencrypt_email: "your@email.com"
```

## Secrets Management (vault.yml)

Sensitive values are stored encrypted in `group_vars/all/vault.yml`:

```yaml
---
vault_grafana_admin_password: "YourPassword"
vault_cloudflare_api_token: "YourCloudflareToken"
```

Encrypt:
```bash
ansible-vault encrypt group_vars/all/vault.yml
```

Edit encrypted file:
```bash
ansible-vault edit group_vars/all/vault.yml
```

Run playbooks with vault:
```bash
ansible-playbook -i inventory.ini setup_cluster.yml --ask-vault-pass
```

**Important**: `vault.yml` is in `.gitignore` and must never be committed unencrypted.

## Prerequisites

- **OS**: Ubuntu 24.04 LTS Server (fresh install)
- **User**: `ubuntu` with sudo privileges on all nodes
- **SSH**: Public key authentication configured
- **Network**: All nodes reachable via IPs in inventory
- **Ansible**: Version 2.9+ on local machine

## Initial Setup (One-Time Bootstrap)

### Step 1: Add SSH Keys
```bash
ssh-copy-id ubuntu@192.168.0.105  # bastion
ssh-copy-id ubuntu@192.168.0.100  # master
ssh-copy-id ubuntu@192.168.0.101  # worker1
ssh-copy-id ubuntu@192.168.0.102  # worker2
```

### Step 2: Bootstrap Passwordless Sudo
```bash
cd ansible/
ansible-playbook -i inventory.ini bootstrap.yml --ask-become-pass
```

What it does:
- Creates `/etc/sudoers.d/ubuntu` with `NOPASSWD:ALL`
- Validates sudoers file syntax

### Step 3: Configure Secrets
```bash
nano group_vars/all/vault.yml
ansible-vault encrypt group_vars/all/vault.yml
```

### Step 4: Deploy Full Stack
```bash
ansible-playbook -i inventory.ini setup_cluster.yml --ask-vault-pass
ansible-playbook -i inventory.ini deploy_gateway.yml --ask-vault-pass
ansible-playbook -i inventory.ini deploy_cert_manager.yml --ask-vault-pass
ansible-playbook -i inventory.ini deploy_cluster_issuer.yml --ask-vault-pass
ansible-playbook -i inventory.ini deploy_monitoring.yml --ask-vault-pass
```

## Role Responsibilities

**common** (all nodes):
- Disable unattended-upgrades (prevents apt lock during automation)
- System package updates
- Essential package installation
- Laptop-specific settings (lid-close, GRUB console timeout)

**security** (cluster nodes):
- UFW firewall reset and configuration
- Default deny incoming / allow outgoing
- SSH restricted to bastion (192.168.0.105)
- K3s, Kubelet, Calico ports restricted to trusted subnet (192.168.0.0/24)

**k3s_master**:
- K3s server installation (Flannel disabled, Traefik disabled)
- Node token extraction for workers

**k3s_worker**:
- K3s agent installation and cluster join
- Idempotent (skips if already joined)

**calico**:
- Tigera operator deployment
- Calico Installation CR (VXLANCrossSubnet, pod CIDR)
- Waits for calico-system pods Ready

**metallb**:
- MetalLB manifest deployment
- IPAddressPool (192.168.0.200-210)
- L2Advertisement

**gateway_api**:
- Standard channel CRDs (server-side apply)
- Verifies gateways.gateway.networking.k8s.io CRD

**envoy_gateway**:
- Envoy Gateway deployment (server-side apply, force-conflicts)
- Waits for envoy-gateway-system pods Ready
- Creates `envoy` GatewayClass

**cert_manager**:
- Helm install (jetstack/cert-manager)
- CRDs installed via Helm
- Waits for cert-manager pods Ready

**cluster_issuer**:
- Cloudflare API token Secret
- ClusterIssuer (letsencrypt-prod, DNS-01)
- Verifies issuer Ready status

**monitoring** (separate lifecycle):
- Helm install kube-prometheus-stack
- All components pinned to tunga-master

## Idempotency

All playbooks are idempotent — safe to run multiple times:

```
First run:   changed: [tunga-master]   # Changes applied
Second run:  ok: [tunga-master]        # Already configured
```

## Disaster Recovery Procedure

### Scenario: Single Worker Node Failure

```bash
# 1. Install Ubuntu 24.04 on replacement node

# 2. Verify connectivity
ping 192.168.0.101

# 3. Bootstrap
ansible-playbook -i inventory.ini bootstrap.yml --ask-become-pass --limit tunga-worker1

# 4. Deploy
ansible-playbook -i inventory.ini setup_cluster.yml --ask-vault-pass --limit tunga-worker1,master

# 5. Verify
kubectl get nodes
```

### Scenario: Complete Cluster Rebuild

```bash
# 1. Install Ubuntu 24.04 on all 4 nodes

# 2. Bootstrap all nodes
ansible-playbook -i inventory.ini bootstrap.yml --ask-become-pass

# 3. Deploy full stack
ansible-playbook -i inventory.ini setup_cluster.yml --ask-vault-pass
ansible-playbook -i inventory.ini deploy_gateway.yml --ask-vault-pass
ansible-playbook -i inventory.ini deploy_cert_manager.yml --ask-vault-pass
ansible-playbook -i inventory.ini deploy_cluster_issuer.yml --ask-vault-pass
ansible-playbook -i inventory.ini deploy_monitoring.yml --ask-vault-pass
```

**Time**: ~30-45 minutes including OS installation.

## Development & Validation Workflow

### Dry Run (Check Mode)
```bash
ansible-playbook -i inventory.ini setup_cluster.yml --check --ask-vault-pass
```

### Diff Mode
```bash
ansible-playbook -i inventory.ini setup_cluster.yml --check --diff --ask-vault-pass
```

### Target Specific Nodes
```bash
ansible-playbook -i inventory.ini setup_cluster.yml --limit tunga-worker1 --ask-vault-pass
ansible-playbook -i inventory.ini setup_cluster.yml --limit workers --ask-vault-pass
```

### Lint
```bash
ansible-lint setup_cluster.yml
```

## Troubleshooting

### Connection Issues
```bash
ansible all -i inventory.ini -m ping
ansible-playbook -i inventory.ini setup_cluster.yml -v --ask-vault-pass
```

### Sudo Password Issues
```bash
ansible-playbook -i inventory.ini bootstrap.yml --ask-become-pass
```

### Apt Lock (unattended-upgrades)
```bash
# Temporary fix on affected node:
ssh tunga-worker1
sudo systemctl stop unattended-upgrades

# Permanent fix is in common role (disabled via systemd)
```

### Playbook Failures
```bash
# Resume from specific task:
ansible-playbook -i inventory.ini setup_cluster.yml \
  --start-at-task="Install K3s Server" --ask-vault-pass
```

### Vault Issues
```bash
# Re-encrypt if needed:
ansible-vault rekey group_vars/all/vault.yml

# View encrypted content:
ansible-vault view group_vars/all/vault.yml
```

## Best Practices

1. Always test with `--check` before applying changes
2. Use version control for all playbook changes
3. Keep `vault.yml` encrypted and out of git
4. Use `--limit` for single-node changes
5. Document all version changes in `group_vars/all/main.yml`
6. Run `ansible-lint` before committing
7. Take etcd snapshot before major changes