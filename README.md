# 3-Node HA RKE2 Cluster on OVHcloud

Ansible playbooks for deploying a 3-node HA RKE2 Kubernetes cluster on OVHcloud VPS instances.

## Architecture

Three OVHcloud VPS instances running Ubuntu 24.04, each acting as an RKE2 control-plane node with embedded etcd. Cloudflare proxies public traffic while UFW restricts access to Cloudflare IP ranges and trusted sources.

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

- **HA**: Any single node can fail without losing the cluster
- **Security**: SSH hardening, fail2ban, UFW firewall, unattended upgrades
- **Storage**: OpenEBS LVM LocalPV on each node's secondary SSD (default StorageClass)
- **Networking**: Canal CNI (WireGuard can be enabled post-install to encrypt pod traffic over the public network)

Tested with OVHcloud VPS-1 (4 CPU, 8 GB RAM, 75 GB NVMe + 50 GB additional SSD) at $7.35/month per node — $22.05/month total for three nodes with no commitment period.

See [docs/plans/architecture.md](docs/plans/architecture.md) for full details.

## Prerequisites

- Ansible 2.15+
- Python 3.10+
- SSH key pair (`~/.ssh/id_ed25519`)
- 3 OVHcloud VPS instances running Ubuntu 24.04

## Quick Start

### 1. Prepare the VPS nodes

After provisioning each OVHcloud VPS, SSH in as the default `ubuntu` user to set a password (OVHcloud requires this on first login):

```bash
ssh ubuntu@<VPS_IP>
# Set a new password when prompted, then exit
```

Copy your SSH key to each node so Ansible can connect:

```bash
ssh-copy-id ubuntu@<NODE1_IP>
ssh-copy-id ubuntu@<NODE2_IP>
ssh-copy-id ubuntu@<NODE3_IP>
```

### 2. Install Ansible collections

```bash
ansible-galaxy collection install -r requirements.yml
```

### 3. Configure your inventory

Replace all `<PLACEHOLDER>` values:

```bash
# Edit inventory/hosts.yml with your node IPs and hostnames
# Edit inventory/group_vars/all.yml with your cluster settings
```

### 4. Create and encrypt secrets

```bash
cp vars/secrets.yml.example vars/secrets.yml
# Edit vars/secrets.yml with your values
ansible-vault encrypt vars/secrets.yml
```

### 5. Bootstrap nodes

This creates the `ansible` user with your SSH key and passwordless sudo. Run once per node:

```bash
ansible-playbook playbooks/00-bootstrap.yml -u ubuntu --ask-become-pass
```

### 6. Deploy the cluster

```bash
ansible-playbook playbooks/site.yml --ask-vault-pass
```

## Playbooks

| Playbook | Description |
|----------|-------------|
| 00-bootstrap.yml | Create ansible user with SSH key |
| 01-harden.yml | UFW, fail2ban, SSH hardening |
| 02-rke2-server.yml | Install 3-node HA RKE2 cluster (all control-plane) |
| 03-storage.yml | LVM + OpenEBS LVM LocalPV provisioner on secondary SSD |
| site.yml | Run all playbooks (except bootstrap) |

## Managing Kubernetes API Access

Add your IP to access kubectl:

```bash
ansible-playbook playbooks/ufw-allow-ip.yml -e "ip=YOUR_IP action=allow"
```

Remove access:

```bash
ansible-playbook playbooks/ufw-allow-ip.yml -e "ip=YOUR_IP action=delete"
```

## Kubeconfig

After running `02-rke2-server.yml`, kubeconfig is saved to `./kubeconfig`:

```bash
export KUBECONFIG=$(pwd)/kubeconfig
kubectl get nodes
```

## Rancher Management

To manage the cluster with Rancher, see [docs/rancher-quadlet.md](docs/rancher-quadlet.md) for running Rancher in a Podman quadlet container on CentOS 10 Stream.
