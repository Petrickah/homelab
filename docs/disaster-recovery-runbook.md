# Disaster Recovery Runbook

Documents the actual, tested recovery procedure for a guest running on the Proxmox host, plus the results of a real end-to-end drill performed on 2026-08-29. This is a procedure log, not a theoretical plan — every command below was actually run.

## What backup mechanism this covers

Every guest (VMs and LXCs) is backed up nightly at 03:00 by a Proxmox `vzdump` job (`/etc/pve/jobs.cfg`, job id `nightly-syno`) to Synology NAS storage over NFS (`syno-nfs`), in `snapshot` mode with `zstd` compression and retention `keep-daily=7,keep-weekly=4,keep-monthly=3`. The NAS additionally takes its own daily Btrfs snapshot of that share, independent of Proxmox. This runbook tests the first layer — recovering a guest from a `vzdump` archive — not the NAS-snapshot layer, and not a from-scratch Ansible rebuild (see "What this does NOT prove" below).

## Recovery procedure (generalized)

1. Identify the backup to restore from:
   ```bash
   pvesm list syno-nfs --vmid <VMID>
   ```
2. Restore it:
   - VM (QEMU): `qmrestore syno-nfs:backup/<archive> <VMID> --storage local-zfs`
   - LXC (container): `pct restore <VMID> syno-nfs:backup/<archive> --storage local-zfs`
3. Start it: `qm start <VMID>` / `pct start <VMID>`
4. Verify, in order: networking (does it get an IP, is it reachable), application (are the expected services active), data (does the expected data/content match what was there before loss).

## Drill performed 2026-08-29

Real guests were deliberately **not** used for this drill — the plan calls for proving the mechanism works, not risking a live service to do it. A disposable throwaway LXC was created for the sole purpose of being destroyed and restored.

**Setup**: created `vmid 199` (`dr-drill-test`), Debian 13, 1 core / 256MB RAM / 2GB disk, on `vmbr0` with DHCP. Installed a trivial systemd-managed app (`python3 -m http.server` serving a directory containing a uniquely-named marker file) so there was something concrete to check afterward — not just "the disk exists."

**Baseline, pre-destroy**:
```
$ curl http://192.168.1.110:8099/marker.txt
dr-drill-marker-8f3a1c-2026-08-29T11:03:25+03:00
$ ssh root@192.168.1.110 hostname
dr-drill-test
```

**Backup** (on-demand, not waiting for the nightly job):
```
$ vzdump 199 --storage syno-nfs --compress zstd --mode snapshot
...
INFO: archive file size: 160MB
INFO: Backup job finished successfully
```

**Destroy**:
```
$ pct stop 199 && pct destroy 199
$ pct status 199
Configuration file 'nodes/pm-mp200/lxc/199.conf' does not exist
```
Container fully gone — not stopped, not just powered off.

**Restore**:
```
$ pct restore 199 syno-nfs:backup/vzdump-lxc-199-2026_08_29-11_03_40.tar.zst --storage local-zfs
$ pct start 199
```
Came back on the same DHCP-assigned IP.

**Post-restore verification** — every check passed:
| Check | Result |
|---|---|
| Networking (SSH reachable) | `ssh root@192.168.1.110` → `dr-drill-test` |
| Application (systemd service survived and auto-started) | `systemctl is-active drtest.service` → `active` |
| Data integrity (marker content byte-identical) | `dr-drill-marker-8f3a1c-2026-08-29T11:03:25+03:00` — exact match against the pre-destroy baseline |

**Cleanup**: test container destroyed again afterward (`pct stop 199 && pct destroy 199`), and the test backup archive removed from the NAS (`/mnt/pve/syno-nfs/dump/vzdump-lxc-199-*`) so it doesn't linger as clutter alongside the real guests' backups.

## What this proves

The actual claim being tested — "if a guest is lost, the existing backup can bring it back with its application and data intact" — is now empirically confirmed for the LXC restore path, not just assumed because a backup job exists and hasn't errored.

## What this does NOT prove

- **Not tested on a real guest.** The drill used a minimal, stateless-ish throwaway container. A real guest (e.g. `k3s-control`, running Gitea/Vaultwarden/the registry as Docker containers with real data) would take longer to restore (its backups are multi-GB, not 160MB) and has more moving parts to verify (Docker daemon starts cleanly, all containers come back healthy, K3s rejoins the cluster). The *mechanism* is identical (`qmrestore`/`pct restore` from the same `vzdump` archives that already back up every real guest nightly), but restoring an actual multi-service VM hasn't itself been drilled.
- **Not a from-scratch rebuild test.** This validates "restore from an existing backup," not "rebuild this host from nothing via Ansible" (see the `Next steps` checklist below — that's a separate, still-open capability).
- **Not an off-site/site-loss test.** Both the Proxmox backups and the Synology snapshots live on hardware in the same location. If the site itself is lost (fire, theft, both devices failing together), none of this helps — that gap is explicitly deferred pending external backup media.
- **Restore time wasn't benchmarked** against any target (no RTO/RPO defined) — for a 160MB test container it took well under a minute; a 30GB VM restore over NFS will take meaningfully longer, not measured here.

## Next steps checklist

- [x] Full recovery test — LXC backup/restore path, drilled 2026-08-29 (this document)
- [ ] Full recovery test — restore one of the real multi-service guests (or an equivalent-complexity clone) to validate the same path at realistic scale
- [ ] Full recovery test — rebuild via Ansible from nothing (separate capability from backup/restore)
- [ ] Off-site copy of backups, pending external storage media
