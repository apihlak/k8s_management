# Self-hosting everything with Kustomize and ArgoCD

**TODO:**

- Integrating Argo Workflows with the stack

**Components:**

- Cilium - CNI Network and Security
- Sealed-secrets - Encrypting secrets
- MetalLB - Load Balancer for bare metal
- Traefik - Ingress Controller / Load Balancer
- Cert-manager - Certificate manager (Letsencrypt)
- Rook (with Ceph) - Persistent storage
- Linkerd - Service Mesh
- Flagger - Automated application analysis, testing, promotion and rollback
- GitLab - Code repository
- Harbor - Registry + scanner for artifacts
- ArgoCD - Continuous Delivery

**Domains:**

- https://traefik.myorg.com
- https://rook.myorg.com
- https://linkerd.myorg.com
- https://gitlab.myorg.com
- https://registry.myorg.com
- https://harbor.myorg.com
- https://argo-cd.myorg.com
- https://argo-workflows.myorg.com
