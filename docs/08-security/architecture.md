# Security Architecture

**Status: First design drafted — Provisional，尚未冻结**

> 第一版安全模型方向。privilege model、ACL、firewall、审计规则与告警策略仍需在 Network / Observability 设计推进后细化。
> Firewall 的最终实现留给 `docs/07-network/`；日志管道与告警留给 `docs/09-observability/`。

## 1. 设计原则

```text
最小权限
  + 最小攻击面
  + 默认拒绝
  + 假设会被攻破（纵深防御）
  + 可审计、可溯源
```

- **最小权限**：用户、服务、进程只拿到完成其职责所必需的最小权限。
- **最小攻击面**：不对外暴露不需要的服务与端口；能用内网/本机访问的就不公网开放。
- **默认拒绝**：firewall、绑定地址、文件权限默认收紧，按需放开。
- **纵深防御**：单层防护失败不应导致全站失守。
- **不依赖隐匿**：安全来自机制，而不是“别人不知道”。

安全最终依赖运维承担持续责任：保持软件与系统处于**受支持、已修补**的版本，否则公开的老版本漏洞利用工具即可击穿防线。

## 2. 信任边界与资产

需要在 Security + Network 阶段明确的边界：

- 公网 ↔ 主服务器（仅 Nginx 服务面）
- 管理网络 ↔ 主机（SSH、监控、iDRAC/带外）
- 主服务器 ↔ `lzu-mirrors-ops`
- LMT Server ↔ LMT Agent 的信任边界

关键资产：

- mirror tree 的**完整性**（内容可重建，但被篡改影响所有下游用户）
- LMT Server database、LMT Agent spool
- credentials / tokens / SSH keys
- 服务配置、systemd units
- 管理入口（SSH、sudo、IPMI/iDRAC）

## 3. 身份与权限模型（最小权限）

第一版模型见 `docs/02-os-host/architecture.md` §10，安全侧要点：

- **root**：禁止直接 SSH 登录；只作为本地/带外应急入口。
- **human admins**：通过 sudo 操作，按人分配，不共享账号。
- **service users**：`lmt-server`、`lmt-agent`、`nginx`、observability 各服务各自独立用户，职责隔离：

```text
Nginx           → 只读正式 mirror tree
LMT Agent       → 只写自己管理的数据树
LMT Server      → 不直接写 mirror tree
monitoring      → 尽可能只读
```

- sudo / IPMI / root：不随意授予与运维无关的人员；包括 `docker` 组这类等价于 root 的权限。
- 具体 UID/GID、ACL、sudoers、su 策略在 Security + Storage 阶段冻结。

## 4. 服务隔离与 systemd hardening

长期服务统一由 systemd 管理（`docs/02-os-host/architecture.md` §5），并利用 systemd 沙箱能力收紧：

- 显式设置 `User=` / `Group=` / `DynamicUser=`，避免服务以 root 运行。
- 按需使用 `ProtectSystem=`、`ProtectHome=`、`PrivateTmp=`、`PrivateDevices=`、`NoNewPrivileges=`、`ProtectKernel*=`、`RestrictAddressFamilies=`、`CapabilityBoundingSet=`、`SystemCallFilter=`、`ReadWritePaths=`、`UMask=` 等隔离项。
- 用 `systemd-analyze security <unit>` 量化 exposure level，作为 hardening 的检查与回归手段。

容器（如有）不作为例外：不随意 `--privileged`，不无限制 `bind 0.0.0.0`，容器内服务同样要有自己的用户。当前架构不引入 Kubernetes / 容器编排（`docs/02-os-host/architecture.md` §1），因此不为其预留额外权限面。

## 5. 攻击面与对外服务

- 先盘点：主机上用 `ss -tlnp` 列出监听端口；外部用 `nmap` 从攻击者视角确认实际可达面。
- 不需要对外的服务，收紧绑定地址或关闭监听。
- 预期只有 Nginx 面向公网；管理端口（SSH 等）限管理网络或做来源限制。
- 对持续扫描/爆破，可部署 `fail2ban` 等工具按日志自动封禁；jail/filter/action 需按实际日志格式配置。
- firewall 的最终规则集由 `docs/07-network/` 与本文共同冻结，`docs/08-security/` 负责策略与最小化目标。

## 6. SSH 策略

- 禁止 root 直接登录。
- **强烈倾向禁用密码认证**，使用密钥登录；条件允许时叠加 2FA 或 SSH 证书。
- 首次连接核对 host key fingerprint；出现 `REMOTE HOST IDENTIFICATION HAS CHANGED` 必须人工确认，不盲目删除 `known_hosts`。
- 不使用公网直连的 RDP/VNC 作为管理通道；需要时通过 SSH 端口转发访问内网管理面。
- SSH 端口/来源限制与 Nginx 的关系由 Network 设计确定。

## 7. 凭据与机密

- tokens、私钥、密码等机密文件权限最小化（如 `0600`），置于受控目录，避免进入世界可读配置。
- 机密**不进入 Git 仓库**（设计文档、脚本均适用）。
- 明文机密尽量交由 systemd 的凭据机制或受控文件传递，而不是写在命令行/环境变量中被日志记录。
- 轮换与吊销流程（LMT token、SSH key、证书）在 Security + Operations 阶段补全。

## 8. 软件包与供应链

遵循 `docs/02-os-host/bootstrap.md` 的包管理策略，安全侧的约束：

- 默认只用 Ubuntu 官方源；第三方源逐个评审并 pin，签名 key 放 `/etc/apt/keyrings`。
- 不使用 `curl | bash` 或来源不明的 `.deb` / 二进制。
- 安装前阅读 bug report 与 CVE 状态；定期审计已安装包与第三方源。
- 关注**固件**供应链：厂商固件缺陷（如 SSD 通电小时数导致的批量损坏）同样属于供应链风险，见 `runbooks/mdraid-build-and-replace.md` §10.1。
- LMT、Nginx、observability 等组件的安装来源与升级方式需各自记录。

## 9. 漏洞与补丁响应

- 订阅发行版安全通告（Ubuntu security notices 的 RSS / 邮件列表），关注带 CVE 编号的公告并评估严重程度。
- 更新策略继承 `docs/02-os-host/architecture.md` §8：security updates 自动安装；normal upgrades 维护窗口；kernel 可更新但不自动重启。
- 维护一份“上游来源清单”（官方源 / 官方 release），便于在公告发布后快速判断是否受影响。

## 10. 审计与日志

- 复用 `docs/02-os-host/architecture.md` §12 的 persistent journald + bounded retention。
- 需要更严格审计时启用 `auditd`，规则可参考 `/usr/share/doc/auditd/examples/audit-rules/` 中的 `30-stig.rules` / `30-ospp-v42.rules`；**必须逐条阅读，不可直接复制**。
- `acct` 可记录执行过的命令（`acct.service`，日志在 `/var/log/account/pacct`），用于事后核对。
- 审计与系统日志统一汇入 Observability 的集中日志（`docs/09-observability/`）。
- 日志保留期限与访问控制由 Observability + Security 共同确定。

## 11. 检测、响应与溯源

- 依赖 Observability 的监控与告警发现异常（登录失败、权限提升、服务异常等）。
- 建立最小的事件响应流程：隔离受影响服务/主机 → 保留证据（日志、内存/磁盘镜像）→ 溯源 → 修复 → 复盘。
- 溯源参考：`journald` / `auditd` / `acct` 日志；进程与连接排查（`ss -tpn` 等）；不轻信单一工具输出。
- 正式的事件响应与溯源步骤属于 `runbooks/`，待架构冻结且演练后编写。

## 12. 自动化检查与加固

- `lynis audit system` 等工具可自动发现待加固项，按输出逐项评估后再修改，不盲目套用。
- `nuclei` 等漏洞扫描器**仅在获得授权的前提下**对自有资产使用；不得对未授权目标扫描。
- 扫描结果作为持续加固输入，纳入定期检查。

## 13. 依赖与待冻结项

- firewall 实现与规则集（依赖 Network）
- 集中日志、告警与审计日志保留（依赖 Observability）
- sudoers / ACL / UID-GID 具体值（依赖 Storage / OS）
- 管理通道（管理网络、iDRAC/带外）的最终形态
- 是否启用 Secure Boot、SSH 2FA/证书的具体方案
- `fail2ban` / `auditd` / `lynis` 的常规运行与告警接入方式
- 机密管理与轮换方案
