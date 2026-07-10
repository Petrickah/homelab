# Homelab DevOps — Infrastructure as Code

Personal project built to prepare for a **Junior DevOps Engineer** role, running on a mini-PC with Proxmox VE. The goal is to practice a full provisioning, containerization, and orchestration workflow (with CI/CD as the next stage), using real production-grade tools at a small scale.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Proxmox VE (host)                     │
│                                                               │
│  ┌──────────────────────┐      ┌──────────────────────────┐  │
│  │   k3s-node             │      │   jenkins-controller       │  │
│  │   192.168.1.20          │      │   192.168.1.21               │  │
│  │   4 vCPU / 8GB RAM      │      │   2 vCPU / 4GB RAM           │  │
│  │                         │      │                              │  │
│  │   Ubuntu Server 26.04   │      │   Ubuntu Server 26.04        │  │
│  │   ├── k3s (Kubernetes)  │      │   (Jenkins — in progress)    │  │
│  │   ├── Traefik (Ingress) │      │                              │  │
│  │   ├── cert-manager      │      └──────────────────────────┘  │
│  │   ├── Vaultwarden       │                                    │
│  │   └── Portainer         │                                    │
│  └──────────────────────┘                                    │
└─────────────────────────────────────────────────────────────┘
```

## Tech stack

| Component | Role |
|---|---|
| Proxmox VE | Hypervisor, hosts the VMs |
| Ubuntu Server 26.04 | OS for all VMs |
| Ansible | Automated provisioning, configuration, secrets management |
| k3s | Lightweight Kubernetes distribution |
| Traefik | Ingress Controller (bundled with k3s) |
| cert-manager | Automated TLS certificate issuance (Let's Encrypt) |
| Cloudflare | DNS provider, used for the DNS-01 challenge |
| Vaultwarden | Self-hosted password manager (Bitwarden-compatible server) |
| Portainer | Kubernetes cluster management UI |
| Jenkins | CI/CD (in progress) |

## Repository structure

```
homelab/
├── ansible/
│   ├── ansible.cfg
│   ├── inventory.ini
│   ├── playbooks/
│   │   ├── baseline.yml
│   │   ├── k3s.yml
│   │   └── cert-manager.yml
│   ├── roles/
│   │   ├── baseline/
│   │   ├── k3s/
│   │   └── cert-manager/
│   └── vars/
│       └── vault.yml          # encrypted with Ansible Vault, no secrets in plaintext
├── k8s/
│   ├── vaultwarden/
│   │   ├── namespace.yaml
│   │   ├── pvc.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── ingress.yaml
│   ├── portainer/
│   │   ├── namespace.yaml
│   │   ├── rbac.yaml
│   │   ├── pvc.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── ingress.yaml
│   └── cert-manager/
│       └── issuer.yaml
└── docs/
    └── configuration-report.md   # detailed setup log, issues and fixes
```

## Reproducing the environment from scratch

1. **Proxmox VMs**: create two Ubuntu Server 26.04 VMs with static IPs, key-based SSH access (no password), and a user with `sudo NOPASSWD`.
2. **Inventory**: fill in `ansible/inventory.ini` with the actual IPs.
3. **Baseline**: `ansible-playbook playbooks/baseline.yml` — updates the system, installs essential packages, configures the firewall (ufw), and disables SSH password authentication.
4. **k3s**: `ansible-playbook playbooks/k3s.yml` — installs k3s on the dedicated node and fetches the kubeconfig locally.
5. **cert-manager**: create `ansible/vars/vault.yml` with the Cloudflare token (`ansible-vault create vars/vault.yml`), then `ansible-playbook playbooks/cert-manager.yml --ask-vault-pass`.
6. **Applications**: `kubectl apply -f k8s/vaultwarden/` and `kubectl apply -f k8s/portainer/`.

## Security — key decisions

- **No hardcoded passwords**: the Cloudflare token is encrypted with Ansible Vault; it never appears in plaintext in Git.
- **`no_log: true`** on Ansible tasks that handle secrets, to keep them out of verbose output.
- **SSH key-only authentication**, with password login explicitly disabled after provisioning.
- **Public DNS, strictly local access**: domains are publicly resolvable via Cloudflare (required for the DNS-01 challenge), but the target IPs are private — services are not reachable from outside the local network.
- **Least-privilege RBAC as a discussed principle** — Portainer runs with `cluster-admin` only because this is a single-user homelab; in a team environment a restricted `ClusterRole` would be used instead.

## Status

- [x] Automated provisioning (Ansible)
- [x] Kubernetes cluster (k3s)
- [x] Automatic TLS (cert-manager + Let's Encrypt via DNS-01)
- [x] Vaultwarden deployed, with persistence
- [x] Portainer deployed, with RBAC
- [ ] Jenkins pipeline for automated deployment (in progress)
- [ ] Monitoring (Prometheus + Grafana)
- [ ] Automated backups for persistent volumes

Step-by-step details, including issues encountered and how they were fixed, are in [`docs/configuration-report.md`](docs/configuration-report.md).
