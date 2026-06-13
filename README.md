# ProxmoxUserQuota

**中文** | [English](README_en.md)

为 Proxmox VE 实现**按用户的资源配额**。PVE 至今没有原生配额功能（[Bugzilla #1141](https://bugzilla.proxmox.com/show_bug.cgi?id=1141) 自 2016 年悬置；官方 2024 年的 pool resource limits RFC 也未合并）。本项目采用**透明拦截反向代理**路线：

- 用户继续使用**原生 PVE Web 界面**（含 noVNC/SPICE 控制台、ISO 上传），不另做面板；
- 代理只拦截会改变资源分配的少数写端点（约 15 个），其余请求逐字转发；
- 配额按「分配值」记账（配置之和），默认拒绝，fail-closed。

## 仓库结构

| 仓库 | 职责 |
|---|---|
| [ProxmoxUserQuota-Docs](https://github.com/WilliamLi0623/ProxmoxUserQuota-Docs) | 设计文档、决策记录、路线图 |
| [ProxmoxUserQuota-Cluster](https://github.com/WilliamLi0623/ProxmoxUserQuota-Cluster) | PVE 集群侧供给与验证脚本 |
| [ProxmoxUserQuota-Proxy](https://github.com/WilliamLi0623/ProxmoxUserQuota-Proxy) | 透明配额代理本体（Go） |

## 架构不变量

1. 用户只经代理访问 PVE；代理对原生 GUI 完全透明；
2. 除资源变更写请求外，一切逐字转发，不改写不缓冲；
3. 用户网络直连 `pveproxy:8006` 必须在网络层封死；
4. **Fail-closed**：无法完成配额判定的写请求一律拒绝。

## 文档索引

设计文档正文为英文。

| 文件 | 内容 |
|---|---|
| [architecture.md](architecture.md) | 总体架构、不变量、双集群（cloud/syscom）拓扑 |
| [quota-model.md](quota-model.md) | 配额主体、维度、作用域、配置 schema |
| [pool-rbac.md](pool-rbac.md) | 每用户一池约定、最小角色、服务账号、LDAP 同步风险 |
| [endpoints.md](endpoints.md) | API 端点分类：拦截清单与直通清单 |
| [topology.md](topology.md) | 部署选址、旁路封锁、OIDC/LDAP 接入 |
| [phases.md](phases.md) | P0–P6 实施阶段与退出门槛 |
| [p0-checklist.md](p0-checklist.md) | P0 可执行清单 |

## 状态

**P0 —— 准备与策略**（进行中）。见 [p0-checklist.md](p0-checklist.md)。

## 许可证

[MIT](LICENSE)
