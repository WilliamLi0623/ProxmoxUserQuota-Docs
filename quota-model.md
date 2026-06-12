# Quota Model

## Subject

The quota subject is a PVE user id (`user@realm`).

- **Single-realm policy:** every human maps to exactly one realm on cloud. Otherwise `alice@ldap` and `alice@oidc` would be two unrelated quota subjects.
- For OIDC realms, `username-claim` should be a stable, human-readable claim (`preferred_username`), not `subject` or `email`. It cannot be changed after users have been created.
- Group/tenant-level aggregation is out of scope for the MVP (the model leaves room: a quota record could later attach to a group instead of a user).

## Accounting Basis

**Allocation-based, not usage-based:** a user's consumption is the sum of *configured* values over all guests in their pool — a stopped VM with 16 GiB configured consumes 16 GiB of quota. (Same choice as the upstream RFC's "config" limits and every surveyed external project; runtime-usage limiting is a non-goal.)

Usage is computed live by summing the pool members' configs (the ProxmoxAAS approach) — no separate accounting database that can drift. The pool is the accounting boundary; RBAC guarantees everything a user creates lands in their pool (see [pool-rbac.md](pool-rbac.md)).

## Dimensions

### MVP (P3/P4)

| Dimension | Unit | Counted as |
|---|---|---|
| `cores` | vCPU | per guest: `sockets × cores` (QEMU), `cores` (LXC); summed |
| `memory` | MiB | per guest: configured `memory` (ballooning counts the max); summed |
| `disk.<storage>` | GiB | per named storage: sum of all volume sizes owned by the user's guests, **including `unused[n]` volumes** |
| `instances` | count | number of guests in the pool |

### Later (P5+)

| Dimension | Notes |
|---|---|
| `net-rate` | aggregate of per-NIC `rate` values; requires every NIC to carry a `rate` (enforced at admission) |
| `cpu-types` | whitelist of allowed CPU models (+ per-model instance caps) |
| `pci` | whitelist of passthrough devices, usually per-node |
| `backups` | max backup count per guest or per user |

### Open Semantic Questions (Record Decisions Here as They Are Made)

| Question | Upstream RFC's position | Our default |
|---|---|---|
| Does CT `swap` count as memory? | disputed in review ("too cgroupv1") | count `memory` only; add a separate `swap` dimension if ever needed |
| EFI disk / TPM state volumes? | unaddressed | count them (they occupy storage) |
| Snapshot space? | excluded (the RFC skipped disk entirely) | not counted in MVP; storage-layer hard quotas backstop it (P6) |
| Linked clones? | unaddressed | count full virtual size (conservative) |

## Scope & Defaults

- **Global limit per dimension**, with optional **per-node overrides** (the ProxmoxAAS model).
- **Default deny:** a user with no quota record has zero quota. Provisioning a user means writing their quota record.
- A quota record may also pin: allowed nodes, allowed storages, allowed bridges/VNets, and an optional VMID range (readable ownership + a hard instance cap).

## Config Schema (Draft v0 — P3 Finalizes)

The proxy reads a declarative config (a file in the MVP; pluggable backend later):

```yaml
# quotas.yaml — draft v0
version: 0
defaults: {}            # intentionally empty: default deny

users:
  alice@ldap:
    pool: uq-alice      # must match the provisioned pool
    cores: 16
    memory-mib: 32768
    instances: 8
    disk-gib:
      tank: 200         # per-storage caps; storages not listed = not allowed
    nodes:              # optional per-node overrides
      node1:
        cores: 8
  bob@ldap:
    pool: uq-bob
    cores: 4
    memory-mib: 8192
    instances: 2
    disk-gib:
      tank: 50
```

Admission rule (P4): for each dimension touched by a request, `used + delta ≤ limit`, evaluated under the user's serialization lock; a dimension absent from the user's record counts as 0 (deny).
