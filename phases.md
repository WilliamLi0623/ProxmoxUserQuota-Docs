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
- Validated on the PVE 9.2.3 test cluster (uq-proxy 0.3.0-p3, `/usage` admin endpoint): for a pool with two stopped VMs the computed `cores`/`memory`/`instances` and per-storage disk matched the manual inventory exactly, including a detached **unused** disk (4 + 8 + 2 = 14 GiB). Finding: the service-account role needs `VM.Config.Disk` (not just `Datastore.Audit`) to read image-volume sizes via the storage content API on PVE 9.x — see [pool-rbac.md](pool-rbac.md).
- Exit: computed usage matches manual inventory exactly on a populated test cluster, including stopped guests and unused disks.

## P4 — Enforcement (Core Writes)

- Admission for: guest create (`POST /nodes/+/qemu|lxc`), config update (`PUT/POST .../config` — delta math against current config), `PUT .../resize`, pool membership (`PUT /pools*` — deny for users).
- Per-user serialization lock around check+forward (closes the TOCTOU race the upstream RFC could not).
- Rejection style: PVE-compatible error envelope so the native GUI shows a readable reason (per-format: json vs extjs).
- Validated on the PVE 9.2.3 test cluster (uq-proxy 0.4.0-p4, `-enforce`): over-quota create and resize were rejected (HTTP 403 / extjs `success:0` with a readable reason), an exactly-at-limit create passed, and **5-way concurrent create and 2-way concurrent resize floods admitted exactly the amount that fit and denied the rest** (cores capped at 4/4; disk capped at 23/30) — no overshoot. Finding: the per-user lock alone is insufficient because PVE applies create/resize via async tasks (the API returns a UPID before pool membership / the config `size=` update lands). The fix is to hold the per-user lock until the change is observable in live accounting — a create until its VMID joins the pool, a config/resize until the guest's config actually changes — then release.
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

- Validated on the PVE 9.2.3 test cluster (uq-proxy 0.5.0-p5, `-enforce`): over-quota **clone**, **move-disk** (to a near-full storage), **snapshot rollback** (to a larger config) and **restore** (8-core backup into a 4-core quota) were each rejected with a readable reason; the same admission path admits within-quota equivalents. Restore reads the backup's embedded config via `vzdump/extractconfig`, which forced two more service-account privileges (`Datastore.AllocateSpace` + `VM.Backup`, see [pool-rbac.md](pool-rbac.md)). Raw storage `content` allocation is checked best-effort only — unattached volumes are invisible to config-based accounting, so its enforcement is left to P6 reconciliation + storage-layer hard quotas.
- Exit: a scripted over-quota attempt through *each* side door is blocked; legitimate within-quota equivalents pass.

## P6 — Hardening & Operations

- **Default-deny for unknown write endpoints** (whitelist; PVE upgrades add endpoints — e.g. 9.1's bulk actions — and a blacklist would silently leak).
- Fail-closed drills: kill the accounting backend / upstream and verify writes are rejected, reads keep flowing.
- Storage-layer hard quotas (ZFS/Ceph) as defense in depth; read-only reconciliation cron that alarms on over-quota state reached via races or unknown paths.
- Upgrade procedure: on every PVE upgrade, re-diff the API schema against the endpoint table before allowing writes.
- HA (active/passive — the serialization point must stay singular), metrics (admission decisions, latency, upstream health), structured audit log retention.
- Validated on the PVE 9.2.3 test cluster (uq-proxy 0.6.0-p6) — the proxy-side hardening: `-default-deny` rejects writes absent from the intercept/pass-through tables (e.g. `migrate`) while known pass-through (status, consoles, `vzdump`…) still forwards; `-fail-closed` denies quota-relevant writes when the accounting backend is unreachable (validated by corrupting the service-account token — reads kept flowing, the enforced create was rejected); `/metrics` exposes Prometheus admission counters; `/usage` flags `over_quota` users for a reconciliation cron.
- **Operations landed on the live single-node deployment (PVE 9.2.3, uq-proxy 0.6.0-p6):**
  - `-fail-closed -default-deny` enabled in production and re-validated with a temporary managed user: a within-quota create still passes, `status/start` still forwards (PVE handles it), and `migrate` (an unclassified write) is rejected by the proxy with the default-deny reason.
  - **Storage-layer ZFS hard quota** as defense in depth: a per-user child dataset (`<zpool>/uq-<user>`) with `zfs set quota`, exposed as a per-user PVE `zfspool` storage and ACL'd to that user only (`Cluster/scripts/40-provision-storage.sh`). Verified independently of the proxy: even `root`+`pvesm alloc` is refused a 2 GiB thick zvol into a 1 GiB-quota dataset (`out of space`) while a 512 MiB alloc succeeds.
  - **Bypass lockdown:** `LISTEN_IP=127.0.0.1` in `/etc/default/pveproxy` binds `pveproxy` to loopback (single-node approach, see topology.md); verified from an external client that direct `:8006` is refused (`curl` exit 7) while the proxy on `:8007` keeps serving 200. SSH stays the out-of-band fallback; reversible by removing the line + `systemctl restart pveproxy`.
  - **Single-node HA:** the proxy is the singular serialization point, so it must always recover. `StartLimitIntervalSec=0` (never stop restarting) + a `uq-proxy-health.timer` liveness watchdog that restarts the service when it is *active-but-wedged* (validated with `SIGSTOP`: the frozen process was detected unhealthy and replaced; a healthy run is a no-op).
- **Still deferred (needs more than one node / a scheduled drill):** cross-node active/passive (VIP + standby; only meaningful at ≥2 nodes — design in topology.md), a live `-fail-closed` chaos drill on prod, and exercising the PVE-upgrade API-schema re-diff runbook.
- Exit: chaos drills pass; upgrade runbook exercised once on the test cluster.
