# OS / Host Architecture

> 本文记录第一版目标。具体分区表、sysctl、服务 hardening 参数等仍将在正式安装前继续细化。

## 1. Host Philosophy

主机应尽量保持简单、长期可维护：

```text
Ubuntu 26.04
    │
    ├── systemd
    ├── Netplan + systemd-networkd
    ├── chrony
    ├── AppArmor
    ├── native Linux storage
    │
    ├── LMT
    ├── Nginx
    └── Observability
```

没有明确收益时，不为基础服务引入 Kubernetes、复杂容器编排或额外 supervisor。长期服务统一由 systemd 管理。

## 2. System Disk Direction

当前方向：

```text
2 × Intel 480 GB SATA SSD
          │
      mdadm RAID1
          │
         ext4
          │
           /
```

EFI / boot 的精确布局尚未冻结，但最终应验证任意一块系统 SSD 缺失时仍可独立启动。

暂不引入 LVM；如后续出现明确需求再重新评估。

## 3. State Boundary

系统 RAID1 保存小体量、关键的持久状态，例如：

- OS 与 systemd units
- LMT Server database
- LMT Agent spool
- service configuration
- credentials / tokens
- Nginx configuration
- observability configuration

大体量或可重建数据不放在 root filesystem：

- mirror contents
- LMT publication generations
- large cache
- staging data
- large observability datasets

## 4. Service Data Namespace

大型生产数据统一使用 `/srv`，mount point 描述**用途**而不是硬件：

```text
/srv/lmt-data
/srv/bulk
/srv/fast
/srv/recovery
```

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

OS 层先确定：

```text
Netplan
  ↓
systemd-networkd
```

Server 不以 NetworkManager 作为生产网络管理层。

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
- Dracut + mdraid root 启动
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

- AppArmor 保持启用
- Secure Boot 倾向启用
- root SSH 禁止
- password SSH 倾向禁用
- 最小化公网开放服务
- 服务使用独立用户
- 暂不对系统盘使用 LUKS

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
- mdraid state
- NVMe health
- filesystem capacity
- thermal state
- NIC state

具体 metrics、dashboard 和 alert policy 留给 Observability。
