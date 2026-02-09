# Setup Guide

## Prerequisites

- 4x machines (laptops/desktops): 1 bastion + 1 master + 2 workers
- Ubuntu 24.04 LTS Server ISO
- Router with DHCP reservation capability
- Network access for all nodes

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
        - 192.168.0.105/24  # tunga-bastion: .105, tunga-master: .100, tunga-worker1: .101, tunga-worker2: .102
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
# /etc/netplan/00-installer-config.yaml
network:
  version: 2
  wifis:
    <interface>:  # WiFi interface. e.g. wlp7s0
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
        "WIFI-SSID":  # WiFi SSID
          password: "wifi_password"
```

Apply: `sudo netplan apply`

**Alternative**: Use DHCP with router-side MAC address reservation.

### Headless Operation (Laptops)

Prevent sleep on lid close:
```bash
sudo nano /etc/systemd/logind.conf

# Set:
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore

sudo systemctl restart systemd-logind
```

# Turn off screen (even lid is open):

```bash
sudo nano /etc/default/grub

# Find this row
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"

# Replace with this
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash consoleblank=60"
# screen will turn off after 60 sec

# Apply
sudo update-grub
sudo reboot
```

### LVM Volume Extension

Ubuntu installer may not use full disk:
```bash
# Check available space
sudo vgs

# Extend logical volume
sudo lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
```

## Security Configuration (Bastion Setup)

### Step 1: SSH Key Generation (Bastion)
```bash
# On bastion:
ssh-keygen -t ed25519 -C "bastion-to-cluster"
# Press Enter 3 times (no passphrase)
```

### Step 2: Copy Keys to Cluster Nodes
```bash
# From bastion:
ssh-copy-id ubuntu@192.168.0.100
ssh-copy-id ubuntu@192.168.0.101
ssh-copy-id ubuntu@192.168.0.102
```

### Step 3: Firewall Configuration (Master + Workers)
```bash
# On tunga-master:
ssh ubuntu@192.168.0.100
sudo ufw reset
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.0.105 to any port 22 proto tcp comment 'Bastion SSH'
sudo ufw allow from 192.168.0.0/24 to any port 6443 proto tcp comment 'K3s API Server'
sudo ufw allow from 192.168.0.0/24 to any port 10250 proto tcp comment 'K3s Kubelet'
sudo ufw allow from 192.168.0.0/24 to any port 8472 proto udp comment 'Flannel VXLAN'
sudo ufw logging on
sudo ufw enable
exit

# On tunga-worker1:
ssh ubuntu@192.168.0.101
sudo ufw reset
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.0.105 to any port 22 proto tcp comment 'Bastion SSH'
sudo ufw allow from 192.168.0.0/24 to any port 6443 proto tcp comment 'K3s API Server'
sudo ufw allow from 192.168.0.0/24 to any port 10250 proto tcp comment 'K3s Kubelet'
sudo ufw allow from 192.168.0.0/24 to any port 8472 proto udp comment 'Flannel VXLAN'
sudo ufw logging on
sudo ufw enable
exit

# On tunga-worker2:
ssh ubuntu@192.168.0.102
sudo ufw reset
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.0.105 to any port 22 proto tcp comment 'Bastion SSH'
sudo ufw allow from 192.168.0.0/24 to any port 6443 proto tcp comment 'K3s API Server'
sudo ufw allow from 192.168.0.0/24 to any port 10250 proto tcp comment 'K3s Kubelet'
sudo ufw allow from 192.168.0.0/24 to any port 8472 proto udp comment 'Flannel VXLAN'
sudo ufw logging on
sudo ufw enable
exit
```

# Disable password auth and root login
sudo sed -i 's/^#\\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/^#\\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl restart ssh


**Result**: Master/workers only accept SSH from bastion (192.168.0.105)

### Step 4: SSH Config (Bastion)
```bash
# On tunga-bastion:
cat >> ~/.ssh/config <<EOF
Host tunga-master
    HostName 192.168.0.100
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519

Host tunga-worker1
    HostName 192.168.0.101
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519

Host tunga-worker2
    HostName 192.168.0.102
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
EOF
```

### Step 5: SSH Config (Local Machine)
```bash
# On your local machine:
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

**Test:**
```bash
# From local machine:
ssh tunga-bastion  # Direct
ssh tunga-master   # Via bastion (ProxyJump)
ssh tunga-worker1  # Via bastion (ProxyJump)
```

## K3s Installation

### Master Node
```bash
# SSH via bastion:
ssh tunga-master

curl -sfL https://get.k3s.io | sh -s - server \
  --cluster-init \
  --disable=traefik \
  --write-kubeconfig-mode=644
```

Retrieve join token:
```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

### Worker Nodes
```bash
# Worker1:
ssh tunga-worker1
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.0.100:6443 \
  K3S_TOKEN="<TOKEN_FROM_MASTER>" sh -

# Worker2:
ssh tunga-worker2
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.0.100:6443 \
  K3S_TOKEN="<TOKEN_FROM_MASTER>" sh -
```

### Verification
```bash
# On master:
ssh tunga-master
kubectl get nodes

# Expected output:
NAME            STATUS   ROLES                AGE
tunga-master    Ready    control-plane,etcd   1m
tunga-worker1   Ready    <none>               30s
tunga-worker2   Ready    <none>               30s
```

## Kubeconfig Setup (Local Machine)
```bash
# Copy config from master via bastion
scp -o ProxyJump=tunga-bastion ubuntu@192.168.0.100:/etc/rancher/k3s/k3s.yaml ~/.kube/tunga-config

# Update server IP (Local Machine)
sed -i '' 's/127.0.0.1/192.168.0.100/g' ~/.kube/tunga-config

# Make it permanent (add to ~/.zshrc or ~/.bashrc on Local Machine)
echo 'export KUBECONFIG=~/.kube/tunga-config' >> ~/.zshrc
source ~/.zshrc

# Test
kubectl get nodes
```

**Note**: kubectl commands work via K3s API (port 6443), not SSH. All cluster management happens from local machine.

## Monitoring Stack

**All commands run from local machine**, not on master (control plane).
```bash
# Add Prometheus Helm repo (Local Machine)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack (Local Machine)
helm install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  --set grafana.adminPassword="ChangeMe123!"

# Wait for pods to be ready
kubectl get pods -n monitoring -w

# Access Grafana

# Terminal 1 (on master):
ssh tunga-master
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# Terminal 2 (SSH tunnel from local machine):
ssh -L 3000:localhost:3000 ubuntu@tunga-master

# Browser: http://localhost:3000 (admin / ChangeMe123!)
```

**Production Pattern**: Never SSH to master for cluster management. Use kubectl/helm from local machine.

## Troubleshooting

### SSH Access Issues
```bash
# Test bastion connectivity:
ssh tunga-bastion

# Test master via bastion:
ssh -J tunga-bastion ubuntu@192.168.0.100

# Check firewall on master/workers:
sudo ufw status
```

### Node NotReady
```bash
# Check kubelet status
ssh tunga-master
sudo systemctl status k3s

ssh tunga-worker1
sudo systemctl status k3s-agent

# View logs
sudo journalctl -u k3s -f
sudo journalctl -u k3s-agent -f
```

### Network Issues
```bash
# Test connectivity
ping 192.168.0.100
curl -k https://192.168.0.100:6443

# From bastion, check cluster nodes:
ssh tunga-bastion
ping 192.168.0.100
ping 192.168.0.101
ping 192.168.0.102
```

### Pod Stuck in Pending
```bash
kubectl describe pod <pod-name>

# Common causes:
# - Insufficient resources
# - Node selector mismatch
# - Volume mount issues
```

### Master Node Recovery

If master crashes and restarts:
```bash
# Worker nodes will show NotReady briefly
# Pods continue running
# Wait 5-10 minutes for automatic recovery

# Force pod rescheduling if needed:
kubectl delete pod <pod-name>
```

### etcd Backup (Recommended)
```bash
# K3s includes automatic etcd snapshots
ssh tunga-master
sudo ls -lh /var/lib/rancher/k3s/server/db/snapshots/

# Manual snapshot:
sudo k3s etcd-snapshot save
```

## Performance Tuning

### Swap Configuration

Workers with 4GB RAM may benefit from swap:
```bash
ssh tunga-worker1

sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Persist:
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### Resource Limits

Always define pod resource requests/limits:
```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "200m"
```

## Next Steps

- Deploy applications with proper resource constraints
- Configure persistent volumes for stateful workloads
- Implement GitOps workflow (ArgoCD/Flux)
- Add monitoring dashboards