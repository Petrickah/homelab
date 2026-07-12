# Configuration Report — Homelab DevOps

Detailed log of the infrastructure build process, including technical decisions, issues encountered, and the fixes applied. This document is meant as reference material for technical interview preparation.

## 1. Base infrastructure

**Hardware**: mini-PC with an Intel i5-12450H (12 threads), 32GB RAM, Proxmox VE as the hypervisor.

**VMs created**:

| VM                   | vCPU | RAM | Disk | IP           | Role               |
|----------------------|------|-----|------|--------------|--------------------|
| `k3s-node`           | 4    | 8GB | 40GB | 192.168.1.20 | Kubernetes cluster |
| `jenkins-controller` | 2    | 4GB | 30GB | 192.168.1.21 | CI/CD (next stage) |

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

## 8. Jenkins — CI/CD

### Installation via Ansible

Jenkins was installed on the dedicated `jenkins-controller` VM (192.168.1.21), automated through a new Ansible role: Java (OpenJDK 21, required by current Jenkins LTS), the Jenkins apt repository and GPG key, the `jenkins` package itself, a UFW rule opening port 8080, and enabling the systemd service.

### Issue — 404 on apt repository (flat repository structure)

```
Failed to fetch https://pkg.jenkins.io/debian-stable/dists/stable/InRelease  404  Not Found
```

**Cause**: the Jenkins apt repository is a **flat repository** — packages sit directly at the repository root, with no `dists/<suite>/<component>/` hierarchy like a standard Debian repo. The initial `deb822_repository` task was configured with `suites: stable` and `components: [main, contrib, non-free]`, which made apt look for a standard suite structure that doesn't exist for this repo — hence the 404 on `dists/stable/InRelease`.

**Fix**: matched the official Jenkins install command (`deb [...] https://pkg.jenkins.io/debian-stable binary/`), which uses the `binary/` suffix — the marker for a flat repository. In `deb822_repository` terms, that means setting `suites: binary/` and removing `components` entirely, since a flat repo has no components:

```yaml
- name: Add Jenkins Repository
  deb822_repository:
    name: jenkins
    types: deb
    uris: https://pkg.jenkins.io/debian-stable
    suites: binary/
    signed_by: https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
  become: true
```

**Lesson**: not all third-party apt repositories follow the standard Debian layout — tools like Jenkins, Docker, and HashiCorp often ship flat repositories for simplicity. The `binary/` suffix (in place of a real suite name) is the distinguishing signal.

### Initial setup

Standard Jenkins setup wizard: unlock using the initial admin password (`/var/lib/jenkins/secrets/initialAdminPassword`, read via SSH), "Install suggested plugins", admin account creation.

### kubectl on the controller

Since the pipeline runs directly on the Jenkins controller (`agent any`, no separate build agents — a deliberate scope decision, see below), `kubectl` needed to be installed on `jenkins-controller` itself. Added as an extra task in the `jenkins` Ansible role, downloading the binary directly from `dl.k8s.io`.

### Credentials management

The `kubeconfig-k3s.yaml` file was uploaded to Jenkins as a **Secret file** credential (ID: `kubeconfig-k3s`), through **Manage Jenkins → Credentials**. The pipeline references it only by ID via the `withKubeConfig` step (from the Kubernetes CLI plugin) — the actual file contents are never exposed in the Jenkinsfile or in build logs. This mirrors the same secrets-management principle already applied with Ansible Vault, now at the CI/CD layer: no secret ever hardcoded, anywhere in the pipeline definition.

### Pipeline

Job `vaultwarden-deploy`, configured as **Pipeline script from SCM**, pointing at the `Jenkinsfile` at the repository root. Declarative pipeline with four stages:

```groovy
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps { checkout scm }
        }
        stage('Validate manifests') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig-k3s']) {
                    sh 'kubectl apply --dry-run=client -f k8s/vaultwarden/'
                }
            }
        }
        stage('Deploy Vaultwarden') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig-k3s']) {
                    sh 'kubectl apply -f k8s/vaultwarden/'
                }
            }
        }
        stage('Verify rollout') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig-k3s']) {
                    sh 'kubectl rollout status deployment/vaultwarden -n vaultwarden --timeout=60s'
                }
            }
        }
    }
    post {
        success { echo 'Deployment finished successfully.' }
        failure { echo 'Deployment failed — check the logs above.' }
    }
}
```

**Design notes**:
- `--dry-run=client` runs first as a fail-fast validation step, catching manifest syntax errors before anything is actually applied to the cluster.
- `withKubeConfig` injects the credential as a temporary `KUBECONFIG` environment variable, scoped only to that steps block.
- `Verify rollout` explicitly confirms the deployment reached its desired state, rather than assuming success just because `kubectl apply` returned without error.

**First run**: succeeded end to end on the first attempt. The log showed `unchanged` for every resource, since the manifests were already applied manually beforehand — a concrete demonstration of idempotency at the `kubectl apply` level, the same principle already seen with Ansible's `creates:`.

### Automatic trigger — Poll SCM

A GitHub webhook was considered but deliberately ruled out: a webhook requires GitHub to reach Jenkins over the public internet, which would mean exposing the Jenkins controller publicly — the same exposure risk already avoided earlier with Vaultwarden (Cloudflare Tunnel triggering a phishing flag). Consistent with that decision, **Poll SCM** was used instead: Jenkins periodically checks the repository for changes ("pull" model) rather than requiring an inbound public endpoint.

Configured with schedule:
```
TZ=Europe/Bucharest
H/5 * * * *
```

The explicit `TZ` setting avoids ambiguity in build logs, since Jenkins would otherwise use the server's default timezone rather than the local one.

**Verified**: a comment-only commit to `k8s/vaultwarden/service.yaml` triggered an automatic build within the polling interval, with no manual "Build Now" click — confirming a fully closed CI/CD loop from `git push` to a verified rollout on the cluster.

### Scope decision — no separate build agents

The pipeline runs directly on the Jenkins controller (`agent any`), rather than on a separate build agent/worker. In a production setup, the standard pattern separates the **controller** (orchestration only) from one or more **agents** (actual step execution) — either static (a dedicated VM with tools preinstalled) or dynamic (an ephemeral container spun up per build, e.g. a `kubectl`-equipped image, torn down afterward). For a single-VM, single-cluster homelab, this added complexity wasn't worth the practical benefit; the trade-off was made consciously and is worth stating explicitly in an interview, the same way the `cluster-admin` RBAC choice for Portainer was.

## 9. Key concepts to mention in the interview

- **Idempotency** in Ansible (`creates:`, module design) — repeated runs don't produce side effects
- **Operation order matters**: the firewall lockout as a concrete example of why task sequencing isn't just stylistic
- **HTTP-01 vs DNS-01** for certificate validation — DNS-01 doesn't require public exposure of the service
- **Least privilege**: applied conceptually to sudoers and explicitly acknowledged as a limitation in Portainer's RBAC setup (`cluster-admin` as an accepted trade-off only in a homelab context)
- **Secrets management**: Ansible Vault + `no_log`, as an alternative to hardcoding, with explicit acknowledgment that production would use a dedicated system (HashiCorp Vault, AWS Secrets Manager)
- **Data persistence**: a PVC as a hard requirement for any stateful service, with SQLite's single-writer limitation as a concrete scaling example
- **Fail-fast validation**: `--dry-run=client` as a cheap pre-check before applying changes to a live cluster
- **Pull vs push automation**: Poll SCM chosen over a GitHub webhook specifically to avoid exposing the CI server publicly, consistent with the DNS-01/no-tunnel decision made for Vaultwarden
- **Controller vs agent separation**: a conscious, stated trade-off — running steps directly on the Jenkins controller for a small, single-cluster homelab, while understanding why production setups separate execution onto dedicated or ephemeral agents

## 10. Next steps

- [x] Jenkins — CI/CD pipeline replacing manual `kubectl apply`, with automatic trigger via Poll SCM
- [ ] Extend the pipeline to cover the Portainer deployment as well
- [ ] Prometheus + Grafana — cluster and container monitoring
- [ ] Automated backups for persistent volumes (Kubernetes CronJob)
- [ ] Full recovery test (destroy VM → recreate via Ansible + Jenkins)
- [ ] Terraform — provisioning of the VMs themselves (currently out of scope; Ansible only configures VMs that already exist). Noted as a deliberate scope boundary rather than an oversight.