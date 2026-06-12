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

## Bypass lockdown (P1)

The proxy is meaningless if users can reach `pveproxy:8006` directly:

- Cluster-grade choice: **firewall** — PVE datacenter-level firewall (or upstream network ACLs): allow 8006 from {proxy IP, admin nets, cluster nodes}; drop from user networks. Apply on **every** node.
- `LISTEN_IP` in `/etc/default/pveproxy` is the single-node alternative (bind pveproxy to a management IP). On a cluster it interferes with node-to-node API proxying unless every node keeps an internal address — the firewall is cleaner.
- SPICE (3128, via `spiceproxy`) and SSH policy for users are separate decisions; quota enforcement itself only requires the 8006 lockdown.
- Verification: from a user network, `curl -k https://<every-node>:8006/` must be refused or time out.

## OIDC (Keycloak on syscom) through the proxy

The PVE GUI builds the OIDC `redirect-url` from the URL the **browser** sees — i.e. the proxy's public URL. Therefore:

1. The Keycloak client's **Valid Redirect URIs must list the proxy's public URL including the port** (forum-verified failure mode: missing port ⇒ `Error 500 Redirect Failed`).
2. The proxy forwards `X-Forwarded-Proto` / `X-Forwarded-Host` (avoid scheme downgrades behind TLS termination).
3. `/access/openid/*` endpoints are not resource writes → the proxy passes them through verbatim; no special logic.
4. The code→token exchange happens **from the cloud nodes directly to Keycloak**: every cloud node must resolve and reach the Keycloak URL and **trust the syscom CA** (drop the root into `/usr/local/share/ca-certificates/` and run `update-ca-certificates`).
5. OIDC realm settings: `autocreate=1`, `username-claim=preferred_username` (stable and readable; cannot be changed later without recreating users).

LDAP realm (if some users keep LDAP): LDAPS to syscom with the same CA-trust requirement; sync hazards in [pool-rbac.md](pool-rbac.md). Remember the single-realm policy: one human, one realm.

## Availability coupling

- PVE tickets last 2 h and renew while the GUI is open: an IdP outage blocks **new logins only** — never active sessions, and never the proxy's accounting (local-realm service account).
- syscom admins must keep `root@pam` access — syscom logins must not depend on an IdP hosted on syscom itself (bootstrap deadlock).

## Parameter sheet (fill in during P0)

| Parameter | Value |
|---|---|
| Proxy public URL | TBD |
| cloud entry node | TBD |
| Keycloak public URL | TBD |
| User networks (blocked at 8006) | TBD |
| Admin networks (exempt) | TBD |
| Allowed storages for user disks | TBD |
| Allowed bridges/VNets | TBD |
