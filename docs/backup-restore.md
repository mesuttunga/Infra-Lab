# Backup & Restore Strategy

Disaster recovery procedures for cluster state and data persistence.

## Overview

This document covers backup strategies for both home lab and production environments.

**Home Lab:** Manual, infrequent backups (etcd snapshots only)  
**Production:** Automated, scheduled backups with off-site storage

---

## What Gets Backed Up

| Component | Contains | Backup Method |
|-----------|----------|---------------|
| **etcd snapshots** | K8s state (deployments, services, secrets, RBAC) | `k3s etcd-snapshot` |
| **Persistent volumes** | Application data (databases, files) | Volume-level backup or Velero |
| **Cluster config** | Infrastructure as Code | Git (Ansible roles) |
| **Container images** | Application images | Container registry (external) |

**Critical:** etcd contains cluster structure, NOT application data inside volumes.

---

## Home Lab Backup Procedure

### Manual etcd Snapshot

K3s automatically creates etcd snapshots every 12 hours and retains 5 snapshots.

**Create manual snapshot:**
```bash
ssh tunga-master
sudo k3s etcd-snapshot save --name manual-$(date +%Y%m%d-%H%M)
```

**List snapshots:**
```bash
sudo ls -lh /var/lib/rancher/k3s/server/db/snapshots/
```

**Snapshot location:**
```
/var/lib/rancher/k3s/server/db/snapshots/
├── on-demand-tunga-master-1234567890
├── etcd-snapshot-tunga-master-1234567891
└── manual-20260102-1430
```

### Copy to Bastion (Off-Node Storage)
```bash
# From master node:
sudo rsync -az /var/lib/rancher/k3s/server/db/snapshots/ \
  ubuntu@192.168.0.105:/home/ubuntu/etcd-backups/

# Verify on bastion:
ssh tunga-bastion
ls -lh ~/etcd-backups/
```

### Backup Schedule (Optional)

Create cron job on master for weekly backups:
```bash
# /etc/cron.weekly/etcd-backup.sh
#!/bin/bash
SNAP_DIR="/var/lib/rancher/k3s/server/db/snapshots"
DATE=$(date +%Y%m%d-%H%M)

# Create snapshot
/usr/local/bin/k3s etcd-snapshot save --name "weekly-$DATE"

# Retention: keep last 30 days
find $SNAP_DIR -type f -mtime +30 -delete

# Copy to bastion
rsync -az $SNAP_DIR/ ubuntu@192.168.0.105:/home/ubuntu/etcd-backups/
```

**Setup:**
```bash
sudo cp etcd-backup.sh /etc/cron.weekly/
sudo chmod +x /etc/cron.weekly/etcd-backup.sh
```

---

## Home Lab Restore Procedure

### Scenario: Master Node Failure (Cluster State Loss)

**Prerequisites:**
- Fresh Ubuntu installation on master node
- etcd snapshot available (from bastion or local backup)

### Step 1: Copy Snapshot to Master
```bash
# From bastion or backup location:
scp ~/etcd-backups/manual-20260102-1430 \
  ubuntu@192.168.0.100:/home/ubuntu/
```

### Step 2: Restore etcd Snapshot
```bash
ssh tunga-master

# Stop K3s (if running)
sudo systemctl stop k3s

# Move snapshot to K3s directory
sudo mkdir -p /var/lib/rancher/k3s/server/db/snapshots/
sudo mv /home/ubuntu/manual-20260102-1430 \
  /var/lib/rancher/k3s/server/db/snapshots/

# Restore from snapshot
sudo k3s server \
  --cluster-reset \
  --cluster-reset-restore-path=/var/lib/rancher/k3s/server/db/snapshots/manual-20260102-1430
```

**WARNING:** `--cluster-reset` deletes existing cluster state. Only use in disaster recovery.

### Step 3: Start K3s
```bash
# Start K3s normally (without reset flags)
sudo systemctl start k3s

# Verify cluster state
kubectl get nodes
kubectl get pods --all-namespaces
```

### Step 4: Rejoin Workers

Workers may need to rejoin if cluster certificates changed:
```bash
# On each worker:
ssh tunga-worker1
sudo systemctl stop k3s-agent
sudo rm -rf /var/lib/rancher/k3s/agent/
sudo systemctl start k3s-agent

# Verify:
kubectl get nodes
```

---

## Production Backup Strategy

### Automated Backup Tools

**Velero** (Industry Standard):
- Kubernetes-native backup solution
- Backs up etcd + persistent volumes
- Supports multiple storage backends (S3, GCS, Azure)
- Schedule-based automation
- Point-in-time recovery

**Installation:**
```bash
velero install \
  --provider aws \
  --bucket k8s-backups \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1
```

**Daily Backup Schedule:**
```bash
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --ttl 720h0m0s
```

### Production Best Practices

**Backup Frequency:**
- **etcd:** Hourly snapshots (automated)
- **Volumes:** Daily snapshots (Velero)
- **Manifests:** Continuous (GitOps)

**Retention Policy:**
- **Hourly:** Keep 24 hours
- **Daily:** Keep 30 days
- **Weekly:** Keep 12 weeks
- **Monthly:** Keep 12 months

**Storage Location:**
- **Primary:** Cloud object storage (S3, GCS)
- **Secondary:** Different region/availability zone
- **Tertiary:** Off-site cold storage (Glacier)

**Backup Testing:**
- **Monthly:** Restore test in staging environment
- **Quarterly:** Full DR drill with production snapshot
- **Document:** Mean Time To Recovery (MTTR)

**Monitoring:**
- Alert on backup failures
- Alert on backup age (> 24 hours old)
- Track backup size trends
- Test restore procedure regularly

### RTO/RPO Targets

| Environment | RTO (Recovery Time) | RPO (Data Loss) |
|-------------|---------------------|-----------------|
| Home Lab | 30-60 minutes | 24 hours |
| Production | < 15 minutes | < 1 hour |
| Critical Prod | < 5 minutes | < 5 minutes |

**RTO:** How quickly can cluster be restored?  
**RPO:** How much data loss is acceptable?

---

## Disaster Scenarios

### Scenario 1: Single Worker Node Failure

**Impact:** Pods on that node stop, reschedule to other workers  
**Recovery:** Ansible rebuild (10-15 minutes)  
**Backup needed:** No (cluster state intact)
```bash
# Rebuild worker
ansible-playbook -i inventory.ini setup_cluster.yml --limit tunga-worker1
```

### Scenario 2: Master Node Failure

**Impact:** Control plane down, cluster unmanageable  
**Recovery:** etcd restore + worker rejoin (30-45 minutes)  
**Backup needed:** Yes (etcd snapshot required)
```bash
# 1. Restore etcd snapshot (see above)
# 2. Rejoin workers
ansible-playbook -i inventory.ini setup_cluster.yml --limit workers --ask-vault-pass
```

### Scenario 3: Complete Cluster Loss

**Impact:** All nodes destroyed  
**Recovery:** Full rebuild with etcd restore (45-60 minutes)  
**Backup needed:** Yes (etcd snapshot + volume backups)
```bash
# 1. Install Ubuntu on all nodes
# 2. Bootstrap sudo
ansible-playbook -i inventory.ini bootstrap.yml --ask-become-pass

# 3. Deploy full stack
ansible-playbook -i inventory.ini setup_cluster.yml --ask-vault-pass
ansible-playbook -i inventory.ini deploy_gateway.yml --ask-vault-pass
ansible-playbook -i inventory.ini deploy_cert_manager.yml --ask-vault-pass
ansible-playbook -i inventory.ini deploy_cluster_issuer.yml --ask-vault-pass
ansible-playbook -i inventory.ini deploy_monitoring.yml --ask-vault-pass

# 4. Restore etcd snapshot on master
# 5. Restore persistent volume data (if needed)
```

### Scenario 4: Persistent Volume Data Loss

**Impact:** Application data lost (databases, uploads)  
**Recovery:** Depends on backup strategy  
**Backup needed:** Volume-level backups (Velero or storage snapshots)

**Home Lab:** No volume backup (acceptable for testing)  
**Production:** Velero volume snapshots or storage-level replication

---

## GitOps as Backup

**Philosophy:** Cluster state should be reproducible from Git.

**What's in Git:**
- Ansible playbooks (infrastructure)
- Kubernetes manifests (applications)
- Helm charts/values (configuration)

**Recovery without etcd backup:**
1. Rebuild cluster with Ansible
2. Deploy applications from Git (ArgoCD/Flux)
3. Restore persistent data from volume backups

**Advantage:** Declarative, auditable, version controlled  
**Limitation:** Doesn't capture secrets or runtime state

---

## Commands Reference

### Backup
```bash
# Manual etcd snapshot
sudo k3s etcd-snapshot save --name manual-$(date +%Y%m%d)

# List snapshots
sudo ls -lh /var/lib/rancher/k3s/server/db/snapshots/

# Copy to bastion
sudo rsync -az /var/lib/rancher/k3s/server/db/snapshots/ \
  ubuntu@192.168.0.105:/home/ubuntu/etcd-backups/
```

### Restore
```bash
# Stop K3s
sudo systemctl stop k3s

# Restore snapshot
sudo k3s server \
  --cluster-reset \
  --cluster-reset-restore-path=/var/lib/rancher/k3s/server/db/snapshots/<SNAPSHOT>

# Start K3s
sudo systemctl start k3s

# Verify
kubectl get nodes
```

### Cleanup
```bash
# Remove old snapshots (30+ days)
find /var/lib/rancher/k3s/server/db/snapshots/ -type f -mtime +30 -delete
```

---

## Resources

- [K3s Backup and Restore](https://docs.k3s.io/cli/etcd-snapshot)
- [Velero Documentation](https://velero.io/docs/)
- [Kubernetes Disaster Recovery Best Practices](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster)