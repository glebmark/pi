# Step 2: Prepare Linux for Kubernetes

Everything here runs on the Pi over SSH (`ssh ra@rara.local`).

## Update the System

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt autoremove -y
sudo reboot
```

Reconnect after reboot:

```bash
ssh ra@rara.local
```

## Set a Static IP Address

Kubernetes binds certificates to the node's IP address. If the IP changes (DHCP lease
renewal), the API server certificate becomes invalid and the cluster breaks. A static IP
is effectively required.

Check the current network details:

```bash
nmcli dev show wlan0 | grep -E 'IP4\.(ADDRESS|GATEWAY|DNS)'
```

Pick an IP outside your router's DHCP range to avoid conflicts. Most routers assign
DHCP addresses in a range like `.20`–`.199`. Check your router's admin page to confirm.

Find the active WiFi connection name (the Imager names it differently across OS versions):

```bash
# Show all connections — note the NAME of the wifi connection on wlan0
nmcli connection show
```

Common names: `preconfigured`, `Wired connection 1`, or your SSID (e.g. `hehe`).

Apply static IP config via NetworkManager — replace `CONN_NAME` with the actual name:

```bash
CONN="$(nmcli -t -f NAME,DEVICE connection show --active | grep wlan0 | cut -d: -f1)"
echo "Active WiFi connection: $CONN"

sudo nmcli connection modify "$CONN" \
  ipv4.method manual \
  ipv4.addresses 192.168.86.54/24 \
  ipv4.gateway 192.168.86.1 \
  ipv4.dns "1.1.1.1,8.8.8.8"

# Reapply without dropping the connection (safe over SSH)
sudo nmcli device reapply wlan0
```

> **Warning:** Do NOT use `nmcli connection down` over SSH — it kills WiFi and your
> session dies before `connection up` can run. Use `device reapply` instead, which
> applies the new settings in-place without disconnecting.
>
> **Adjust the IP values above to match your network.** The IP `192.168.86.54` is what the Pi
> currently has — keeping the same address avoids disrupting your SSH session, but make sure
> it's outside your router's DHCP pool.

Verify:

```bash
ip addr show wlan0
ping -c 2 google.com
```

**Alternative:** Instead of a static IP on the Pi, you can set a **DHCP reservation** on
your router (assign a fixed IP to the Pi's MAC address `d8:3a:dd:66:ec:3e`). Either
approach works — the goal is a stable IP.

## Disable Swap

Kubernetes requires swap to be completely off. Check what's active:

```bash
free -h | grep Swap
swapon --show
```

Raspberry Pi OS Trixie uses **zram swap** by default (compressed RAM-based swap),
not the old `dphys-swapfile`. Disable all swap sources:

```bash
# Turn off all swap immediately
sudo swapoff -a

# Disable zram swap (Trixie default)
sudo systemctl stop zramswap.service 2>/dev/null || true
sudo systemctl disable zramswap.service 2>/dev/null || true
sudo systemctl stop systemd-zram-setup@zram0.service 2>/dev/null || true
sudo systemctl mask systemd-zram-setup@zram0.service 2>/dev/null || true

# Disable the swap target so systemd doesn't reactivate swap
sudo systemctl mask swap.target

# Disable dphys-swapfile if present (older RPi OS)
sudo systemctl disable dphys-swapfile.service 2>/dev/null || true
sudo apt purge dphys-swapfile -y 2>/dev/null || true

# Comment out any swap entries in fstab
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

Reboot and verify — the Swap line must show all zeros:

```bash
sudo reboot
# after reconnecting:
free -h | grep Swap
swapon --show
```

Expected: `Swap: 0B 0B 0B` and no output from `swapon --show`.

## Load Required Kernel Modules

Kubernetes networking needs `overlay` (for container filesystems) and `br_netfilter`
(for iptables to see bridged traffic):

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Verify
lsmod | grep -E 'overlay|br_netfilter'
```

## Configure Kernel Network Parameters

Enable IP forwarding and allow iptables to process bridged traffic:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

Verify:

```bash
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

All three should return `= 1`.

## Verify Cgroup Configuration

Modern Raspberry Pi OS (Trixie) uses **cgroups v2** by default, which Kubernetes 1.35
supports natively. Verify:

```bash
stat -fc %T /sys/fs/cgroup/
```

- `cgroup2fs` — cgroups v2 (good).
- `tmpfs` — cgroups v1. Add `cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1`
  to `/boot/firmware/cmdline.txt` (same line, appended) and reboot.

### Enable the memory cgroup controller

Raspberry Pi OS disables the `memory` controller by default to save overhead.
Kubernetes needs it for resource limits and pod eviction. Check:

```bash
cat /sys/fs/cgroup/cgroup.controllers
```

If the output does **not** include `memory` (e.g. you see `cpuset cpu io pids` only),
enable it:

```bash
sudo sed -i 's/$/ cgroup_enable=memory/' /boot/firmware/cmdline.txt

# Verify the line looks correct (should be a single line with the param appended)
cat /boot/firmware/cmdline.txt

sudo reboot
```

After reboot, verify again:

```bash
cat /sys/fs/cgroup/cgroup.controllers
```

Should now include `memory`: `cpuset cpu io memory pids`.

## Install containerd

containerd is the container runtime that Kubernetes uses to run containers. Install it
from Docker's official apt repository (provides newer versions than Debian's own repos):

```bash
# Install prerequisites
sudo apt-get install -y ca-certificates curl gnupg

# Add Docker's GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install containerd
sudo apt-get update
sudo apt-get install -y containerd.io
```

> **If the Trixie codename isn't available in Docker's repo yet**, replace
> `$VERSION_CODENAME` with `bookworm` in the echo command above. The packages
> are binary-compatible.

## Configure containerd

Generate the default config and enable the systemd cgroup driver (required by Kubernetes):

```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```

Edit the config to set `SystemdCgroup = true`:

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

Verify the change:

```bash
grep SystemdCgroup /etc/containerd/config.toml
```

Should output: `SystemdCgroup = true`

Optionally update the sandbox (pause) image to match what kubeadm expects:

```bash
# Check what kubeadm will use
PAUSE_IMAGE=$(sudo kubeadm config images list 2>/dev/null | grep pause || echo "registry.k8s.io/pause:3.10")
echo "Pause image: $PAUSE_IMAGE"

# You can update this after installing kubeadm in Step 3.
# For now, the default is usually fine.
```

Restart and enable containerd:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd
```

Verify it's working:

```bash
sudo ctr version
```

## Install crictl (optional but recommended)

`crictl` is a CLI tool for interacting with CRI-compatible container runtimes.
Useful for debugging:

```bash
# Check latest version at https://github.com/kubernetes-sigs/cri-tools/releases
CRICTL_VERSION="v1.35.0"
ARCH="arm64"
curl -L "https://github.com/kubernetes-sigs/cri-tools/releases/download/${CRICTL_VERSION}/crictl-${CRICTL_VERSION}-linux-${ARCH}.tar.gz" | \
  sudo tar -C /usr/local/bin -xz

# Configure crictl to use containerd
cat <<EOF | sudo tee /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
EOF

# Verify
sudo crictl info
```

> Adjust `CRICTL_VERSION` to match your Kubernetes minor version.

## Configure Firewall

Set up `ufw` and open the ports required by Kubernetes. See
[official port list](https://kubernetes.io/docs/reference/networking/ports-and-protocols/).

```bash
sudo apt install -y ufw

# Default policy: deny incoming, allow outgoing
sudo ufw default deny incoming
sudo ufw default allow outgoing

# SSH (keep this first so you don't lock yourself out!)
sudo ufw allow 22/tcp comment 'SSH'

# Kubernetes API server
sudo ufw allow 6443/tcp comment 'K8s API server'

# etcd
sudo ufw allow 2379:2380/tcp comment 'etcd'

# Kubelet API
sudo ufw allow 10250/tcp comment 'Kubelet API'

# kube-scheduler
sudo ufw allow 10259/tcp comment 'kube-scheduler'

# kube-controller-manager
sudo ufw allow 10257/tcp comment 'kube-controller-manager'

# NodePort range (for Services of type NodePort)
sudo ufw allow 30000:32767/tcp comment 'NodePort range'

# Cilium requirements
sudo ufw allow 4240/tcp comment 'Cilium health'
sudo ufw allow 4244/tcp comment 'Cilium Hubble'
sudo ufw allow 8472/udp comment 'Cilium VXLAN'
sudo ufw allow 4245/tcp comment 'Cilium Hubble Relay'

# MetalLB memberlist
sudo ufw allow 7946/tcp comment 'MetalLB memberlist'
sudo ufw allow 7946/udp comment 'MetalLB memberlist'

# Gateway traffic (HTTP/HTTPS via MetalLB LoadBalancer IPs)
sudo ufw allow 80/tcp comment 'HTTP Gateway'
sudo ufw allow 443/tcp comment 'HTTPS Gateway'

# Allow routed/forwarded traffic — essential for MetalLB + Cilium to forward
# external traffic arriving at LoadBalancer IPs to pods inside the cluster
sudo ufw default allow routed

# Enable the firewall
sudo ufw enable
sudo ufw status verbose
```

> **Single-node cluster on a home network:** If you're the only user on the network,
> you can skip ufw entirely. The router's NAT already blocks incoming traffic from the
> internet. Firewall is more important if the Pi is on a shared or public network.

## Configure Time Synchronisation

Accurate time is critical for certificate validation and etcd:

```bash
# Check if timesyncd is running
timedatectl status

# If NTP is not active, enable it
sudo timedatectl set-ntp true

# Verify
timedatectl show --property=NTPSynchronized
```

Should show `NTPSynchronized=yes`.

## Install Useful Tools

```bash
sudo apt install -y \
  bash-completion \
  dnsutils \
  jq \
  net-tools \
  socat \
  conntrack \
  ipset \
  ethtool
```

These are used by Kubernetes components and are helpful for debugging.

## Final Reboot and Verification

```bash
sudo reboot
```

After reboot, reconnect and verify everything persisted:

```bash
ssh ra@rara.local

# Swap is off
free -h | grep Swap

# Kernel modules are loaded
lsmod | grep -E 'overlay|br_netfilter'

# Sysctl settings are applied
sysctl net.ipv4.ip_forward

# containerd is running
sudo systemctl is-active containerd

# Static IP is set
ip addr show wlan0

# Firewall is active (if enabled)
sudo ufw status
```

## Checklist

| Requirement                       | How to verify                                      | Expected         |
| --------------------------------- | -------------------------------------------------- | ---------------- |
| Static IP                         | `ip addr show wlan0`                               | 192.168.86.54/24 |
| Swap disabled                     | `free -h`                                          | Swap: 0B         |
| overlay module                    | `lsmod \| grep overlay`                            | loaded           |
| br_netfilter module               | `lsmod \| grep br_netfilter`                       | loaded           |
| IP forwarding                     | `sysctl net.ipv4.ip_forward`                       | = 1              |
| bridge-nf-call-iptables           | `sysctl net.bridge.bridge-nf-call-iptables`        | = 1              |
| cgroups v2                        | `stat -fc %T /sys/fs/cgroup/`                      | cgroup2fs        |
| containerd running                | `sudo systemctl is-active containerd`              | active           |
| containerd SystemdCgroup          | `grep SystemdCgroup /etc/containerd/config.toml`   | true             |
| Time synced                       | `timedatectl show --property=NTPSynchronized`      | yes              |
| Internet access                   | `ping -c 2 google.com`                             | success          |

---

**Next:** [03-setup-kubernetes.md](./03-setup-kubernetes.md) — install and configure the Kubernetes cluster.
