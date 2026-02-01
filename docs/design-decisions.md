# Design Decisions & Rationale

This document explains the reasoning behind key design choices in this Kubernetes lab.  
The goal is to balance realism, simplicity and reproducibility on limited homelab hardware.

---

# Why These Choices?

## Why K3s?

- K3s was chosen because it is lightweight, production-compatible and optimized for low-resource environments.
- It allows running a realistic multi-node Kubernetes cluster on older computers without sacrificing core Kubernetes behavior such as scheduling, networking and self-healing.
- This makes it ideal for testing production patterns at home.

---

## Why a Bastion Host?

- A bastion host simulates common enterprise security patterns.
- Instead of exposing SSH on every node, access is restricted through a single jump host. This reduces attack surface and provides a central audit point.
- It also reflects how many production environments control administrative access.

---

## Why Ansible Automation?

- Manual setup is error-prone and not repeatable.
- Ansible ensures the cluster can be rebuilt from scratch in a consistent and reproducible way. This supports disaster recovery scenarios and aligns with Infrastructure as Code principles.
- If a node fails, the cluster can be recreated on fresh machines using the same playbooks.

---

## Why local-path Storage?

- local-path storage is simple and sufficient for non-critical workloads.
- The goal of this lab is orchestration and automation not building a distributed storage system.
- This keeps complexity low while still allowing persistent volumes for testing.

---

# Known Limitations

This lab intentionally accepts some trade-offs due to hardware and scope.

- Single control plane (no HA etcd quorum)
- No external load balancer
- WiFi-based networking
- Local storage only (no distributed storage)
- Not designed for zero-downtime production workloads

These limitations are acceptable for development and testing scenarios.

---

# Quick Start

For rebuilding the cluster from scratch:

1. Install Ubuntu 24.04 LTS on all nodes
2. Ensure SSH access and keys are configured
3. Update `inventory.ini` with node IPs
4. Run:
```bash
cd ansible/
ansible-playbook -i inventory.ini bootstrap.yml --ask-become-pass
ansible-playbook -i inventory.ini setup_cluster.yml
```