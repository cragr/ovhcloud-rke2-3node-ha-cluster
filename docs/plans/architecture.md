# Architecture: 3-Node HA RKE2 Cluster on OVHcloud

## Overview

Three OVHcloud VPS instances form a high-availability RKE2 Kubernetes cluster. All nodes run as control-plane + etcd. Cloudflare proxies public HTTP/HTTPS traffic while UFW restricts direct access to Cloudflare IP ranges and trusted sources.

```
                       Internet
                          |
                          v
                +-----------------------+
                |   Cloudflare Proxy    |
                |   (TLS, DDoS, DNS)    |
                +-----------+-----------+
                            |
             Only Cloudflare IPs allowed (UFW)
                            |
           +----------------+----------------+
           v                v                v
     +-----------+    +-----------+    +-----------+
     |   node1   |    |   node2   |    |   node3   |
     |  (server) |    |  (server) |    |  (server) |
     |           |    |           |    |           |
     |  RKE2     |    |  RKE2     |    |  RKE2     |
     |  server   |    |  server   |    |  server   |
     |  + etcd   |    |  + etcd   |    |  + etcd   |
     +-----------+    +-----------+    +-----------+
           |                |                |
           +----------------+----------------+
                  Public network (no vRack)
                  Canal CNI
                  (enable WireGuard for encryption)
```

## Deployment Stages

| Playbook | Target | Purpose |
|----------|--------|---------|
| 00-bootstrap | all (as ubuntu) | Create ansible user, SSH key, passwordless sudo |
| 01-harden | all | UFW, fail2ban, sysctl, SSH hardening, unattended-upgrades |
| 02-rke2-server | all servers | Install RKE2 on first server, then join additional servers |
| 03-storage | all + localhost | LVM volume group on each node + OpenEBS LVM LocalPV provisioner |

## Node Hardening

### SSH

- Root login disabled
- Password authentication disabled (key-only)
- Access limited to the ansible user via `AllowUsers`

### Firewall (UFW)

```
Default: deny incoming, allow outgoing

22/tcp                  SSH (from anywhere, protected by fail2ban)
6443/tcp from <IPs>     Kubernetes API (IP allowlist)
9345/tcp from nodes     RKE2 supervisor (inter-node)
10250/tcp from nodes    Kubelet (inter-node)
8472/udp from nodes     Flannel VXLAN (inter-node)
2379-2380/tcp from nodes  etcd (inter-node)
80/tcp from Cloudflare  HTTP (Cloudflare IP ranges only)
443/tcp from Cloudflare HTTPS (Cloudflare IP ranges only)
```

### System

- **fail2ban** protects SSH against brute-force attacks
- **unattended-upgrades** applies security patches automatically
- **sysctl** tunes kernel network parameters
- Required kernel modules: `br_netfilter`, `overlay`

## RKE2 Cluster

### Configuration

The first server initializes the cluster. Additional servers join via the first server's IP on port 9345.

All servers share the same TLS SANs list (node IPs, hostnames, API endpoint) so the API certificate is valid regardless of which node handles a request.

### Key Decisions

- **All-server HA**: Three control-plane nodes with embedded etcd provide fault tolerance for any single node failure
- **Canal CNI**: Pod networking over the public network (no vRack). Enable WireGuard post-install to encrypt pod-to-pod traffic.
- **nftables kube-proxy**: Native nftables mode (GA in Kubernetes 1.33+)
- **No node taints**: All servers run workloads
- **No VIP**: Cloudflare handles external failover via health checks on proxied DNS records

## Storage

### LVM + OpenEBS LVM LocalPV

Each node has a secondary SSD configured as an LVM volume group. The OpenEBS LVM LocalPV provisioner runs on the cluster and dynamically creates logical volumes from the volume group when PersistentVolumeClaims are created.

```
/dev/sdb (secondary SSD)
+-- LVM: vg_data (volume group)
    +-- LVs created dynamically by OpenEBS LVM CSI driver
```

The `openebs-lvm` StorageClass is set as the cluster default with `WaitForFirstConsumer` binding mode, which ensures volumes are created on the same node as the consuming pod.

Playbook `03-storage.yml` handles both stages:
1. Creates the LVM VG on each node's secondary disk
2. Installs the OpenEBS LVM LocalPV Helm chart and creates the default StorageClass

## DNS Strategy

Split infrastructure and application DNS across two domains:

| Record | Purpose | Cloudflare Mode |
|--------|---------|-----------------|
| `node[1-3].<infra-domain>` | Node hostnames | DNS-only (grey cloud) |
| `k8s.<infra-domain>` | Kubernetes API | DNS-only (grey cloud) |
| `app.<public-domain>` | Application traffic | Proxied (orange cloud) |

Separating domains prevents DNS enumeration from exposing origin IPs used by proxied services.

## Secrets Management

1. Copy `vars/secrets.yml.example` to `vars/secrets.yml`
2. Fill in secret values (RKE2 cluster token, etc.)
3. Encrypt: `ansible-vault encrypt vars/secrets.yml`
4. Run playbooks with `--ask-vault-pass`
