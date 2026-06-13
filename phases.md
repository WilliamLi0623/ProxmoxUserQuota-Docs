# Implementation Phases

Standing rule: the four invariants in [architecture.md](architecture.md) hold at all times. Every phase has an explicit exit gate — do not start the next phase before passing it.

| Phase | Goal | Exit gate |
|---|---|---|
| P0 | Preparation & policy | test users create VMs into their own pools via the native GUI; mutual invisibility |
| P1 | Fully transparent pass-through proxy + bypass lockdown | every GUI flow works through the proxy; direct 8006 refused |
| P2 | Identity parsing + request classification (audit mode) | audit log attributes every action to the right user/action/delta |
| P3 | Usage accounting + quota store | computed usage matches actual allocations exactly |
| P4 | Enforcement on core write endpoints | over-quota create/config/resize rejected with a clear GUI error; concurrency-safe |
| P5 | Side doors | clone/restore/move-disk/rollback/storage-alloc all enforced |
| P6 | Hardening & operations | fail-closed on backend loss; default-deny for unknown endpoints; HA; metrics; reconciliation |

## P0 — Preparation & Policy

- Decide quota dimensions and defaults ([quota-model.md](quota-model.md)), pool naming and roles ([pool-rbac.md](pool-rbac.md)), topology and parameter sheet ([topology.md](topology.md)).
- Stand up a disposable test cluster (same major PVE version as prod; ≥2 nodes recommended).
- Provision two test users with the Cluster repo scripts; verify with `20-verify-p0.sh` + manual GUI pass.
- Full checklist: [p0-checklist.md](p0-checklist.md).

## P1 — Transparent Pass-Through + Lockdown

- Go reverse proxy (`httputil.ReverseProxy`): TLS in/out, websocket upgrade pass-through, streaming bodies (no buffering — ISO uploads), preserve all headers/cookies.
- Known traps to verify explicitly: the GUI uses `/api2/extjs/*` (different error envelope than `/api2/json`); noVNC/xterm/SPICE websocket stickiness; multipart upload streaming; long-polling task logs.
- Block direct 8006 from user networks (firewall; verify from a user VLAN).
- Validated on a PVE 9.2.3 test cluster (uq-proxy on an alternate port, upstream the local pveproxy, CA-verified): direct-vs-proxy response parity across `/api2/json` + `/api2/extjs`, cookie login + auth round-trip, console websocket `101 Switching Protocols`, and byte-exact multipart ISO upload streaming. A publicly-trusted TLS cert (Let's Encrypt via acme.sh DNS-01) was deployed so clients validate without exceptions.
- Exit: a full day of normal GUI usage through the proxy with zero behavioral differences; consoles and uploads included; direct 8006 refused. Still pending: the full-day soak (spot-check long-polling task logs and OIDC redirects) and the bypass lockdown.

## P2 — Identity + Classification (Audit Mode)

- Parse `PVEAuthCookie` (`PVE:user@realm:hex::sig`) and `Authorization: PVEAPIToken=...`; optional ticket verification via callback to `/access/ticket` (no IdP involvement, see architecture.md).
- Classify every request against the write-endpoint table; parse bodies for resource deltas (`/api2/json` and `/api2/extjs`, form-encoded and JSON).
- **No blocking** — log only: `(user, endpoint, action, parsed delta)`. Run for days; collect real traffic as test fixtures for P4.
- Validated on the PVE 9.2.3 test cluster (uq-proxy 0.2.0-p2): cookie and API-token identities correctly attributed; guest create/config and pool writes classified with resource params parsed (non-resource fields like `description` dropped, token secret never logged); non-quota writes such as `status/start` excluded; bodies read for parsing are restored byte-for-byte. Remaining operational step: a multi-day soak to collect P4 fixtures.
- Exit: sampled audit log is 100% correctly attributed and parsed.

## P3 — Accounting + Quota Store

- Service account `uq-proxy@pve` + token (role `UQ-ProxyAudit` on `/`).
- Live usage computation per user: sum configs over pool members (cores, memory, per-storage disk incl. `unused[n]`, instance count).
- Quota store: `quotas.yaml` (draft in quota-model.md) with validation + hot reload.
- Exit: computed usage matches manual inventory exactly on a populated test cluster, including stopped guests and unused disks.

## P4 — Enforcement (Core Writes)

- Admission for: guest create (`POST /nodes/+/qemu|lxc`), config update (`PUT/POST .../config` — delta math against current config), `PUT .../resize`, pool membership (`PUT /pools*` — deny for users).
- Per-user serialization lock around check+forward (closes the TOCTOU race the upstream RFC could not).
- Rejection style: PVE-compatible error envelope so the native GUI shows a readable reason (per-format: json vs extjs).
- Exit: over-quota attempts fail with a clear message in the GUI; an exactly-at-limit request passes; one-unit-over fails; concurrent same-user floods never overshoot.

## P5 — Side Doors

| Side door | Hazard | Treatment |
|---|---|---|
| clone (`POST .../clone`) | copies a possibly-large config | compute delta from source config |
| restore (`POST /nodes/+/qemu|lxc` with archive) | config hidden inside the backup | pre-check via `vzdump/extractconfig` before forwarding |
| move-disk / move-volume | can transfer volumes cross-guest | validate the target guest's owner quota |
| snapshot rollback | restores an older, larger config | pre-read the snapshot's config and admit the delta |
| storage content alloc (`POST .../content`) | raw volume allocation | count against per-storage quota |
| hotplug | config changes on a running guest | covered by the config-update interceptor |

- Exit: a scripted over-quota attempt through *each* side door is blocked; legitimate within-quota equivalents pass.

## P6 — Hardening & Operations

- **Default-deny for unknown write endpoints** (whitelist; PVE upgrades add endpoints — e.g. 9.1's bulk actions — and a blacklist would silently leak).
- Fail-closed drills: kill the accounting backend / upstream and verify writes are rejected, reads keep flowing.
- Storage-layer hard quotas (ZFS/Ceph) as defense in depth; read-only reconciliation cron that alarms on over-quota state reached via races or unknown paths.
- Upgrade procedure: on every PVE upgrade, re-diff the API schema against the endpoint table before allowing writes.
- HA (active/passive — the serialization point must stay singular), metrics (admission decisions, latency, upstream health), structured audit log retention.
- Exit: chaos drills pass; upgrade runbook exercised once on the test cluster.
