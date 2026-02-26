## Home k8s cluster based on bare-metal Raspberry Pi

### Guides

1. **[01-install-os.md](01-install-os.md)** — Flash Raspberry Pi OS, configure hostname/WiFi/SSH
2. **[02-prepare-linux.md](02-prepare-linux.md)** — Static IP, swap, kernel modules, containerd, firewall
3. **[03-setup-kubernetes.md](03-setup-kubernetes.md)** — kubeadm cluster, Cilium, MetalLB, Nginx Gateway Fabric
4. **[04-setup-gitops.md](04-setup-gitops.md)** — Flux CD GitOps for declarative cluster management

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Raspberry Pi 4 (rara)                 │
│                   192.168.86.54                         │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │              Kubernetes v1.35                    │    │
│  │                                                  │    │
│  │  Control Plane:                                  │    │
│  │    ├── kube-apiserver                            │    │
│  │    ├── kube-controller-manager                   │    │
│  │    ├── kube-scheduler                            │    │
│  │    └── etcd                                      │    │
│  │                                                  │    │
│  │  Networking:                                     │    │
│  │    ├── Cilium (CNI + kube-proxy replacement)     │    │
│  │    └── MetalLB (L2 LoadBalancer)                 │    │
│  │                                                  │    │
│  │  Traffic:                                        │    │
│  │    ├── Gateway API CRDs                          │    │
│  │    └── Nginx Gateway Fabric (Gateway impl.)      │    │
│  │                                                  │    │
│  │  Storage:                                        │    │
│  │    └── local-path-provisioner                    │    │
│  │                                                  │    │
│  │  GitOps:                                         │    │
│  │    └── Flux CD (reconciles from GitHub)           │    │
│  │                                                  │    │
│  │  Optional:                                       │    │
│  │    └── cert-manager (TLS)                        │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  MetalLB pool: 192.168.86.200–210                       │
│  Pod CIDR:     10.244.0.0/16                            │
│  Service CIDR: 10.96.0.0/12 (default)                   │
└─────────────────────────────────────────────────────────┘
```
