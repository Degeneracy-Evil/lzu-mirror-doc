# lzu-mirrors-ops

## Platform

- OpenStack VM
- 本次项目不计划重装该节点 OS

## Storage

- 约 98 TB storage volume
- Filesystem: XFS
- 当前约 72 TB used / 26 TB free

## Current Role Direction

该节点当前考虑作为：

- secondary storage
- migration buffer
- recovery source

它是否承担长期生产 Serving、第二 LMT Agent 节点或其他职责，尚未冻结。

由于底层 OpenStack storage 的物理拓扑、冗余和 fault domain 尚未完全掌握，目前不把该节点假定为与主机等价的可靠性副本。
