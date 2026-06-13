# API Endpoint Classification

Basis: the PVE 9.1 API schema (675 endpoints). These tables are the data source for P2 (classification), P4 (core enforcement), P5 (side doors) and P6 (default-deny whitelist). Paths are relative to `/api2/{json|extjs}` — the interceptor must treat both envelopes identically.

## Quota-Relevant Write Endpoints (Intercept)

| # | Method & Path | Effect | Dimensions Touched | Phase |
|---|---|---|---|---|
| 1 | `POST /nodes/{node}/qemu` | create VM; with `archive` = **restore** (config hidden inside the backup) | cores, memory, disk.*, instances | P4; restore P5 |
| 2 | `POST /nodes/{node}/lxc` | create CT; `ostemplate`/`restore` variants | cores, memory, disk.*, instances | P4; restore P5 |
| 3 | `PUT` / `POST /nodes/{node}/qemu/{vmid}/config` | change `sockets`/`cores`/`memory`; add disks (`scsiN`/`virtioN`/`sataN`/`ideN`/`efidisk0`/`tpmstate0`, incl. `import-from`); NIC `rate` | cores, memory, disk.*, net-rate | P4 |
| 4 | `PUT /nodes/{node}/lxc/{vmid}/config` | same for CT (`cores`, `memory`, `swap`, `rootfs`, `mpN`) | cores, memory, disk.* | P4 |
| 5 | `PUT /nodes/{node}/qemu/{vmid}/resize` | grow a disk | disk.&lt;storage&gt; | P4 |
| 6 | `PUT /nodes/{node}/lxc/{vmid}/resize` | grow rootfs/mountpoint | disk.&lt;storage&gt; | P4 |
| 7 | `POST /nodes/{node}/qemu/{vmid}/clone` | full/linked clone | all (delta = source config) | P5 |
| 8 | `POST /nodes/{node}/lxc/{vmid}/clone` | clone CT | all | P5 |
| 9 | `POST /nodes/{node}/qemu/{vmid}/move_disk` | move disk across storages; **`target-vmid` reassigns across guests** | disk.* of both owners | P5 |
| 10 | `POST /nodes/{node}/lxc/{vmid}/move_volume` | CT equivalent (also has `target-vmid`) | disk.* | P5 |
| 11 | `POST /nodes/{node}/qemu/{vmid}/snapshot/{snap}/rollback` | restores an older — possibly larger — config | all (delta = snapshot config) | P5 |
| 12 | `POST /nodes/{node}/lxc/{vmid}/snapshot/{snap}/rollback` | same | all | P5 |
| 13 | `POST /nodes/{node}/storage/{storage}/content` | raw volume allocation | disk.&lt;storage&gt; | P5 |
| 14 | `PUT /pools` (and legacy `PUT /pools/{poolid}`) | pool membership = the accounting boundary | integrity — **deny for users** | P4 |
| 15 | `POST /nodes/{node}/storage/{storage}/upload`, `POST .../download-url` | ISO/template occupies storage | disk.&lt;storage&gt; | P5+ (only if template upload is ever granted) |

Implementation notes:

- **Restore pre-check (#1/#2):** read the embedded config via `GET /nodes/{node}/vzdump/extractconfig?volume=...` (as the service account) before forwarding.
- **Rollback pre-check (#11/#12):** read the snapshot's config via `GET .../snapshot/{snap}/config` and admit the delta vs the current config.
- **Config endpoints (#3/#4) carry mixed params:** description, tags, boot order etc. ride the same endpoint. The interceptor evaluates only the resource delta and must let pure non-resource edits pass untouched.
- **Deletes free quota** (`DELETE .../qemu/{vmid}`, `DELETE .../lxc/{vmid}`, disk `unlink`, volume delete): always pass — usage can only shrink.

## Write Endpoints That Must Pass Through Untouched

| Path Family | Why |
|---|---|
| `POST .../status/(start\|stop\|shutdown\|reboot\|suspend\|resume\|reset)` | power ops change no allocation |
| `POST .../vncproxy`, `.../termproxy`, `.../spiceproxy` | console ticket creation — blocking these breaks the GUI |
| `GET .../vncwebsocket` (websocket upgrade) | the console data channel itself |
| `POST /access/ticket`, `POST /access/openid/*`, `PUT /access/password` | login flows (incl. Keycloak OIDC) and self-service password |
| `POST .../snapshot` (create), `DELETE .../snapshot/{snap}` | snapshot space is not counted in the MVP (storage-layer hard quotas backstop it) |
| `POST /nodes/{node}/vzdump` | backups; backup-count quota arrives P5+, pass in MVP |
| `POST .../agent/*`, firewall, tags, notes | guest-agent and non-resource guest config |

## Default-Deny Rule (P6)

Write endpoints not present in either table are **rejected** until classified — PVE upgrades add new writes (e.g. the bulk-action endpoints added in 9.x), and a blacklist would silently leak. GET/HEAD/OPTIONS always pass. On every PVE upgrade, re-diff the API schema against these tables before re-enabling writes (see phases.md, P6).
