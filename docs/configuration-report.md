# Configuration Report — Homelab DevOps

Detailed log of the infrastructure build process, including technical decisions, issues encountered, and the fixes applied. This document is meant as reference material for technical interview preparation.

## 1. Base infrastructure

**Hardware**: mini-PC with an Intel i5-12450H (12 threads), 32GB RAM, Proxmox VE as the hypervisor.

**VMs created**:

| VM | vCPU | RAM | Disk | IP | Role |
|---|---|---|---|---|---|
| `k3s-node` | 4 | 8GB | 40GB | 192.168.1.20 | Kubernetes cluster |
| `jenkins-controller` | 2 | 4GB | 30GB | 192.168.1.21 | CI/CD (next stage) |

Both run Ubuntu Server 26.04, fresh install, with a `tiberiu` user, SSH access via public key (imported from GitHub), password login disabled.

**Decision**: no separate VM was created for Ansible — it runs directly from the workstation (Mac), communicating over SSH with the targets defined in the inventory. This simplifies the workflow (editing with a familiar editor, no extra SSH hop just to write code).

## 2. Provisioning with Ansible

### Structure

The project uses **roles** from the start, rather than monolithic playbooks — a reusable structure closer to real-world practice.

```
ansible/
├── ansible.cfg
├── inventory.ini
├── playbooks/
└── roles/
```

### Issue #1 — interactive sudo

The first playbook (`baseline.yml`) failed on tasks using `become: true`, with the error:

```
sudo: A terminal is required to authenticate
```

**Cause**: the `tiberiu` user required a password for `sudo`, but Ansible runs commands non-interactively over SSH and cannot supply password input.

**Fix**: configured `NOPASSWD` for the user via a dedicated file (not by editing `/etc/sudoers` directly, to avoid the risk of a syntax error locking out sudo access entirely):

```bash
sudo visudo -f /etc/sudoers.d/tiberiu
```
```
tiberiu ALL=(ALL) NOPASSWD: ALL
```

**Production note**: `NOPASSWD: ALL` is appropriate for a homelab only. In real production environments, the correct practice is to limit passwordless commands to a specific set, following the least-privilege principle.

### Issue #2 — role not found

```
[ERROR]: The role 'baseline' was not found in: .../playbooks/roles:...
```

**Cause**: the project structure keeps `roles/` at the repository root, alongside `playbooks/`, rather than nested inside it (`playbooks/roles/`, which is Ansible's default search path).

**Fix**: explicit configuration in `ansible.cfg`:
```ini
[defaults]
roles_path = roles
```

### Issue #3 — UFW module syntax error

```
parameters are mutually exclusive: name|proto|logging
```

**Cause**: the Ansible `ufw` module does not allow combining the `name` parameter with `port`/`proto` in the same task.

**Fix**: clear separation — using either `name` (for predefined profiles like `OpenSSH`) or explicit `port` + `proto`, never both.

### Issue #4 (critical) — firewall self-lockout

**What happened**: after fixing the syntax error, the task enabling UFW (`policy: deny`, the default for inbound traffic) ran **before** the allow rule for port 22 (SSH) had actually taken effect — the result was SSH access being blocked on both VMs.

**Diagnosis**: confirmed via direct console access through the Proxmox web interface (`sudo ufw status` → `active`, with no working SSH access).

**Immediate fix**: `sudo ufw disable`, executed directly from the Proxmox console (SSH was unavailable at that point).

**Permanent fix**: reordered the tasks in the playbook so the SSH allow rule is applied **before** the firewall is enabled:

```yaml
- name: Allow SSH through UFW
  ufw:
    rule: allow
    port: "22"
    proto: tcp
  become: true

- name: Enable and start UFW
  ufw:
    state: enabled
    policy: deny
  become: true
```

**Lesson**: task ordering matters not just logically but operationally — a wrong sequence can produce a complete access lockout on infrastructure. Having an alternative access path (hypervisor console, IPMI, etc.) is essential as a safety net.

## 3. k3s installation

Installed via Ansible using the official install script (`get.k3s.io`), with an idempotent task (`creates: /usr/local/bin/k3s`, so repeated playbook runs don't unnecessarily reinstall it).

The kubeconfig was fetched locally via the `fetch` module, then manually corrected — by default it contains `server: https://127.0.0.1:6443`, which doesn't work outside the VM and needs to be replaced with the node's real IP.

**Verification**: `kubectl get nodes` → `Ready` status, k3s version confirmed.

## 4. Vaultwarden deployment

The first service deployed to the cluster, split into 4 separate manifests: `namespace`, `pvc`, `deployment`, `service`.

**Design decisions**:
- `replicas: 1` — Vaultwarden uses SQLite by default (single-writer), which doesn't support horizontal scaling without migrating to PostgreSQL/MySQL.
- A `PersistentVolumeClaim` is mandatory — without it, data (including passwords) is lost on any pod restart.
- Initially exposed via `NodePort` (simple, no extra dependencies), later migrated to `ClusterIP` + Ingress.

**Minor issue**: on the first `kubectl apply -f .`, `deployment.yaml` was processed before `namespace.yaml`, resulting in a `namespaces "vaultwarden" not found` error. Fixed by re-running the command (the namespace already existed on the second pass). To avoid this systematically, numeric file prefixes or a tool like Kustomize/Helm can be used to resolve dependency order automatically.

## 5. TLS — from local certificate to public certificate

### Stage 1 — mkcert (local development)

First working setup: local hostname (`vault.homelab.local`) added to `/etc/hosts`, certificate generated with `mkcert`, mounted as a Kubernetes Secret, Ingress via Traefik (bundled with k3s).

**Acknowledged limitation**: mkcert is suitable for local development only — the certificate is trusted only on machines where the local CA has been installed, which isn't a valid solution for a multi-device or public-facing setup.

### Stage 2 — Let's Encrypt with DNS-01 challenge (Cloudflare)

**Specific requirement**: services need to remain accessible only from the local network, but with a valid public TLS certificate (not self-signed). A Cloudflare Tunnel approach was explicitly ruled out — prior experience showed that publicly exposing an unfamiliar login page can trigger a phishing flag from Google Safe Browsing.

**Chosen solution**:
- Cloudflare used **only as a DNS provider** ("DNS only" record, no proxying)
- Public DNS record → private IP (192.168.1.20), unreachable from outside the local network
- Certificate obtained via **DNS-01 challenge**, which verifies domain ownership through a temporary TXT record, without requiring the server to be publicly reachable (unlike HTTP-01)

**Components installed**:
- `cert-manager` (CRDs + controller)
- `ClusterIssuer` configured with a `dns01.cloudflare` solver, authenticated via an API token with minimal permissions (`Zone.DNS: Edit`, scoped to the specific zone)

**Result**: a real, Let's Encrypt-signed certificate for a service that remains fully isolated from public access.

## 6. Automating cert-manager with Ansible

The manual configuration above was later fully automated:

- The `kubernetes.core` collection was installed to enable Ansible ↔ Kubernetes interaction
- The Cloudflare API token is managed via **Ansible Vault** (`ansible-vault create vars/vault.yml`), never in plaintext in Git
- `no_log: true` on the task creating the Kubernetes Secret, to prevent the token from appearing in verbose output

### Issues encountered

**Undefined variable**: `'cloudflare_api_token' is undefined` — caused by a mismatch between the filename referenced in `vars_files` (`vars.yml`) and the actual file created with Ansible Vault (`vault.yml`). Fixed by correcting the reference in the playbook.

**Invalid module parameter**: `Unsupported parameters for (kubernetes.core.k8s) module: no_log` — caused by placing `no_log` *inside* the module block, when it's actually a task-level keyword that must be aligned with `name:`, not with the module's parameters. Fixed by correcting the indentation.

## 7. Portainer deployment

The second service, structured similarly to Vaultwarden, with one additional component: **RBAC**.

**RBAC**: a dedicated `ServiceAccount` + `ClusterRoleBinding` to the `cluster-admin` role, required so Portainer can view and manage resources across the whole cluster, not just its own namespace. Explicitly acknowledged as a permissive choice, suitable only for a single-user environment; in a team setting a restricted `ClusterRole` would be used instead.

### Issue — HTTP 400 error when accessing via Ingress

**Cause**: Portainer exposes port `9443` by default with its own internal TLS (self-signed). Traefik, already configured to terminate TLS at the cluster edge (using the cert-manager certificate), was sending requests to a port expecting a separate TLS handshake — a "double TLS termination" conflict.

**Fix**: exposed and used Portainer's internal HTTP port (`9000`) as the target for the Service and Ingress, leaving TLS termination to happen in a single place (Traefik, at the edge).

**Lesson**: applications that perform their own internal TLS termination frequently conflict with an Ingress Controller doing the same at the edge — the general solution is identifying the application's "plain" HTTP port.

### Setup token

On first boot, Portainer generates an initial setup token, visible only in the pod logs:

```bash
kubectl get pods -n portainer
kubectl logs -n portainer -l app=portainer --tail=50
```

The token expires after about 5 minutes; if it does, deleting the pod (`kubectl delete pod ...`) triggers an automatic recreation (via the Deployment) with a fresh token.

## 8. Key concepts to mention in the interview

- **Idempotency** in Ansible (`creates:`, module design) — repeated runs don't produce side effects
- **Operation order matters**: the firewall lockout as a concrete example of why task sequencing isn't just stylistic
- **HTTP-01 vs DNS-01** for certificate validation — DNS-01 doesn't require public exposure of the service
- **Least privilege**: applied conceptually to sudoers and explicitly acknowledged as a limitation in Portainer's RBAC setup (`cluster-admin` as an accepted trade-off only in a homelab context)
- **Secrets management**: Ansible Vault + `no_log`, as an alternative to hardcoding, with explicit acknowledgment that production would use a dedicated system (HashiCorp Vault, AWS Secrets Manager)
- **Data persistence**: a PVC as a hard requirement for any stateful service, with SQLite's single-writer limitation as a concrete scaling example

## 9. Next steps

- [ ] Jenkins — CI/CD pipeline replacing manual `kubectl apply`
- [ ] Prometheus + Grafana — cluster and container monitoring
- [ ] Automated backups for persistent volumes (Kubernetes CronJob)
- [ ] Full recovery test (destroy VM → recreate via Ansible + Jenkins)
