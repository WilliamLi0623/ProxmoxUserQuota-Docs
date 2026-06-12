# P0 Checklist — Preparation & Policy

Exit gate: **two test users can each create a VM into their own pool via the native GUI, and cannot see each other's resources.**

## 0. Decisions (Defaults Pre-Filled; Override by Editing the Docs)

- [x] Approach: transparent intercepting proxy ("Mode A") — decided
- [x] Quota dimensions (MVP): cores / memory / per-storage disk / instances — [quota-model.md](quota-model.md)
- [x] Accounting: allocation-based, live summation over pool members, default deny — [quota-model.md](quota-model.md)
- [x] Pool naming: `uq-<username>`, no silent sanitization — [pool-rbac.md](pool-rbac.md)
- [x] Single-realm policy: one human = one realm — [quota-model.md](quota-model.md)
- [x] Service account in local realm (`uq-proxy@pve`), never IdP — [pool-rbac.md](pool-rbac.md)
- [ ] Proxy placement: syscom VM (recommended) / cloud VM / other → record in [topology.md](topology.md)
- [ ] Allowed storages for user disks: ______
- [ ] Allowed bridges/VNets: ______
- [ ] Default quota per user class (e.g. member / power user): ______
- [ ] Fill the parameter sheet in [topology.md](topology.md)

## 1. Test Cluster

- [ ] Disposable PVE cluster matching prod major version (nested VMs are fine; **≥2 nodes recommended** so P1 can exercise cross-node GUI proxying)
- [ ] One storage for test disks, one bridge (e.g. `vmbr0`)

## 2. Provision Test Users (on a Cluster Node, as root)

    git clone https://github.com/WilliamLi0623/ProxmoxUserQuota-Cluster.git
    cd ProxmoxUserQuota-Cluster/scripts
    ./00-create-roles.sh
    pveum user add testuser1@pve --password '<pw1>'
    pveum user add testuser2@pve --password '<pw2>'
    ./10-provision-user.sh testuser1@pve -s <storage> -b vmbr0
    ./10-provision-user.sh testuser2@pve -s <storage> -b vmbr0

## 3. Verify — API Level

    ./20-verify-p0.sh https://<node>:8006 testuser1@pve '<pw1>' testuser2@pve '<pw2>' <node>

- [ ] All checks PASS (pool visibility, mutual invisibility, no-pool create denied, foreign-pool create denied, membership edit denied)
- [ ] Optionally re-run with `--create <vmid>` to prove the positive path via API

## 4. Verify — Native GUI (the Actual Exit Gate)

- [ ] testuser1 logs into `https://<node>:8006` (no proxy yet — P0 validates RBAC only)
- [ ] Create VM wizard: user **must select pool `uq-testuser1`** — record whether the GUI preselects it (UX note)
- [ ] VM creates successfully and appears under the user's pool
- [ ] testuser2 sees neither testuser1's pool nor the VM (search + pool view)
- [ ] testuser1 cannot migrate the VM, cannot edit permissions, cannot open the node shell
- [ ] Both users: disk creation only succeeds on the allowed storage; NIC only on the allowed bridge
- [ ] Record evidence (screenshots + PVE version)

## 5. Known P0 Caveats

- The GUI create wizard fails if the user forgets to pick their pool (they have no global `VM.Allocate`). Acceptable — document it for users.
- If any GUI step fails with a missing-privilege error, adjust the role in [pool-rbac.md](pool-rbac.md) + `00-create-roles.sh`, re-run, and note the PVE version.
- IdP (LDAP/Keycloak) wiring is **not** part of P0 — test users live in `@pve`. IdP integration arrives with the proxy (P1, see [topology.md](topology.md)).
