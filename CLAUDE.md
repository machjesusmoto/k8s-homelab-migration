# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Kubernetes homelab infrastructure deployment project using Talos Linux - an immutable, API-driven operating system designed specifically for Kubernetes. The project deploys a highly available 5-node cluster (3 control plane + 2 worker nodes) on Proxmox VMs.

## Key Commands

### Cluster Deployment Workflow

```bash
# 1. Build custom Talos image with QEMU guest agent
./scripts/build-talos-image.sh

# 2. Generate Talos configurations (creates secrets.yaml - MUST backup!)
./scripts/generate-configs.sh

# 3. Apply configurations to all nodes
./scripts/apply-configs.sh

# 4. Bootstrap the Kubernetes cluster
./scripts/bootstrap-cluster.sh

# 5. Verify cluster health
talosctl health
kubectl get nodes -o wide
```

### Common Development Commands

```bash
# Talos cluster management
talosctl config merge talos/talosconfig
talosctl config endpoints 192.168.1.241 192.168.1.242 192.168.1.243
talosctl config nodes 192.168.1.241 192.168.1.242 192.168.1.243
talosctl kubeconfig
talosctl logs -n <node-ip>
talosctl dmesg -n <node-ip>

# Kubernetes operations
kubectl apply -k kubernetes/core/        # Deploy core infrastructure
kubectl apply -k kubernetes/gitops/      # Deploy GitOps controllers
kubectl apply -k kubernetes/apps/        # Deploy applications
kubectl get pods -A -w                   # Watch all pods

# Execute commands across cluster nodes (legacy from K3s)
./scripts/cluster-exec.sh <cluster> "command"
```

### Testing and Verification

There is no formal testing framework. Verification is done through:
- `talosctl health` - Check Talos cluster health
- `kubectl get nodes` - Verify all nodes are Ready
- Manual verification steps in deployment scripts

## Architecture and Structure

### Network Configuration
- **Cluster VIP**: 192.168.1.240 (Kubernetes API endpoint)
- **Control Planes**: 192.168.1.241-243 (talos-cp-01 to talos-cp-03)
- **Workers**: 192.168.1.244-245 (talos-worker-01 to talos-worker-02)
- **VLAN**: 1200 for all nodes

### Directory Structure
```
/talos/                    # Talos configurations
  /patches/               # Node-specific and common patches
    common.yaml          # Shared settings (DNS, NTP, CNI)
    controlplane.yaml    # Control plane specific (VIP, etcd)
    cp-0X.yaml          # Individual control plane configs
    worker-0X.yaml      # Individual worker configs
  schematic.yaml        # Custom Talos image definition
  
/kubernetes/             # Kubernetes manifests
  /core/                # Core infrastructure (namespaces, CSI, LB, ingress)
  /apps/                # Application deployments (*arr stack, monitoring)
  /gitops/              # GitOps controller configurations
  
/scripts/               # Deployment automation
/docs/                  # Documentation
```

### Key Architectural Decisions

1. **Talos Linux**: Immutable OS with no SSH access - all management via API
2. **High Availability**: 3 control plane nodes with shared VIP
3. **Storage**: NFS from external TrueNAS Scale server
4. **GitOps**: Designed for Flux/ArgoCD deployment patterns
5. **External Services**: Plex runs on dedicated VM, not in Kubernetes

### Important Files to Understand

1. **secrets.yaml** (generated, not in git) - Contains all cluster secrets, critical to backup
2. **talos/patches/common.yaml** - Defines cluster-wide settings
3. **MIGRATION_TRACKER.md** - Deployment checklist with 8 phases
4. **kubernetes/core/namespaces/namespaces.yaml** - Defines all namespaces

### Security Considerations

- Talos has no SSH/shell access - only API with mTLS authentication
- All configuration is declarative via YAML
- Never commit secrets.yaml to git
- Use talosctl for all node interactions

### Common Pitfalls

1. Always apply patches when configuring nodes (use apply-configs.sh)
2. Bootstrap only runs on first control plane node
3. Wait for all nodes to be Ready before deploying workloads
4. Talos nodes need proper DNS resolution for cluster operations