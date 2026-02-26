# Step 3: Set Up a Kubernetes Cluster with kubeadm

All commands run on the Pi over SSH. This guide sets up a **single control-plane node**
cluster using kubeadm — no k3s, no kubespray, everything manual.

## Architecture Overview

| Component          | Choice              | Why                                                     |
| ------------------ | ------------------- | ------------------------------------------------------- |
| Cluster bootstrap  | kubeadm             | Official Kubernetes tool, full control                   |
| Container runtime  | containerd          | Industry standard CRI runtime (installed in Step 2)      |
| CNI                | Cilium              | eBPF-based, replaces kube-proxy, modern observability    |
| Load Balancer      | MetalLB             | L2 mode — gives Services real IPs on bare metal          |
| Traffic management | Gateway API + Envoy Gateway | Successor to Ingress, role-oriented, extensible |
| Storage            | local-path-provisioner | Simple local PersistentVolumes for single-node       |
| TLS                | cert-manager        | Automated certificate management (optional)              |

## Install kubeadm, kubelet, kubectl

```bash
# Install dependencies
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# Add the Kubernetes apt repository (v1.35)
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# Pin versions so apt doesn't auto-upgrade them
sudo apt-mark hold kubelet kubeadm kubectl

# Enable kubelet (it will crash-loop until kubeadm init — that's normal)
sudo systemctl enable kubelet
```

Verify:

```bash
kubeadm version
kubectl version --client
kubelet --version
```

## Update containerd Pause Image (if needed)

kubeadm and containerd must agree on which pause (sandbox) image to use:

```bash
# See what kubeadm expects
PAUSE_IMAGE=$(kubeadm config images list 2>/dev/null | grep pause)
echo "kubeadm expects: $PAUSE_IMAGE"

# Update containerd config to match
sudo sed -i "s|sandbox_image = .*|sandbox_image = \"${PAUSE_IMAGE}\"|" /etc/containerd/config.toml
sudo systemctl restart containerd
```

## Pre-flight: Pull Images

Download all required images before initialising — avoids timeouts during init:

```bash
sudo kubeadm config images pull
```

## Initialise the Control Plane

```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=192.168.86.54 \
  --skip-phases=addon/kube-proxy
```

Flags explained:

| Flag                            | Purpose                                              |
| ------------------------------- | ---------------------------------------------------- |
| `--pod-network-cidr`            | CIDR for pod IPs — Cilium will manage this range     |
| `--apiserver-advertise-address` | Explicit IP for the API server (your static IP)      |
| `--skip-phases=addon/kube-proxy`| Don't install kube-proxy — Cilium replaces it        |

> **Replace `192.168.86.54`** with your Pi's actual static IP if you chose a different one.

**Save the output!** It contains the `kubeadm join` command you'd need if adding worker
nodes later.

## Configure kubectl

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verify the API server is reachable:

```bash
kubectl cluster-info
kubectl get nodes
```

The node will show `NotReady` — that's expected until we install the CNI.

### Set up kubectl autocompletion

```bash
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc
source ~/.bashrc
```

## Allow Scheduling on the Control Plane

By default, the control plane node has a taint that prevents workloads from being
scheduled on it. Since this is a single-node cluster, remove it:

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

## Install Helm

Several components below are installed via Helm charts:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

## Install Cilium (CNI)

Cilium provides pod networking using eBPF and replaces kube-proxy with a more
efficient eBPF-based implementation.

```bash
# Add the Cilium Helm repo
helm repo add cilium https://helm.cilium.io/
helm repo update

# Install Cilium
# - envoy.enabled=false: Cilium's embedded Envoy proxy crashes on Raspberry Pi 4
#   because the Cortex-A72 has a 39-bit virtual address space but Envoy's TCMalloc
#   assumes 48-bit. Envoy is only needed for L7 (HTTP-aware) network policies.
# - operator.replicas=1: single-node cluster, default 2 replicas can't schedule.
helm install cilium cilium/cilium \
  --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=192.168.86.54 \
  --set k8sServicePort=6443 \
  --set ipam.operator.clusterPoolIPv4PodCIDRList="10.244.0.0/16" \
  --set envoy.enabled=false \
  --set operator.replicas=1
```

> Replace `192.168.86.54` with your Pi's static IP.

Wait for Cilium to become ready:

```bash
# Watch pods (Ctrl+C when all are Running/Ready)
kubectl -n kube-system get pods -l app.kubernetes.io/part-of=cilium -w
```

After Cilium is ready, the node should become `Ready`:

```bash
kubectl get nodes
```

### Install the Cilium CLI (optional, useful for debugging)

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
GOOS=linux
GOARCH=arm64
curl -L --remote-name-all "https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-${GOOS}-${GOARCH}.tar.gz{,.sha256sum}"
sha256sum --check "cilium-${GOOS}-${GOARCH}.tar.gz.sha256sum"
sudo tar -C /usr/local/bin -xzvf "cilium-${GOOS}-${GOARCH}.tar.gz"
rm -f "cilium-${GOOS}-${GOARCH}.tar.gz" "cilium-${GOOS}-${GOARCH}.tar.gz.sha256sum"

# Run connectivity test
cilium status
```

## Install MetalLB (Bare-Metal Load Balancer)

On bare metal there's no cloud provider to hand out external IPs for `LoadBalancer`
Services. MetalLB fills that gap using Layer 2 (ARP) announcements.

### Preparation: enable strict ARP

```bash
# If you didn't skip kube-proxy, you'd need to set strictARP.
# Since we're using Cilium as kube-proxy replacement, this step is not required.
# Keeping it here for reference if you ever switch to kube-proxy.
```

### Install MetalLB

```bash
helm repo add metallb https://metallb.github.io/metallb
helm repo update

helm install metallb metallb/metallb \
  --namespace metallb-system \
  --create-namespace \
  --wait
```

Wait for pods to be ready:

```bash
kubectl -n metallb-system get pods -w
```

### Find a free IP range for MetalLB

MetalLB needs a pool of IPs it can assign to LoadBalancer Services. These must be
on your LAN subnet, unused by other devices, and outside the router's DHCP range.

**Step 1:** Scan your network to see what IPs are in use:

```bash
sudo apt install -y nmap
sudo nmap -sn 192.168.86.0/24
```

**Step 2:** Check a candidate range (e.g., .200–.210) is free:

```bash
for ip in $(seq 200 210); do
  ping -c 1 -W 1 192.168.86.$ip > /dev/null 2>&1 \
    && echo "192.168.86.$ip — IN USE" \
    || echo "192.168.86.$ip — free"
done
```

**Step 3:** Optionally check the ARP table for recently seen devices:

```bash
ip neigh show
```

If all IPs show "free" and `ip neigh` shows FAILED/INCOMPLETE (not REACHABLE) for
that range, it's safe to use.

> **Tip:** Check your router's admin page (`http://192.168.86.1`) to see its DHCP
> range. Google Wifi / Nest routers typically use `.20`–`.250` but rarely assign
> addresses above `.200` unless you have many devices. Picking `.200–.210` is
> usually safe for a home network.

### Configure IP Address Pool

```bash
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.86.200-192.168.86.210
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
    - default-pool
EOF
```

> **Adjust the IP range** to fit your network. Make sure these IPs are:
> - On the same subnet as the Pi (192.168.86.0/24)
> - Outside the router's DHCP range
> - Not used by any other device

### WiFi workaround: secondary IP for MetalLB L2

MetalLB L2 mode works by responding to ARP requests for LoadBalancer IPs. However,
**many WiFi routers (especially Google WiFi / Nest) filter ARP responses for IPs they
didn't assign via DHCP**. This means devices on your network can't reach MetalLB's
virtual IPs — ARP stays `(incomplete)` and connections time out.

The fix is to add the first MetalLB pool IP as a secondary address on the Pi's WiFi
interface. This makes the router recognise the IP as belonging to the Pi, so ARP
works normally:

```bash
CONN="$(nmcli -t -f NAME,DEVICE connection show --active | grep wlan0 | cut -d: -f1)"
sudo nmcli connection modify "$CONN" +ipv4.addresses 192.168.86.200/32
sudo nmcli device reapply wlan0
```

> **Warning:** `nmcli device reapply` may briefly drop your SSH session. Reconnect
> after a few seconds. The secondary IP persists across reboots since it's saved in
> the NetworkManager connection profile.

Verify both IPs are on the interface:

```bash
ip addr show wlan0
```

You should see both `192.168.86.54/24` and `192.168.86.200/32`.

> **Not needed on Ethernet:** If your Pi is connected via Ethernet cable, ARP works
> normally and you can skip this step. This workaround is only needed for WiFi
> connections where the router filters ARP for non-DHCP IPs.

## Install Gateway API and Nginx Gateway Fabric

The **Gateway API** is the official successor to the Kubernetes Ingress API. It provides a
role-oriented, extensible model for traffic management:

- **GatewayClass** — defines the controller implementation (managed by infra team)
- **Gateway** — a load balancer instance (managed by infra team)
- **HTTPRoute** / **GRPCRoute** — routing rules (managed by app teams)

**Nginx Gateway Fabric** (NGF) is a CNCF project that implements the Gateway API using
nginx as the data plane. We use NGF instead of Envoy Gateway because Envoy's TCMalloc
memory allocator assumes a 48-bit virtual address space, which is incompatible with the
Raspberry Pi 4's Cortex-A72 (39-bit VA space) — Envoy proxy crashes on startup.

### Install Gateway API CRDs

```bash
kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v2.4.2" | kubectl apply -f -
```

### Install Nginx Gateway Fabric

```bash
helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric \
  --create-namespace \
  -n nginx-gateway \
  --wait
```

Wait for the control plane to be ready:

```bash
kubectl -n nginx-gateway get pods -w
```

The Helm chart automatically creates a GatewayClass named `nginx`.

### Create a Gateway

When you create a Gateway referencing the `nginx` GatewayClass, NGF automatically
provisions an nginx Deployment and a LoadBalancer Service in the same namespace:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway
  namespace: default
spec:
  gatewayClassName: nginx
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
EOF
```

Wait for the Gateway to get an external IP from MetalLB and become `Programmed`:

```bash
kubectl get gateway main-gateway -w
```

You should see an IP from the MetalLB pool (e.g. `192.168.86.200`) and `PROGRAMMED: True`.

Verify the auto-provisioned nginx data plane:

```bash
kubectl get deployments
kubectl get services
```

You should see a `main-gateway-nginx` Deployment and LoadBalancer Service.

## Install local-path-provisioner (Storage)

Provides `PersistentVolume` support using local node storage. Simple and effective for
a single-node cluster:

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.30/deploy/local-path-storage.yaml
```

Make it the default StorageClass:

```bash
kubectl patch storageclass local-path \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

Verify:

```bash
kubectl get storageclass
```

Should show `local-path (default)`.

## Verify the Cluster

### Deploy a test application

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
        - name: echo
          image: ealen/echo-server:latest
          ports:
            - containerPort: 80
          env:
            - name: PORT
              value: "80"
---
apiVersion: v1
kind: Service
metadata:
  name: echo
  namespace: default
spec:
  selector:
    app: echo
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: echo
  namespace: default
spec:
  parentRefs:
    - name: main-gateway
  hostnames:
    - "echo.rara.local"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: echo
          port: 80
EOF
```

### Test the deployment

```bash
# Check pods are running
kubectl get pods -l app=echo

# Check the Service
kubectl get svc echo

# Get the Gateway's external IP
GATEWAY_IP=$(kubectl get gateway main-gateway -o jsonpath='{.status.addresses[0].value}')
echo "Gateway IP: $GATEWAY_IP"

# Test via curl with Host header
curl -s -H "Host: echo.rara.local" http://$GATEWAY_IP/ | jq .

# Test directly via the Service (ClusterIP)
kubectl run tmp --image=curlimages/curl --rm -it --restart=Never -- \
  curl -s http://echo.default.svc.cluster.local/ | jq .
```

### Access from your Mac

To access `echo.rara.local` from your Mac, add an entry to `/etc/hosts`:

```bash
# On your Mac (not the Pi), run:
echo "192.168.86.200  echo.rara.local" | sudo tee -a /etc/hosts
```

Then: `curl http://echo.rara.local/`

> Replace `192.168.86.200` with the actual Gateway IP from the command above.

### DNS verification (on the Pi)

```bash
kubectl apply -f https://k8s.io/examples/admin/dns/dnsutils.yaml
kubectl wait --for=condition=Ready pod/dnsutils --timeout=60s
kubectl exec dnsutils -- nslookup kubernetes.default
kubectl exec dnsutils -- nslookup echo.default.svc.cluster.local
kubectl delete pod dnsutils
```

## Cluster Health Check

```bash
# All nodes Ready
kubectl get nodes -o wide

# All system pods running
kubectl get pods -n kube-system

# Cilium status
kubectl -n kube-system get pods -l app.kubernetes.io/part-of=cilium

# MetalLB status
kubectl -n metallb-system get pods

# Nginx Gateway Fabric status
kubectl -n nginx-gateway get pods

# Gateway has an address and is Programmed
kubectl get gateway main-gateway

# All component statuses
kubectl get componentstatuses 2>/dev/null || kubectl get --raw='/readyz?verbose'
```

## Optional: Install cert-manager (TLS Certificates)

cert-manager automates issuing and renewing TLS certificates. Useful if you want
HTTPS on your Gateway.

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true \
  --wait
```

Create a self-signed ClusterIssuer for testing:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned
spec:
  selfSigned: {}
EOF
```

> For production-like setup with Let's Encrypt, you'd need a domain pointing to your
> Pi's public IP. For a home lab, self-signed certificates are fine.

## Optional: Add a Worker Node

If you have a second Pi, prepare it with Steps 1 and 2, then install kubeadm + kubelet
(not kubectl) and join it:

```bash
# On the control plane — generate a new join token
kubeadm token create --print-join-command

# On the worker — run the output from above, e.g.:
# sudo kubeadm join 192.168.86.54:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

## Access the Cluster from Your Mac

Copy the kubeconfig from the Pi to your Mac:

```bash
scp ra@rara.local:~/.kube/config ~/.kube/config-rpi
```

The default kubeconfig uses generic names (`kubernetes`, `kubernetes-admin`) that will
collide with other clusters in your `~/.kube/config`. Rename them to something unique:

```bash
# Rename context, cluster, and user to avoid merge conflicts
kubectl --kubeconfig ~/.kube/config-rpi config rename-context kubernetes-admin@kubernetes rpi
sed -i '' 's/cluster: kubernetes/cluster: rpi/' ~/.kube/config-rpi
sed -i '' 's/name: kubernetes$/name: rpi/' ~/.kube/config-rpi
sed -i '' 's/user: kubernetes-admin/user: rpi-admin/' ~/.kube/config-rpi
sed -i '' 's/name: kubernetes-admin$/name: rpi-admin/' ~/.kube/config-rpi
```

Verify it works in isolation:

```bash
kubectl --kubeconfig ~/.kube/config-rpi get nodes
```

Merge with your existing kubeconfig and switch context:

```bash
export KUBECONFIG=~/.kube/config:~/.kube/config-rpi
kubectl config use-context rpi
kubectl get nodes
```

> **Make it permanent:** Add the `export KUBECONFIG=...` line to your `~/.zshrc` so it
> persists across terminal sessions.

## Clean Up Test Resources

```bash
kubectl delete httproute echo
kubectl delete svc echo
kubectl delete deployment echo
```

## Troubleshooting

### Node stuck in NotReady

```bash
# Check kubelet logs
sudo journalctl -u kubelet -f --no-pager | tail -50

# Check if CNI is running
kubectl -n kube-system get pods -l app.kubernetes.io/part-of=cilium
```

### Pods stuck in Pending

```bash
kubectl describe pod <pod-name>
# Look at Events section — usually a scheduling or resource issue
```

### Pods stuck in ContainerCreating

```bash
# Usually a CNI issue
kubectl describe pod <pod-name>
sudo crictl pods
sudo crictl ps -a
```

### Can't pull images

```bash
# Test containerd can pull
sudo ctr image pull docker.io/library/nginx:latest

# Check DNS from the node
nslookup registry.k8s.io
```

### Gateway has no address

```bash
# Check MetalLB is running
kubectl -n metallb-system get pods
kubectl -n metallb-system logs -l app.kubernetes.io/name=metallb

# Check IPAddressPool exists
kubectl -n metallb-system get ipaddresspool
kubectl -n metallb-system get l2advertisement
```

### Reset and start over

If something goes badly wrong:

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d
sudo rm -rf $HOME/.kube
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F
sudo iptables -X && sudo iptables -t nat -X && sudo iptables -t mangle -X

# Then re-run kubeadm init from the beginning
```

## Useful Commands Reference

```bash
# Cluster info
kubectl cluster-info
kubectl get nodes -o wide
kubectl get all --all-namespaces

# Watch events in real time
kubectl get events --sort-by='.lastTimestamp' -w

# Resource usage (requires metrics-server)
kubectl top nodes
kubectl top pods --all-namespaces

# Describe everything about a resource
kubectl describe node rara
kubectl describe pod <name> -n <namespace>

# Exec into a pod
kubectl exec -it <pod-name> -- /bin/sh

# Port-forward a service to localhost
kubectl port-forward svc/<name> 8080:80

# View logs
kubectl logs <pod-name> -n <namespace> -f

# Check what kubeadm set up
sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep -E 'image:|--'
```