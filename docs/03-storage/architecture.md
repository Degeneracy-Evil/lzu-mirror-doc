# Storage Architecture

> **Provisional**：本设计尚未冻结。正式迁移前允许调整。

## 1. Design Principles

1. 镜像内容属于可重建数据，不为几十 TiB 镜像副本投入大规模冗余容量。
2. 容量利用率优先。
3. 允许完整停站迁移，不为在线迁移引入额外复杂度。
4. 优先通过本机磁盘和 `lzu-mirrors-ops` 搬迁已有数据，避免从公网重新全量同步。
5. 不把六块 HDD 合并为一个 RAID0，当前采用两个独立三盘故障域的方向。
6. 3 × 18 TB、3 × 10 TB 与 ops 98 TB volume 都视为**正式主存储**，不再划分“主池 / bulk 次池”。

## 2. Controller

Dell PERC H730P Mini 保持 **HBA mode**：

```text
Physical Disks
     ↓
PERC H730P (HBA)
     ↓
Linux software storage
```

不计划为了本次重构切换为 PERC hardware RAID。

## 3. Current Direction

```text
lzu-mirrors-main

3 × 18 TB WDC
    ↓
independent RAID0
    ↓
production primary storage pool A

3 × 10 TB HGST
    ↓
independent RAID0
    ↓
production primary storage pool B

single dedicated system disk
    ↓
ext4
    ↓
Ubuntu / local control-plane state


lzu-mirrors-ops

~98 TB XFS volume
    ↓
production primary storage pool C
```

两个 HDD RAID0 当前文件系统方向仍倾向：

```text
mdadm RAID0
    ↓
XFS
```

系统盘不做 RAID，具体物理盘待正式迁移前决定。

## 4. Why 3 + 3 Instead of 6-Disk RAID0

两组 HDD 在容量、型号和服役时间上本来就形成自然分组。

六盘统一 RAID0 会把整个本地镜像数据绑定到任意一块盘的寿命；拆成两个三盘 RAID0 不改变“六块盘中出现故障”的总体可能性，但显著缩小单盘故障的影响范围。

尤其 10 TB 盘通电时间约 6.9 万小时，而 18 TB 盘约 4.2 万小时，因此当前不希望把两组盘绑定成一个统一故障域。

## 5. Multi-Pool Production Storage

三个存储域都计划承担长期生产镜像数据：

- main / 3 × 18 TB RAID0
- main / 3 × 10 TB RAID0
- ops / ~98 TB XFS volume

它们之间不是“primary / cache”层级关系；具体哪些 mirrors 放在哪个 pool，将根据容量、同步方式、Serving 拓扑和 LMT 集成方式后续设计。

### LMT Constraint

当前冻结的 LMT v0.9.0-rc.1 在一个 Agent 配置中只有一个 `mirror_root` 和一个 `publication_root`，且 Atomic publication 要求两者位于同一 mounted local filesystem。

因此，同一台 `lzu-mirrors-main` 上两个独立生产文件系统如何同时承载 LMT-managed mirrors，是当前必须保留的开放问题。

在正式设计完成前，不通过以下方式偷偷绕过这个约束：

- mergerfs
- 隐式 cross-filesystem publication
- 不透明 symlink / mount trick
- 其他改变 LMT durability / Atomic 语义的组合层

如果真实生产需求证明 LMT 需要多 storage pool 能力，再作为明确的后续能力重新评估，而不是在基础设施层伪装成一个文件系统。

## 6. Other Devices

### 2 × 480 GB Intel SSD

不再预设组成系统 RAID1。

其中一块是否作为系统盘、另一块承担何种生产或辅助职责，正式迁移前再决定。

### 1 TB NVMe

用途待定。可能承担高 IOPS 数据，例如 observability hot data、cache 或 temporary workspace，但正式角色需要根据具体型号和后续工作负载决定。

### 32 GB Optane

v1 不为了“使用 Optane”而主动增加 cache / journal 层。只有实际 benchmark 证明明确收益时才进入关键 I/O path。

### 300 GB HDD

是否继续作为系统盘、local recovery / auxiliary disk 或直接退出生产，尚未决定。

## 7. Ops Node

`lzu-mirrors-ops` 是正式生产主存储节点之一。

它同时可以在迁移阶段承担 temporary buffer / recovery source，但这些只是附加职责，不代表其生产存储地位较低。

是否运行第二个 LMT Agent、是否直接承担公网 Serving，以及 main / ops 之间如何分配 mirror，留到 Network / Serving / Sync Policy 设计阶段。

## 8. Open Decisions

尚未冻结：

- RAID chunk size
- filesystem mkfs parameters
- mount options
- exact directory layout
- capacity reserve
- 两个本地 pool 与 ops 之间的 mirror placement
- 同一 main 主机两个文件系统与 LMT 的集成方式
- NVMe / Optane / 两块 480 GB SSD / 300 GB disk 的最终职责
- ops 节点 Serving 与 LMT 拓扑
