# Design Decisions & Rationale

This document explains the reasoning behind key design choices in this Kubernetes lab.
The goal is to balance realism, simplicity and reproducibility on limited homelab hardware.

---

## Why K3s?

K3s was chosen because it is lightweight, production-compatible and optimized for low-resource environments. It allows running a realistic multi-node Kubernetes cluster on older hardware without sacrificing core Kubernetes behaviour such as scheduling, networking and self-healing.

K3s is also used in production for edge and on-premises deployments, making it a valid choice beyond just homelab use.

---

## Why Calico CNI?

Calico supports NetworkPolicy enforcement. This is essential for production-grade workload isolation. Calico also supports BGP routing, which is the standard in enterprise on-premises environments. The current setup uses VXLAN encapsulation but is BGP-ready for future router upgrades.

---

## Why MetalLB?

Bare-metal Kubernetes clusters have no built-in LoadBalancer implementation. Without MetalLB, Services of type LoadBalancer remain in a Pending state indefinitely.

MetalLB provides L2 (ARP-based) load balancing on the local network which is the standard approach for on-premises clusters without a cloud provider. The IP pool (192.168.0.200-210) is reserved outside the DHCP range to avoid conflicts.

---

## Why Gateway API and Envoy Gateway?

The Kubernetes project announced that Ingress NGINX will be retired in March 2026. Gateway API is the official successor and is now GA (v1.0+).

Gateway API introduces a role-oriented model with three separate resources: GatewayClass (infrastructure), Gateway (cluster operator) and HTTPRoute (application developer). This separation of concerns reflects how real organisations manage ingress at scale.

Envoy Gateway was chosen as the Gateway API implementation because it is built on Envoy Proxy, which is the de facto standard data plane in production environments. It also has strong compatibility with Calico and supports advanced traffic management features.

---

## Why cert-manager with DNS-01?

cert-manager automates TLS certificate issuance and renewal from Let's Encrypt. This eliminates manual certificate management entirely.

DNS-01 challenge was chosen over HTTP-01 because it works without exposing port 80 to the internet during validation. It also supports wildcard certificates if needed in the future. Cloudflare was chosen as the DNS provider because it has a well-supported cert-manager integration.

---

## Why a Bastion Host?

A bastion host simulates a common enterprise security pattern. Instead of exposing SSH on every node, access is restricted through a single jump host. This reduces the attack surface and provides a central audit point for all administrative access.

---

## Why Ansible Automation?

Manual setup is error-prone and not repeatable. Ansible ensures the cluster can be rebuilt from scratch in a consistent way. This supports disaster recovery and aligns with Infrastructure as Code principles.

The full stack (K3s, Calico, MetalLB, Gateway API, Envoy Gateway, cert-manager) can be deployed on fresh Ubuntu nodes in approximately 30 minutes.

---

## Why local-path Storage?

local-path storage is simple and sufficient for non-critical workloads. The goal of this lab is orchestration and automation, not building a distributed storage system. This keeps complexity low while still supporting persistent volumes for stateful applications.

---

## Why Separate Playbooks for Each Component?

The cluster core (K3s, Calico, MetalLB) and the application stack (Gateway, TLS, Monitoring) have different lifecycles. Separating them into individual playbooks allows each component to be updated, redeployed or troubleshot independently without touching the rest of the cluster.

---

## Known Limitations

This lab intentionally accepts some trade-offs due to hardware and scope.

- Single control plane (no HA etcd quorum)
- L2 load balancing only (no BGP router connected yet)
- WiFi-based networking between nodes
- Local storage only (no distributed storage)
- Not designed for zero-downtime production workloads

These limitations are acceptable for development and testing purposes.

---

## Quick Start

For rebuilding the cluster from scratch:

```bash
cd ansible/
ansible-playbook -i inventory.ini bootstrap.yml --ask-become-pass
ansible-playbook -i inventory.ini setup_cluster.yml --ask-vault-pass
ansible-playbook -i inventory.ini deploy_gateway.yml --ask-vault-pass
ansible-playbook -i inventory.ini deploy_cert_manager.yml --ask-vault-pass
ansible-playbook -i inventory.ini deploy_cluster_issuer.yml --ask-vault-pass
```