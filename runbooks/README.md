# Runbooks

本目录用于保存生产环境中可直接执行的运维流程。

与 `docs/` 的区别：

- `docs/` 说明系统为什么这样设计、目标状态是什么；
- `runbooks/` 说明实际发生某个事件时应该按什么步骤操作。

当前草稿（**未验证**，待冻结后定稿）：

- [mdraid 建组与磁盘更换](mdraid-build-and-replace.md)

预计后续包含：

- service deployment / restart
- mirror onboarding / removal
- disk failure / replacement
- RAID rebuild or data restore
- LMT recovery / fence handling
- host reboot
- maintenance mode
- backup restore
- full disaster recovery

在相关架构和实施方案冻结前，不提前编写未经验证的操作步骤。上述草稿仅记录方向，不代表已验证的生产步骤。
