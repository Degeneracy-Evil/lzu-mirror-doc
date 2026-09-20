# OS Bootstrap

**Status: Not started**

本文件后续记录 Ubuntu 26.04 clean install 完成后的可重复 bootstrap 流程。

预计覆盖：

- boot / partitioning
- mdraid root
- base packages
- users / groups
- SSH
- chrony
- Netplan
- package/update policy
- systemd baseline
- AppArmor
- hardware health tooling
- initial validation

在 Storage、Network 与 Security 的相关设计冻结前，不提前写最终命令。


## 软件包管理

> **Provisional**：最终 APT source 列表、版本 pin 与 update 策略，需等 Storage / Network / Security 设计冻结后定稿。以下记录当前方向。

### 来源策略

基本原则：**默认只用 Ubuntu 官方源；只有当官方版本明显落后且确有功能或安全需求时，才引入软件官方源。**

- 默认启用 Ubuntu archive + security，不启用 PPA 和来源不明的第三方仓库。
- 引入第三方源时逐个记录理由（需要的版本、官方源不满足的原因、维护状态），不做“顺手加一个源”。
- 第三方源的签名 key 统一放入 `/etc/apt/keyrings`，不使用已废弃的 `apt-key`。
- 对第三方源设置 pin / priority，避免其无意覆盖 Ubuntu 基础包。
- 不使用 `curl | bash` 或随手下载的 `.deb` 绕过上述策略。
- 主机自身的 APT upstream **不指向本机正在提供的 LZU Mirror**，避免修复镜像站时形成循环依赖（见 [architecture.md](architecture.md) §8）。
- 发行版自带且已满足需求的组件（如 systemd、chrony、mdadm、AppArmor 工具）保持使用 Ubuntu 版本，不引入上游替代品。

### 版本与安装位置

- 优先使用发行版打包版本；确需新版本时才考虑软件官方源或本地编译。
- 系统级、需要长期由 systemd 管理的软件安装到标准系统路径。
- 仅为运维人员个人使用的工具，优先采用用户级安装（位于各自 `/home` 下），不进入系统包管理，避免污染主机基线与扩大补丁面。
- 非 APT 方式手工部署的软件，必须记录来源、版本、安装位置与升级方式。
- 保持 `apt-mark showmanual` 所反映的“手工安装集”尽量小，可作为 clean reinstall 时重建主机的输入。

### 安装前评审

任何安装或升级前：

- 阅读上游 release notes / changelog；
- 查阅 Ubuntu / Debian bug tracker 中已知 regression 与 bug report；
- 确认相关 CVE 与安全公告状态；
- 先用 `apt-get -s`（simulate）确认依赖与将要改动的包；
- 评估磁盘占用与回滚方式，尤其是落在小容量系统盘上的软件。

### 变更流程

- 每一次 `apt install` / `remove` / `upgrade` 都先登记工单，至少包含：目的、包与版本、来源、依赖影响、回滚方式。
- 生产主机上的包变更属于受控运维事件，不临时随手执行。
- 平时保持安全检查习惯，定期审计已安装包与第三方源。

### 更新与补丁

与 [architecture.md](architecture.md) §8 保持一致：

- security updates：自动安装
- normal package upgrades：维护窗口执行
- kernel updates：允许安装
- automatic reboot：禁止，reboot 是明确的人工运维事件

`unattended-upgrades` 的具体范围、是否 hold 关键包、以及补丁后重启的触发条件，留待 Security / Operations 设计阶段确定。
