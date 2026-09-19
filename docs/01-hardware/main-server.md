# lzu-mirrors-main

## Server

- Model: Dell PowerEdge R740xd
- Architecture: x86-64

## Storage Controller

- Dell PERC H730P Mini
- Broadcom / LSI MegaRAID SAS-3 3108
- Current controller mode: **HBA**
- No PERC Virtual Disk
- Physical disks are exposed directly to Linux

HBA 模式是后续存储设计的重要前提，目前没有理由切回硬 RAID 模式。

## Installed Disks

### 2 × Intel SSDSC2KB480G7R

- 480 GB SATA enterprise SSD
- 512 B logical / 4096 B physical sector
- 当前 SMART overall 正常
- Power-on hours: 约 6.8 万小时
- Host writes: 每块约 678 TB
- Percent Life Remaining: 74
- Power Loss Protection test 正常
- 当前无 reallocated / uncorrectable / CRC error

两块盘仍可继续使用，但应视为长期服役设备。

### 3 × HGST HUH721010AL

- 10 TB enterprise HDD
- 512e / 4 KiB physical sector
- Power-on hours: 约 6.85–6.91 万小时
- SMART overall: PASSED
- 当前无 reallocated / pending / offline uncorrectable / CRC error

### 3 × WDC WUH721818AL

- 18 TB enterprise HDD
- 512e / 4 KiB physical sector
- Power-on hours: 约 4.14–4.20 万小时
- SMART overall: PASSED
- 当前无 reallocated / pending / offline uncorrectable / CRC error

六块主要 HDD 已启动 extended SMART self-test；正式迁移前应确认最终结果。

### 1 × Seagate ST300MP0026

- 约 300 GB HDD
- 当前为旧系统盘
- 新架构中的用途尚未确定

## Planned but Not Installed

以下设备尚未安装，不属于当前硬件基线：

- 1 × 1 TB NVMe SSD
- 若干 × 32 GB Optane

最终用途应在设备到位并确认具体型号、健康状态和性能后决定。
