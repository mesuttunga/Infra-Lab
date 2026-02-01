# Architecture

## Network Topology
```mermaid
graph TB
    Internet((Internet))
    Router[Router<br/>192.168.0.1]
    Bastion[tunga-bastion<br/>192.168.0.105<br/>2GB RAM, 121GB SSD<br/>SSH Jump Host]
    Master[tunga-master<br/>192.168.0.100<br/>16GB RAM, 240GB SSD]
    W1[tunga-worker1<br/>192.168.0.101<br/>4GB RAM, 500GB SSD]
    W2[tunga-worker2<br/>192.168.0.102<br/>4GB RAM, 300GB SSD]
    
    Internet --- Router
    Router --- Bastion
    Router -.-> Master
    Router -.-> W1
    Router -.-> W2
    
    Bastion --> Master
    Bastion --> W1
    Bastion --> W2
    
    style Bastion fill:#ff6b6b,stroke:#c92a2a,color:#fff
    style Master fill:#4a90e2,stroke:#2e5c8a,color:#fff
    style W1 fill:#7ed321,stroke:#5fa319,color:#fff
    style W2 fill:#7ed321,stroke:#5fa319,color:#fff
    style Router fill:#f5a623,stroke:#d68910,color:#fff
```

**Subnet**: 192.168.0.0/24  
**Gateway**: 192.168.0.1  
**DNS**: 8.8.8.8, 1.1.1.1

**Security**: Master and worker nodes accept SSH only from bastion (192.168.0.105)

## Node Specifications

### tunga-bastion (Jump Host)
- **Hardware**: MacBook Air A1466 (2010)
- **CPU**: Intel Core 2 Duo (2 cores @ 1.86GHz)
- **RAM**: 2GB
- **Storage**: 121GB
- **Network**: WiFi 802.11n
- **Role**: SSH bastion host, secure gateway to cluster
- **Services**: SSH jump server

### tunga-master (Control Plane + Worker)
- **Hardware**: MacBook Pro A1278
- **CPU**: Intel Core i5 (8 cores @ 2.5GHz)
- **RAM**: 16GB
- **Storage**: 240GB
- **Network**: WiFi 802.11n
- **Role**: K3s server, etcd, scheduler, controller-manager
- **Workload**: System pods + application pods
- **SSH Access**: Via bastion only

### tunga-worker1
- **Hardware**: i5 Laptop
- **CPU**: Intel Core i5-3210M (2 cores @ 2.5GHz)
- **RAM**: 4GB
- **Storage**: 500GB
- **Network**: WiFi 802.11n
- **Role**: K3s agent
- **Workload**: Application pods
- **SSH Access**: Via bastion only

### tunga-worker2
- **Hardware**: i3 Laptop
- **CPU**: Intel Core i3 (2 cores)
- **RAM**: 4GB
- **Storage**: 300GB
- **Network**: WiFi 802.11n
- **Role**: K3s agent
- **Workload**: Application pods
- **SSH Access**: Via bastion only

## Security Design

### SSH Bastion Pattern

Production-style security with jump host:
```
Local Machine → tunga-bastion → tunga-master/workers
                (only entry point)
```

**Access Flow:**
1. Direct SSH to bastion (192.168.0.105)
2. From bastion, SSH to master/workers
3. Or use ProxyJump for transparent access

**Firewall Rules (UFW):**
- Master/Workers: Allow SSH only from 192.168.0.105
- Bastion: Allow SSH from any (192.168.0.0/24)

**Benefits:**
- Single point of access (audit trail)
- Hardened bastion host
- Production security pattern
- Reduced attack surface

### Production Comparison

| Aspect                | Home Lab          | Production |
|-----------------------|-------------------|-------------------------------|
| Control Plane         | 1 node            | 3+ nodes (etcd quorum)        |
| LoadBalancer          | None              | HAProxy/cloud LB              |
| etcd                  | Single instance   | Clustered (raft consensus)    |
| Acceptable Downtime   | Minutes           | < 1 second                    |

**Trade-offs:**
- Single master crash = control plane down (kubectl stops working)
- Worker pods continue running during master outage
- No new pod scheduling until master recovery
- Sufficient for development/testing workloads

## Cluster Architecture
```mermaid
graph TB
    subgraph Bastion["tunga-bastion (Jump Host)"]
        SSH[SSH Server]
    end
    
    subgraph Master["tunga-master (Control Plane + Worker)"]
        API[API Server]
        SCHED[Scheduler]
        CTRL[Controller Manager]
        ETCD[(etcd)]
        KUBELET_M[kubelet]
        DOCKER_M[containerd]
        PODS_M[Application Pods]
    end
    
    subgraph W1["tunga-worker1"]
        KUBELET_W1[kubelet]
        DOCKER_W1[containerd]
        PODS_W1[Application Pods]
    end
    
    subgraph W2["tunga-worker2"]
        KUBELET_W2[kubelet]
        DOCKER_W2[containerd]
        PODS_W2[Application Pods]
    end
    
    KUBECTL[kubectl] --> SSH
    SSH -.->|ProxyJump| API
    API --> SCHED
    API --> CTRL
    API --> ETCD
    API -.-> KUBELET_W1
    API -.-> KUBELET_W2
    
    style Bastion fill:#ff6b6b,stroke:#c92a2a,color:#fff
    style Master fill:#4a90e2,stroke:#2e5c8a,color:#fff
    style W1 fill:#7ed321,stroke:#5fa319,color:#fff
    style W2 fill:#7ed321,stroke:#5fa319,color:#fff
    style ETCD fill:#d0021b,stroke:#9b0116,color:#fff
```

## Pod Network (CNI)

**CIDR**: 10.42.0.0/16 (K3s default)  
**Implementation**: Flannel VXLAN

Node CIDR allocation:
- `10.42.0.0/24` → tunga-master
- `10.42.1.0/24` → tunga-worker1
- `10.42.2.0/24` → tunga-worker2

## Resource Allocation

Pods are scheduled with resource requests/limits:
```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "200m"
```

Scheduler distributes based on available resources across nodes.

## Service Discovery

- **DNS**: CoreDNS (cluster.local domain)
- **Service CIDR**: 10.43.0.0/16
- **Service Types**: ClusterIP, NodePort

## Storage

- **Provisioner**: local-path (K3s built-in)
- **StorageClass**: local-path (default)
- **Volumes**: Hostpath-backed persistent volumes