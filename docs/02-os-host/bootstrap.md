# OS Bootstrap

**Status: Not started**

本文件后续记录 Ubuntu 26.04 clean install 完成后的可重复 bootstrap 流程。

预计覆盖：

- boot / partitioning
- single-disk root filesystem
- base packages
- users / groups
- SSH
- chrony
- Netplan + systemd-networkd
- package/update policy
- systemd baseline
- hardware health tooling
- initial validation

在 Storage、Network 与 Security 的相关设计冻结前，不提前写最终命令。
