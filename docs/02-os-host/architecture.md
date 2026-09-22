# OS / Host Architecture

> 本文记录第一版目标。具体分区表、sysctl、服务 hardening 参数等仍将在正式安装前继续细化。

## 1. Host Philosophy

主机应尽量保持简单、长期可维护：

```text
Ubuntu 26.04
    │
    ├── systemd
    ├── Netplan → systemd-networkd
    ├── chrony
    ├── native Linux storage
    │
    ├── LMT
    ├── Nginx
    └── Observability
```

没有明确收益时，不为基础服务引入 Kubernetes、复杂容器编排或额外 supervisor。长期服务统一由 systemd 管理。

## 2. System Disk Direction

系统盘不做 RAID。

当前方向：

```text
single dedicated system disk
          │
         ext4
          │
           /
```

具体使用哪一块现有或新增磁盘，在正式迁移前再决定。

系统盘故障通过配置备份、关键状态备份和可重复 bootstrap 解决，而不是通过本机 RAID1 增加系统盘层级复杂度。

暂不引入 LVM；如后续出现明确需求再重新评估。

## 3. State Boundary

系统盘保存小体量、关键的持久状态，例如：

- OS 与 systemd units
- LMT Server database
- LMT Agent spool
- service configuration
- credentials / tokens
- Nginx configuration
- observability configuration

这些关键状态需要有独立的备份 / 恢复方案，不能因为系统盘是单盘就只保留唯一副本。

大体量或可重建数据不放在 root filesystem：

- mirror contents
- LMT publication generations
- large cache
- staging data
- large observability datasets

## 4. Service Data Namespace

大型生产数据统一使用 `/srv`。最终 mount point 应描述服务用途，而不是历史硬件名称。

具体多个主存储池如何映射到目录树尚未冻结；当前不预设 `/srv/bulk` 这类“次级存储”角色。

不继续使用类似 `/mnt/tank2`、`/mnt/9traid` 这种绑定历史硬件的名字。

## 5. Service Management

长期服务包括 LMT Server、LMT Agent、Nginx 与 observability services，统一由 systemd 管理。

不允许生产服务长期依赖：

- screen / tmux
- nohup
- 手工后台 shell
- cron @reboot

周期性系统任务优先使用 systemd timers。

## 6. Network Baseline

Netplan 与 NetworkManager 不是同一层的替代关系。

本项目选择：

```text
/etc/netplan/*.yaml
        │
      Netplan
        │
        ▼
systemd-networkd
```

也就是说：

- **Netplan** 是持久网络配置入口；
- **systemd-networkd** 是实际管理服务器网卡的 renderer；
- **NetworkManager** 不作为本机生产网络管理器。

NIC、bond、VLAN、IPv4/IPv6、MTU、routing 等细节留给 Network 设计。

## 7. Time

- RTC / kernel time: UTC
- NTP: chrony
- host timezone: Asia/Shanghai
- structured logs / metrics / protocol timestamps: 尽量使用 UTC 或 Unix timestamp

## 8. Updates

第一版策略：

- security updates: 自动安装
- normal package upgrades: maintenance window
- kernel updates: 可以安装
- automatic reboot: 禁止
- reboot: 明确的人工运维事件

主机自己的 APT upstream 不依赖本机正在提供的 LZU Mirror，避免修复镜像站时形成循环依赖。

## 9. Ubuntu 26.04 Compatibility Notes

升级后需要专门验证：

- systemd / cgroup v2 行为
- rust-coreutils 对现有运维脚本的兼容性
- LMT installer / maintenance scripts
- `/tmp` 默认 tmpfs 对大文件任务的影响

规则：

> 大型同步、解压、repository build、staging 不使用 `/tmp`，而使用明确的大容量工作目录。

## 10. Users and Permissions

第一版模型：

- root: 禁止直接 SSH 登录
- human admins: sudo
- lmt-server: dedicated service user
- lmt-agent: dedicated service user
- nginx: dedicated service user
- observability services: 各自 service user

目标职责：

- Nginx 只读正式 mirror tree
- LMT Agent 只写自己管理的数据树
- LMT Server 不直接写 mirror tree
- monitoring 尽可能只读

具体 UID/GID、ACL 与 sudo policy 留给 Security 设计。

## 11. Security Baseline

当前方向：

- Secure Boot：**关闭**
- root SSH 禁止
- password SSH 倾向禁用
- 最小化公网开放服务
- 服务使用独立用户
- 暂不对系统盘使用 LUKS

Secure Boot 当前不属于本项目的主要 threat model，关闭它可以减少启动链、驱动和后续维护上的额外约束。若未来安全模型发生变化，再重新评估。

Firewall 的最终实现和完整 host hardening 留给 Network / Security 设计。

## 12. Logging Baseline

systemd service 默认写入 journald。

第一版目标：

```text
persistent journald
+ bounded retention
        ↓
central logging / Loki
```

Nginx access log 因数据量特殊，后续单独设计。

## 13. Hardware Health Baseline

OS 层至少应提供：

- SMART
- RAID state
- NVMe health
- filesystem capacity
- thermal state
- NIC state

具体 metrics、dashboard 和 alert policy 留给 Observability。
