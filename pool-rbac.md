# Pool Ownership & RBAC

## One Pool per User

| Convention | Value |
|---|---|
| Pool name | `uq-<username>` (the part before `@`) |
| Created by | `ProxmoxUserQuota-Cluster/scripts/10-provision-user.sh` |
| Accounting boundary | the pool — usage = sum over pool members |

Pool ids only allow `[A-Za-z0-9._-]`. The username → pool mapping must stay **injective**:

- the provisioning script refuses usernames containing characters outside that set — no silent sanitization (two different usernames must never collapse onto one pool);
- the single-realm policy ([quota-model.md](quota-model.md)) lets the realm be omitted from pool names; if multiple user realms ever coexist, revisit this.

## The Ownership Binding

The entire ownership model hangs on one rule:

> `VM.Allocate` is granted **only** on `/pool/uq-<user>` — never on `/`, never on `/vms`.

Consequences, enforced by PVE's own RBAC (no proxy involved):

- Creating a guest requires `VM.Allocate` on the target path; the only place the user has it is their pool → **every guest they create must enter their pool** (in the GUI create wizard they must select the pool).
- Permissions on guests flow through pool membership → users see and manage only their own guests ("mutual invisibility").
- Users never get `Pool.Allocate`, so they cannot edit pool membership (`PUT /pools`). This closes a real quota leak: removing a running VM from your own pool would remove it from accounting while it keeps serving you. The proxy additionally guards pool-membership endpoints from P4 on (defense in depth).

## Roles

Created idempotently by `00-create-roles.sh`:

| Role | Granted on | Privileges |
|---|---|---|
| `UQ-VMUser` | `/pool/uq-<user>` | `Pool.Audit VM.Allocate VM.Audit VM.Backup VM.Clone VM.Config.CDROM VM.Config.CPU VM.Config.Cloudinit VM.Config.Disk VM.Config.HWType VM.Config.Memory VM.Config.Network VM.Config.Options VM.Console VM.PowerMgmt VM.Snapshot VM.Snapshot.Rollback` |
| `UQ-Storage` | `/storage/<each allowed storage>` | `Datastore.AllocateSpace Datastore.Audit` |
| `UQ-Net` | `/sdn/zones/localnetwork/<bridge>` (or specific VNets) | `SDN.Use` |
| `UQ-ProxyAudit` | `/` (proxy service account, from P3) | `VM.Audit VM.Config.Disk VM.Backup Pool.Audit Datastore.Audit Datastore.AllocateSpace Sys.Audit SDN.Audit` |

Deliberately **excluded** from `UQ-VMUser`, and why:

| Privilege | Why excluded |
|---|---|
| `Pool.Allocate` | membership lock (see above) |
| `VM.Migrate` | placement is an admin/scheduler concern |
| `VM.Monitor` | raw QEMU monitor allows host/device poking |
| `Datastore.Allocate` / `Datastore.AllocateTemplate` | storage admin; ISO/template upload can be granted per user later if wanted |
| `Permissions.Modify`, `User.*`, `Realm.*`, `Sys.*`, `Mapping.*` | self-service must not touch ACLs, users, or the host |

Notes:

- `VM.Snapshot.Rollback` stays (users expect it) even though a rollback can restore an older, larger config. That side door is closed by the proxy in P5, not by RBAC.
- The minimal set must be validated empirically in P0 — PVE occasionally gates GUI actions on extra privileges. If the GUI create wizard fails with a missing-privilege error, adjust the role here and in `00-create-roles.sh`, and record the PVE version.

## Service Account

- `uq-proxy@pve` — **local PVE realm, never LDAP/OIDC** (must survive IdP outages; no circular dependency).
- Used with an API token; role `UQ-ProxyAudit` on `/` (read-only). The proxy forwards user requests under the *user's own* credentials — the service account exists only for accounting reads (pool membership, guest configs) and health checks.
- Several "write-ish" privileges are included so the **read-only** accounting calls actually succeed on PVE 9.2.3 — PVE gates these reads behind them:
  - `VM.Config.Disk` — the storage content API (`GET .../storage/{storage}/content`), used to size **unused** disks with no inline `size=`, filters VM-owned image volumes by `VM.Config.Disk`; `Datastore.Audit` alone returns an empty list.
  - `Datastore.AllocateSpace` + `VM.Backup` — `GET .../vzdump/extractconfig` (the P5 restore pre-check that reads the config embedded in a backup) requires `Datastore.AllocateSpace` on the backup storage **and** `VM.Backup` on the source VMID.
  The token is used only for `GET`s by the proxy and is stored root-only, so it stays effectively read-only, but the role is wider than pure audit. If these privileges are missing, the affected accounting read fails and admission currently **fails open** (allows) — P6 makes it fail closed. Revisit if PVE changes these checks.
- Created in P2/P3; not needed for P0.

## LDAP Sync Hazard

`pveum realm sync` with `remove-vanished` semantics **overwrites `user.cfg` from the directory** — and pool ACLs live in `user.cfg`. An aggressive sync can silently delete the ACLs that bind users to their pools: that either opens a quota hole or locks everyone out.

Mitigations:

- Prefer **OIDC autocreate** (users created on first login) over periodic full LDAP sync.
- If LDAP sync is used: never enable `remove-vanished=acl`; treat `10-provision-user.sh` as an idempotent **reconciler** and re-run it (cron or sync hook) after every sync.
- P6 adds a read-only auditor that alarms when a pool member exists without a matching quota record or ACL.
