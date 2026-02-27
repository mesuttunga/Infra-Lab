# Setup Guide

## Prerequisites

- 4x machines (laptops/desktops): 1 bastion + 1 master + 2 workers
- Ubuntu 24.04 LTS Server ISO
- Router with DHCP reservation capability
- Network access for all nodes
- Ansible 2.9+ on local machine
- Cloudflare account with API token (DNS-01 TLS challenge)

## Hardware Preparation

### Bastion Node (2GB+ RAM)
1. Boot from Ubuntu installation media
2. Configure BIOS to ignore lid close (laptops)
3. Hostname: tunga-bastion
4. User: ubuntu

### Master Node (16GB RAM recommended)
1. Boot from Ubuntu installation media
2. Configure BIOS to ignore lid close (laptops)
3. Hostname: tunga-master
4. User: ubuntu

### Worker Nodes
1. Ensure 4GB+ RAM available
2. Hostname: tunga-worker1, tunga-worker2
3. User: ubuntu

## Ubuntu Installation

### Network Configuration

Static IP assignment via netplan:
```yaml
# /etc/netplan/00-installer-config.yaml
network:
  version: 2
  ethernets:
    <interface>:  # e.g., enp0s25
      dhcp4: no
      addresses:
        - 192.168.0.105/24  # bastion: .105, master: .100, worker1: .101, worker2: .102
      routes:
        - to: default
          via: 192.168.0.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
```

For WiFi interface:
```yaml
network:
  version: 2
  wifis:
    <interface>:  # e.g. wlp7s0
      dhcp4: no
      addresses:
        - 192.168.0.105/24
      routes:
        - to: default
          via: 192.168.0.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
      access-points:
        "WIFI-SSID":
          password: "wifi_password"
```

Apply: `sudo netplan apply`

### Headless Operation (Laptops)

Prevent sleep on lid close:
```bash
sudo vi /etc/systemd/logind.conf

# Set:
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore

sudo systemctl restart systemd-logind
```

Turn off screen after 60 seconds:
```bash
sudo nano /etc/default/grub

# Replace:
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
# With:
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash consoleblank=60"

sudo update-grub
sudo reboot
```

### LVM Volume Extension

Ubuntu installer may not use full disk:
```bash
sudo vgs
sudo lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
```

## Security Configuration (Bastion Setup)

### Step 1: SSH Key Generation (Local Machine)
```bash
ssh-keygen -t ed25519 -C "homelab"
```

### Step 2: Copy Keys to All Nodes
```bash
ssh-copy-id ubuntu@192.168.0.105  # bastion
ssh-copy-id ubuntu@192.168.0.100  # master
ssh-copy-id ubuntu@192.168.0.101  # worker1
ssh-copy-id ubuntu@192.168.0.102  # worker2
```

### Step 3: SSH Config (Local Machine)
```bash
cat >> ~/.ssh/config <<EOF
Host tunga-bastion
    HostName 192.168.0.105
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519

Host tunga-master
    HostName 192.168.0.100
    User ubuntu
    ProxyJump tunga-bastion
    IdentityFile ~/.ssh/id_ed25519

Host tunga-worker1
    HostName 192.168.0.101
    User ubuntu
    ProxyJump tunga-bastion
    IdentityFile ~/.ssh/id_ed25519

Host tunga-worker2
    HostName 192.168.0.102
    User ubuntu
    ProxyJump tunga-bastion
    IdentityFile ~/.ssh/id_ed25519
EOF
```

Test:
```bash
ssh tunga-bastion   # Direct
ssh tunga-master    # Via bastion (ProxyJump)
ssh tunga-worker1   # Via bastion (ProxyJump)
ssh tunga-worker2   # Via bastion (ProxyJump)
```

## Cluster Deployment (Ansible)

All cluster components are deployed via Ansible. Manual installation is not required.

### Step 1: Bootstrap Passwordless Sudo
```bash
cd ansible/
ansible-playbook -i inventory.ini bootstrap.yml --ask-become-pass
```

### Step 2: Configure Secrets
```bash
nano group_vars/all/vault.yml
```

```yaml
---
vault_grafana_admin_password: "YourPassword"
vault_cloudflare_api_token: "YourCloudflareToken"
```

Encrypt the vault file:
```bash
ansible-vault encrypt group_vars/all/vault.yml
```

### Step 3: Deploy Cluster
```bash
# Full cluster: K3s + Calico + MetalLB + Gateway API + Envoy Gateway
ansible-playbook -i inventory.ini setup_cluster.yml --ask-vault-pass
```

### Step 4: Deploy Gateway and TLS
```bash
ansible-playbook -i inventory.ini deploy_gateway.yml --ask-vault-pass
ansible-playbook -i inventory.ini deploy_cert_manager.yml --ask-vault-pass
ansible-playbook -i inventory.ini deploy_cluster_issuer.yml --ask-vault-pass
```

### Step 5: Deploy Monitoring (Optional)
```bash
ansible-playbook -i inventory.ini deploy_monitoring.yml --ask-vault-pass
```

### Step 6: Verify
```bash
kubectl get nodes
kubectl get pods -n calico-system
kubectl get pods -n metallb-system
kubectl get pods -n envoy-gateway-system
kubectl get pods -n cert-manager
kubectl get gatewayclass
kubectl get clusterissuer
```

Expected output:
```
NAME            STATUS   ROLES                AGE
tunga-master    Ready    control-plane,etcd   5m
tunga-worker1   Ready    <none>               3m
tunga-worker2   Ready    <none>               3m
```

## Kubeconfig Setup (Local Machine)
```bash
scp -o ProxyJump=tunga-bastion ubuntu@192.168.0.100:/etc/rancher/k3s/k3s.yaml ~/.kube/tunga-config
sed -i '' 's/127.0.0.1/192.168.0.100/g' ~/.kube/tunga-config
echo 'export KUBECONFIG=~/.kube/tunga-config' >> ~/.zshrc
source ~/.zshrc
kubectl get nodes
```

## Router Configuration

For external access:
1. Reserve 192.168.0.200-210 outside DHCP range
2. Port forward 80 and 443 to 192.168.0.200 (MetalLB pool start)
3. Cloudflare DNS: A record pointing to your public IP

## Monitoring Stack

### Accessing Grafana
```bash
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80
# Browser: http://localhost:3000 / Username: admin
```

Retrieve password:
```bash
kubectl get secret --namespace monitoring prometheus-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

### Accessing Prometheus
```bash
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090
# Browser: http://localhost:9090
```

## Troubleshooting

### SSH Access Issues
```bash
ssh tunga-bastion
ssh -J tunga-bastion ubuntu@192.168.0.100
sudo ufw status
```

### Node NotReady
```bash
ssh tunga-master && sudo systemctl status k3s
sudo journalctl -u k3s -f

ssh tunga-worker1 && sudo systemctl status k3s-agent
sudo journalctl -u k3s-agent -f
```

### Calico Issues
```bash
kubectl get pods -n calico-system
kubectl describe pod -n calico-system <pod-name>
```

### MetalLB Issues
```bash
kubectl get pods -n metallb-system
kubectl get ipaddresspool -n metallb-system
kubectl get l2advertisement -n metallb-system
```

### cert-manager Issues
```bash
kubectl get clusterissuer
kubectl describe clusterissuer letsencrypt-prod
kubectl get certificate -A
kubectl get certificaterequest -A
```

### Pod Stuck in Pending
```bash
kubectl describe pod <pod-name>
# Common causes: insufficient resources, node selector mismatch, PVC not bound
```

### Master Node Recovery
```bash
# Workers continue running during master outage
# Wait 5-10 minutes for automatic recovery after restart
kubectl delete pod <pod-name>  # Force rescheduling if needed
```

### etcd Backup
```bash
ssh tunga-master
sudo ls -lh /var/lib/rancher/k3s/server/db/snapshots/
sudo k3s etcd-snapshot save
```

## Performance Tuning

### Swap Configuration (Workers with 4GB RAM)
```bash
ssh tunga-worker1
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### Resource Limits

Always define pod resource requests and limits:
```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "200m"
```