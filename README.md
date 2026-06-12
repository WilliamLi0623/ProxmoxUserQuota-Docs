# ProxmoxUserQuota

**[中文]**

为 Proxmox VE 实现**按用户的资源配额**。PVE 至今没有原生配额功能（[Bugzilla #1141](https://bugzilla.proxmox.com/show_bug.cgi?id=1141) 自 2016 年悬置；官方 2024 年的 pool resource limits RFC 也未合并）。本项目采用**透明拦截反向代理**路线：

- 用户继续使用**原生 PVE Web 界面**（含 noVNC/SPICE 控制台、ISO 上传），不另做面板；
- 代理只拦截会改变资源分配的少数写端点（约 15 个），其余请求逐字转发；
- 配额按「分配值」记账（配置之和），默认拒绝，fail-closed。

**[English]**

Per-user **resource quotas** for Proxmox VE. PVE has no native quota support ([Bugzilla #1141](https://bugzilla.proxmox.com/show_bug.cgi?id=1141), open since 2016; the upstream 2024 "pool resource limits" RFC was never merged). This project takes the **transparent intercepting reverse proxy** approach:

- Users keep the **native PVE web GUI** (incl. noVNC/SPICE consoles and ISO uploads) — no custom panel;
- The proxy intercepts only the ~15 write endpoints that change resource allocation; everything else is forwarded verbatim;
- Quotas are allocation-based (sum of configured values), default-deny, fail-closed.

## Repositories | 仓库

| Repo | Role | 职责 |
|---|---|---|
| [ProxmoxUserQuota-Docs](https://github.com/WilliamLi0623/ProxmoxUserQuota-Docs) | Design docs, decisions, roadmap | 设计文档、决策记录、路线图 |
| [ProxmoxUserQuota-Cluster](https://github.com/WilliamLi0623/ProxmoxUserQuota-Cluster) | PVE cluster-side provisioning & verification scripts | 集群侧供给与验证脚本 |
| [ProxmoxUserQuota-Proxy](https://github.com/WilliamLi0623/ProxmoxUserQuota-Proxy) | The transparent quota-enforcing proxy (Go) | 透明配额代理本体（Go） |

## Architecture invariants | 架构不变量

1. Users reach PVE **only through the proxy**; the proxy is fully transparent to the native GUI.
   用户只经代理访问 PVE；代理对原生 GUI 完全透明。
2. Everything except resource-mutating writes is forwarded **verbatim**.
   除资源变更写请求外，一切逐字转发，不改写不缓冲。
3. Direct access to `pveproxy:8006` from user networks is **blocked at the network layer**.
   用户网络直连 8006 必须在网络层封死。
4. **Fail closed**: a write that cannot be quota-evaluated is rejected.
   无法完成配额判定的写请求一律拒绝。

## Documents | 文档索引

| File | Content |
|---|---|
| [architecture.md](architecture.md) | System overview, invariants, two-cluster (cloud/syscom) topology |
| [quota-model.md](quota-model.md) | Quota subjects, dimensions, scopes, config schema |
| [pool-rbac.md](pool-rbac.md) | Pool-per-user convention, minimal roles, service account, LDAP sync hazards |
| [topology.md](topology.md) | Deployment placement, bypass lockdown, OIDC/LDAP integration |
| [phases.md](phases.md) | P0–P6 implementation phases with verification & exit gates |
| [p0-checklist.md](p0-checklist.md) | Actionable P0 checklist |

## Status | 状态

**P0 — Preparation & policy**（进行中 / in progress）。See [p0-checklist.md](p0-checklist.md).
