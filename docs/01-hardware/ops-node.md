# lzu-mirrors-ops

## Platform

- OpenStack VM
- 本次项目不计划重装该节点 OS

## Storage

- 约 98 TB storage volume
- Filesystem: XFS
- 当前约 72 TB used / 26 TB free

## Current Role Direction

该节点是镜像站的**正式主存储节点之一**，不是仅用于迁移或恢复的辅助节点。

除了承担长期生产存储外，它仍可在本次重构过程中兼任：

- migration buffer
- recovery source

它是否运行第二个 LMT Agent、是否直接承担公网 Serving，以及与 main 节点之间的流量路径，留到 Network / Serving 设计阶段确定。

由于底层 OpenStack storage 的物理拓扑、冗余和 fault domain 尚未完全掌握，目前不把该节点视为其他主存储池的“可靠性副本”；它是独立的生产存储域。
