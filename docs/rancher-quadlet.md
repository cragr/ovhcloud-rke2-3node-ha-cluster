# Running Rancher in a Quadlet Container on CentOS 10 Stream

This guide covers running Rancher as a rootless Podman quadlet container on a CentOS 10 Stream host for managing the RKE2 cluster.

## Architecture

Rancher runs behind an Nginx Proxy Manager (NPM Plus) container that handles SSL termination:

```
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│  Downstream  │       │   Reverse    │       │   Rancher    │
│  Agent       │──TLS──▶   Proxy      │──HTTP──▶  Container  │
│              │       │  (SSL term)  │       │  :8181/8282  │
└──────────────┘       └──────────────┘       └──────────────┘
```

- **Reverse Proxy (NPM Plus)**: Terminates TLS, handles certificates via Let's Encrypt
- **Rancher Container**: Runs with `--no-cacerts` since SSL is handled externally
- **Downstream Agents**: Connect to Rancher via the proxy's public HTTPS endpoint

This setup allows proper certificate management without Rancher's built-in CA, which simplifies cluster agent communication.

## Prerequisites

- CentOS 10 Stream host
- Podman installed
- Network access to the RKE2 cluster API (port 6443)

## Host Configuration

### Load Required Kernel Modules

CentOS 10 Stream requires additional kernel modules for container networking:

```bash
# Load the required modules
sudo modprobe iptable_nat
sudo modprobe iptable_filter
sudo modprobe iptable_mangle
sudo modprobe br_netfilter
sudo modprobe overlay
sudo modprobe ip_tables

# Verify they loaded
lsmod | grep -E "iptable_nat|br_netfilter|overlay"
```

### Persist Kernel Modules Across Reboots

```bash
sudo tee /etc/modules-load.d/k3s-rancher.conf <<EOF
iptable_nat
iptable_filter
iptable_mangle
br_netfilter
overlay
ip_tables
EOF
```

### Set Required Sysctl Parameters

```bash
sudo tee /etc/sysctl.d/90-k3s-rancher.conf <<EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

## Quadlet Container Setup

### Create Data Directory

```bash
sudo mkdir -p /opt/containers/rancher
sudo chown $USER:$USER /opt/containers/rancher
```

### Create the Quadlet File

Create the container definition at `~/.config/containers/systemd/rancher.container`:

```ini
[Unit]
Description=Rancher container
After=network-online.target
Wants=network-online.target

[Container]
ContainerName=rancher
Image=docker.io/rancher/rancher:stable

# --- Security ---
# Required: Rancher runs K3s inside the container (containers-in-containers)
SecurityLabelDisable=true
PodmanArgs=--privileged
PodmanArgs=--cgroupns=host

# --- Networking ---
# Host networking avoids Podman bridge IP churn that corrupts K3s etcd endpoints
Network=host

# --- Mounts ---
# cgroup v2 filesystem — K3s needs read-write access with shared propagation
Mount=type=bind,src=/sys/fs/cgroup,dst=/sys/fs/cgroup,bind-propagation=rshared

# K3s runtime directories
Tmpfs=/run
Tmpfs=/var/run

# Persist Rancher data (includes embedded k3s/etcd)
Volume=/opt/containers/rancher:/var/lib/rancher:Z

# --- Custom Ports ---
# Passed as arguments to the Rancher entrypoint
Exec=--http-listen-port=8181 --https-listen-port=8282 --no-cacerts

[Service]
Restart=always
TimeoutStartSec=900

[Install]
WantedBy=default.target
```

### Start the Container

```bash
# Reload systemd to pick up the new quadlet
systemctl --user daemon-reload

# Start Rancher
systemctl --user start rancher

# Enable on boot
systemctl --user enable rancher

# Check status
systemctl --user status rancher
```

### View Logs

```bash
podman logs -f rancher
```

## Accessing Rancher

1. Open `https://<host-ip>:8282` in your browser
2. Accept the self-signed certificate warning (using `--no-cacerts`)
3. Retrieve the bootstrap password:
   ```bash
   podman logs rancher 2>&1 | grep "Bootstrap Password:"
   ```
4. Set the admin password and configure the Rancher server URL

## Import the RKE2 Cluster

1. In Rancher, go to **Cluster Management** → **Import Existing**
2. Select **Generic**
3. Enter a cluster name (e.g., `my-cluster`)
4. Copy the provided kubectl command
5. Run it against your RKE2 cluster:

```bash
export KUBECONFIG=./kubeconfig
kubectl apply -f <rancher-import-manifest-url>
```

6. Since we're using `--no-cacerts`, remove strict CA verification from the agent:

```bash
kubectl -n cattle-system set env deployment/cattle-cluster-agent \
  STRICT_VERIFY- \
  CATTLE_CA_CHECKSUM-
```

This allows the cluster agent to communicate with Rancher without CA certificate validation.

## Firewall Configuration

Ensure the CentOS host can reach the RKE2 cluster:

```bash
# On the RKE2 nodes, allow the Rancher host IP
ansible-playbook playbooks/ufw-allow-ip.yml -e "ip=<rancher-host-ip> action=allow"
```

## Troubleshooting

### Container Won't Start

Check for missing kernel modules:

```bash
dmesg | grep -i "module"
lsmod | grep -E "iptable|bridge|overlay"
```

### Network Issues

Verify sysctl settings are applied:

```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.ipv4.ip_forward
```

### Permission Denied Errors

For rootless Podman, ensure user lingering is enabled:

```bash
loginctl enable-linger $USER
```
