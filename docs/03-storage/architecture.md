# Storage Architecture

> **Provisional**：本设计尚未冻结。正式迁移前允许调整。

## 1. Design Principles

1. 镜像内容属于可重建数据，不为几十 TiB 镜像副本投入大规模冗余容量。
2. 容量利用率优先。
3. 控制面和配置等小体量关键状态与镜像大数据采用不同可靠性策略。
4. 允许完整停站迁移，不为在线迁移引入额外复杂度。
5. 优先通过本机磁盘和 `lzu-mirrors-ops` 搬迁已有数据，避免从公网重新全量同步。
6. 不把六块 HDD 合并为一个 RAID0，当前更倾向两个独立三盘故障域。

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
3 × 18 TB WDC
    ↓
independent RAID0
    ↓
primary mirror data pool

3 × 10 TB HGST
    ↓
independent RAID0
    ↓
bulk / cache / staging pool

2 × 480 GB Intel SSD
    ↓
RAID1
    ↓
OS / control-plane persistent state

1 TB NVMe
    ↓
TBD

32 GB Optane × N
    ↓
TBD

300 GB HDD
    ↓
TBD
```

当前文件系统方向：

- HDD RAID0：倾向 `mdadm + XFS`
- system SSD RAID1：倾向 `mdadm + ext4`

## 4. Why 3 + 3 Instead of 6-Disk RAID0

两组 HDD 在容量、型号和服役时间上本来就形成自然分组。

六盘统一 RAID0 会把整个镜像数据池绑定到任意一块盘的寿命；拆成两个三盘 RAID0 不改变“六块盘中出现故障”的总体可能性，但显著缩小单盘故障的影响范围。

尤其 10 TB 盘通电时间约 6.9 万小时，而 18 TB 盘约 4.2 万小时，因此当前不希望把两组盘绑定成一个统一故障域。

## 5. Primary vs Bulk

### Primary pool

3 × 18 TB：

- 正式镜像数据
- LMT Atomic publication 所需数据
- 对公网 Serving 的主数据树

### Bulk pool

3 × 10 TB：

- cache
- proxy data
- staging
- repack
- 临时迁移数据
- 其他可快速重建的低价值内容

当前 LMT v0.9.0-rc.1 的 Agent 只有一个 `mirror_root` 和一个 `publication_root`，Atomic 要求二者位于同一 mounted local filesystem。因此不为了利用第二个 pool 而引入跨文件系统 publication、mergerfs 或其他绕过 LMT 存储模型的方案。

## 6. Other Devices

### 1 TB NVMe

用途待定。可能承担高 IOPS、可重建的数据，例如 observability hot data、cache 或 temporary workspace，但正式角色需要根据具体型号和后续工作负载决定。

### 32 GB Optane

v1 不为了“使用 Optane”而主动增加 cache / journal 层。只有实际 benchmark 证明明确收益时才进入关键 I/O path。

### 300 GB HDD

是否作为 local recovery / auxiliary disk 保留，尚未决定。

## 7. Ops Node

`lzu-mirrors-ops` 当前定位：

- secondary storage
- migration buffer
- recovery source

是否成为第二个 LMT Node 或承担 Serving，留到 Network / Serving 设计阶段。

## 8. Open Decisions

尚未冻结：

- RAID chunk size
- filesystem mkfs parameters
- mount options
- exact directory layout
- capacity reserve
- primary / bulk 最终内容分配
- NVMe / Optane / 300 GB disk 的最终职责
- ops 节点最终生产角色
