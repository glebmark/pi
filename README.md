## Home k8s cluster based on bare-metal Raspberry Pi

```
┌─────────────────────────────────────────────────────────┐
│                   Raspberry Pi 4 (rara)                 │
│                   192.168.86.54                         │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │              Kubernetes v1.35                   │    │
│  │                                                 │    │
│  │  Control Plane:                                 │    │
│  │    ├── kube-apiserver                           │    │
│  │    ├── kube-controller-manager                  │    │
│  │    ├── kube-scheduler                           │    │
│  │    └── etcd                                     │    │
│  │                                                 │    │
│  │  Networking:                                    │    │
│  │    ├── Cilium (CNI + kube-proxy replacement)    │    │
│  │    └── MetalLB (L2 LoadBalancer)                │    │
│  │                                                 │    │
│  │  Ingress:                                       │    │
│  │    ├── Gateway API CRDs                         │    │
│  │    └── Nginx Gateway Fabric (Gateway impl.)     │    │
│  │                                                 │    │
│  │  Storage:                                       │    │
│  │    └── local-path-provisioner                   │    │
│  │                                                 │    │
│  │  Misc:                                          │    │
│  │    └── cert-manager (TLS)                       │    │
│  │                                                 │    │
│  │  Workloads:                                     │    │
│  │    └── WIP                                      │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  MetalLB pool: 192.168.86.200–210                       │
│  Pod CIDR:     10.244.0.0/16                            │
│  Service CIDR: 10.96.0.0/12 (default)                   │
└─────────────────────────────────────────────────────────┘
```
