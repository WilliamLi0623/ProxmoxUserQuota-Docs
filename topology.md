# Deployment Topology & IdP Integration

## Placement

```
users ───HTTPS:443──▶ [quota proxy VM] ───HTTPS:8006──▶ cloud entry node ──▶ other cloud nodes
admins ──────────────HTTPS:8006 (direct, exempt)──────▶ cloud nodes
cloud nodes ─────────LDAPS / HTTPS────────────────────▶ syscom (LDAP / Keycloak)
user browsers ───────HTTPS (OIDC redirects)───────────▶ Keycloak
```

Proxy host — options, to be decided in P0:

| Option | Pros | Cons |
|---|---|---|
| VM on syscom (recommended) | survives cloud outages; infra lives with infra | cross-cluster network path on every request |
| VM on cloud | no cross-cluster dependency in daily traffic | chicken-and-egg when cloud is unhealthy |
| Bare host / container elsewhere | independent failure domain | one more pet to manage |

MVP: **single proxy instance, single designated entry node** on cloud. PVE's built-in cross-node proxying handles the rest. Multi-upstream/HA: P6.

## Bypass Lockdown (P1)

The proxy is meaningless if users can reach `pveproxy:8006` directly:

- Cluster-grade choice: **firewall** — PVE datacenter-level firewall (or upstream network ACLs): allow 8006 from {proxy IP, admin nets, cluster nodes}; drop from user networks. Apply on **every** node.
- `LISTEN_IP` in `/etc/default/pveproxy` is the single-node alternative (bind pveproxy to a management IP). On a cluster it interferes with node-to-node API proxying unless every node keeps an internal address — the firewall is cleaner.
- SPICE (3128, via `spiceproxy`) and SSH policy for users are separate decisions; quota enforcement itself only requires the 8006 lockdown.
- Verification: from a user network, `curl -k https://<every-node>:8006/` must be refused or time out.
- **Live single-node deployment:** the proxy and `pveproxy` share one host and the proxy already dials `https://127.0.0.1:8006`, so `LISTEN_IP="127.0.0.1"` binds `pveproxy` to loopback only — direct `:8006` becomes unreachable off-host while the proxy path is untouched. Verified: `pveproxy` moved from `*:8006` to `127.0.0.1:8006`; an external client gets connection-refused on `:8006` and 200 on the proxy's `:8007`. SSH (22) is the out-of-band fallback; an admin who needs the raw GUI during a proxy outage tunnels it (`ssh -L 8006:127.0.0.1:8006 root@host`). Rollback is one line + `systemctl restart pveproxy`.

## OIDC (Keycloak on syscom) Through the Proxy

The PVE GUI builds the OIDC `redirect-url` from the URL the **browser** sees — i.e. the proxy's public URL. Therefore:

1. The Keycloak client's **Valid Redirect URIs must list the proxy's public URL including the port** (forum-verified failure mode: missing port ⇒ `Error 500 Redirect Failed`).
2. The proxy forwards `X-Forwarded-Proto` / `X-Forwarded-Host` (avoid scheme downgrades behind TLS termination).
3. `/access/openid/*` endpoints are not resource writes → the proxy passes them through verbatim; no special logic.
4. The code→token exchange happens **from the cloud nodes directly to Keycloak**: every cloud node must resolve and reach the Keycloak URL and **trust the syscom CA** (drop the root into `/usr/local/share/ca-certificates/` and run `update-ca-certificates`).
5. OIDC realm settings: `autocreate=1`, `username-claim=preferred_username` (stable and readable; cannot be changed later without recreating users).

LDAP realm (if some users keep LDAP): LDAPS to syscom with the same CA-trust requirement; sync hazards in [pool-rbac.md](pool-rbac.md). Remember the single-realm policy: one human, one realm.

## Availability Coupling

- PVE tickets last 2 h and renew while the GUI is open: an IdP outage blocks **new logins only** — never active sessions, and never the proxy's accounting (local-realm service account).
- syscom admins must keep `root@pam` access — syscom logins must not depend on an IdP hosted on syscom itself (bootstrap deadlock).

## High Availability (P6)

The proxy is the **singular serialization point** — the per-user admission lock lives in one process, so two active enforcers would each admit up to the limit and overshoot. HA therefore means *fast single-active recovery*, never active/active.

**Single node (current deployment).** The only failure domain is the one process:

- `Restart=always` + `StartLimitIntervalSec=0` — systemd restarts it forever, even after rapid crash loops (it never hits a start-rate limit and gives up).
- A liveness watchdog (`deploy/uq-proxy-health.timer` → `uq-proxy-healthcheck.sh`, every 30 s) covers the case systemd's process-liveness misses: a process that is still *active* but **wedged** (deadlock, stuck listener). It restarts the unit only when `is-active=active` **and** `/healthz` is not OK, so it never fights systemd's own restart of a dead process. Validated with `SIGSTOP` (frozen proxy detected and replaced; a healthy run is a no-op).
- Out-of-band access during an outage is SSH (the 8006 lockdown is loopback-bound, not firewalled), so an admin can always tunnel the raw GUI.

**Multi-node (deferred until ≥2 nodes/hosts).** Active/passive only:

- One **VIP** (keepalived/VRRP) in front of a single *active* proxy; the passive node runs `uq-proxy` stopped (or behind the VIP health check) and takes the VIP only on failure. The lock state is in-memory and per-request, so a cold passive is correct — no shared lock store needed, as long as only one instance ever holds the VIP.
- Both nodes mount the same `quotas.yaml` (or sync it) and each has its own `uq-proxy@pve` token; accounting is stateless (recomputed from PVE), so failover needs no state transfer.
- Keep the single designated upstream entry node (PVE cross-node proxying handles the rest); the VIP fronts the proxy, not pveproxy.

## Parameter Sheet (Fill In During P0)

| Parameter | Value |
|---|---|
| Proxy public URL | TBD |
| cloud entry node | TBD |
| Keycloak public URL | TBD |
| User networks (blocked at 8006) | TBD |
| Admin networks (exempt) | TBD |
| Allowed storages for user disks | TBD |
| Allowed bridges/VNets | TBD |
