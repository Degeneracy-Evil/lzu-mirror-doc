# mdraid 建组与磁盘更换

**Status: Draft — 未冻结、未在目标硬件上验证**

> 本文步骤尚未在 `lzu-mirrors-main` 上完整执行验证。chunk size、mkfs 参数、mount options、`/boot` 布局，以及各 pool 到 `/srv/*` 的映射，均待 `docs/03-storage/` 冻结后定稿。
>
> PERC H730P Mini 保持 **HBA 模式**，所有 RAID 由 Linux `mdadm` 管理，不使用硬件 RAID，也不使用 MegaCLI/StorCLI 的 VD/BBU/Patrol Read 等功能。

## 1. Scope

- 从裸盘建立系统盘 RAID1 与数据池 RAID0。
- 系统盘单盘故障后的更换与重建。
- 数据池（RAID0）故障后的处理原则。

不覆盖：OS 安装本身（见 `docs/02-os-host/bootstrap.md`）、pool 内数据布局、LMT 重新同步流程。

## 2. 设计前提

来自 `docs/03-storage/architecture.md`：

```text
2 × 480 GB Intel SSD   →  mdadm RAID1  →  ext4  →  /
3 × 18 TB WDC         →  independent RAID0  →  XFS  →  primary
3 × 10 TB HGST        →  independent RAID0  →  XFS  →  bulk
```

- 系统盘是唯一需要冗余的块设备，其余镜像数据可重建。
- 两个三盘 RAID0 互为独立故障域，不合并为六盘 RAID0。
- 任一块系统 SSD 缺失时，主机必须仍能独立启动（`docs/02-os-host/architecture.md`）。

## 3. 磁盘识别

**不要用 `/dev/sdX` 作为长期标识**，顺序会在重启或换盘后改变。优先使用：

```bash
lsblk -o NAME,SIZE,TYPE,MODEL,SERIAL
ls -l /dev/disk/by-id/
ls -l /dev/disk/by-path/
smartctl -i /dev/sdX          # HBA 直通，无需 -d megaraid
```

约定（示例，实际值在执行时填写）：

```bash
SSD_A=/dev/disk/by-id/ata-INTEL_SSDSC2KB480G7R_XXXXXXXX
SSD_B=/dev/disk/by-id/ata-INTEL_SSDSC2KB480G7R_YYYYYYYY
HDD_18_1=/dev/disk/by-id/ata-WDC_WUH721818AL_...
HDD_18_2=/dev/disk/by-id/ata-WDC_WUH721818AL_...
HDD_18_3=/dev/disk/by-id/ata-WDC_WUH721818AL_...
HDD_10_1=/dev/disk/by-id/ata-HGST_HUH721010AL_...
HDD_10_2=/dev/disk/by-id/ata-HGST_HUH721010AL_...
HDD_10_3=/dev/disk/by-id/ata-HGST_HUH721010AL_...
```

机柜槽位 ↔ 设备映射：优先看 `/dev/disk/by-path/`；如确需点 locate LED，使用 Dell `perccli`（属非 APT 手工工具，需按包管理策略登记工单、只从 Dell 官方获取）。

## 4. 系统盘建组（2 × SSD，RAID1）

### 4.1 分区

每块 SSD：ESP（`ef00`）+ boot（`fd00`）+ root（`fd00`）。

```bash
for d in "$SSD_A" "$SSD_B"; do
  sgdisk --zap-all "$d"
  sgdisk -n1:0:+1G -t1:ef00 -c1:ESP  "$d"
  sgdisk -n2:0:+2G -t2:fd00 -c2:boot "$d"
  sgdisk -n3:0:0   -t3:fd00 -c3:root "$d"
done
partprobe
```

### 4.2 创建阵列

```bash
# /boot：metadata 1.0（superblock 在盘尾），GRUB 读取最稳
mdadm --create /dev/md0 --level=1 --raid-devices=2 --metadata=1.0 \
  --name=host:boot "${SSD_A}-part2" "${SSD_B}-part2"

# /：metadata 1.2（默认）
mdadm --create /dev/md1 --level=1 --raid-devices=2 --metadata=1.2 \
  --name=host:root --bitmap=internal "${SSD_A}-part3" "${SSD_B}-part3"
```

### 4.3 文件系统与挂载

```bash
mkfs.ext4 -L boot /dev/md0
mkfs.ext4 -L root /dev/md1
mount /dev/md1 /mnt
mkdir -p /mnt/boot /mnt/boot/efi
mount /dev/md0 /mnt/boot
mount "${SSD_A}-part1" /mnt/boot/efi
```

### 4.4 引导（UEFI）

ESP 不能放进 mdraid，需**每块盘各一个**，并保证两块盘的 ESP 上都有引导文件，才能单盘启动。

```bash
grub-install --efi-directory=/mnt/boot/efi --bootloader-id=ubuntu --recheck /dev/disk/by-id/<SSD_A_base>
# 第二块盘：挂载其 ESP 后再安装一份，或复制 EFI 目录，并加 --removable 作为兜底
```

> ESP 镜像与 NVRAM 条目的具体做法尚未验证，是本文最需要在真机上确认的部分。

## 5. 数据池建组（3 + 3 HDD，RAID0）

```bash
for d in "$HDD_18_1" "$HDD_18_2" "$HDD_18_3" "$HDD_10_1" "$HDD_10_2" "$HDD_10_3"; do
  sgdisk --zap-all "$d"
  sgdisk -n1:0:0 -t1:fd00 -c1=data "$d"
done
partprobe

# primary：3 × 18 TB
mdadm --create /dev/md2 --level=0 --raid-devices=3 --metadata=1.2 \
  --chunk=512 --name=host:primary \
  "${HDD_18_1}-part1" "${HDD_18_2}-part1" "${HDD_18_3}-part1"

# bulk：3 × 10 TB
mdadm --create /dev/md3 --level=0 --raid-devices=3 --metadata=1.2 \
  --chunk=512 --name=host:bulk \
  "${HDD_10_1}-part1" "${HDD_10_2}-part1" "${HDD_10_3}-part1"

mkfs.xfs -f -L primary /dev/md2   # mkfs.xfs 自动识别 md 几何
mkfs.xfs -f -L bulk    /dev/md3
```

RAID0 无冗余：任意一块盘损坏即整组失效，数据从上游重新同步，不做原地修复。

## 6. 持久化与启动

```bash
# 写入阵列定义（避免重复追加）
mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo update-initramfs -u

# 文件系统用 UUID 写 fstab（示例）
blkid /dev/md0 /dev/md1 /dev/md2 /dev/md3
```

`/etc/fstab` 示例：

```fstab
UUID=<root-uuid>    /            ext4  defaults        0 1
UUID=<boot-uuid>    /boot        ext4  defaults        0 2
UUID=<primary-uuid> /srv/primary xfs   defaults        0 2
UUID=<bulk-uuid>    /srv/bulk    xfs   defaults        0 2
```

pool → `/srv/*` 的最终命名待 Storage 冻结。

## 7. 验证

```bash
cat /proc/mdstat
mdadm --detail /dev/md1
mdadm --examine /dev/disk/by-id/<SSD_A_base>-part3

# RAID1 一致性检查与修复
echo check  | sudo tee /sys/block/md1/md/sync_action
cat /sys/block/md1/md/mismatch_cnt
echo repair | sudo tee /sys/block/md1/md/sync_action
```

**单盘启动测试（强制项）**：分别在物理拔除 / `perccli` 下线 SSD_A、SSD_B 的情况下启动，确认主机仍能进入系统，阵列处于 `clean, degraded`；随后复位并观察自动重建完成。

## 8. RAID1 单盘故障更换

```bash
# 1. 确认故障成员
cat /proc/mdstat
mdadm --detail /dev/md1
dmesg | tail

# 2. 标记并移除（若未自动失败）
mdadm --manage /dev/md1 --fail   "${SSD_A}-part3"
mdadm --manage /dev/md1 --remove "${SSD_A}-part3"

# 3. 定位并物理更换（serial / by-path / locate LED）

# 4. 新盘按 4.1 做同样分区（尺寸一致）
sgdisk --zap-all "$SSD_NEW"
sgdisk -n1:0:+1G -t1:ef00 -c1:ESP  "$SSD_NEW"
sgdisk -n2:0:+2G -t2:fd00 -c2:boot "$SSD_NEW"
sgdisk -n3:0:0   -t3:fd00 -c3:root "$SSD_NEW"
partprobe

# 5. 清残留 superblock 后加入，观察自动重建
mdadm --zero-superblock "${SSD_NEW}-part2" "${SSD_NEW}-part3" 2>/dev/null
mdadm --manage /dev/md0 --add "${SSD_NEW}-part2"
mdadm --manage /dev/md1 --add "${SSD_NEW}-part3"
watch -n2 cat /proc/mdstat

# 6. 重建完成后，向新盘 ESP 安装引导并同步 EFI 内容
# 7. 确认状态
mdadm --detail /dev/md0 /dev/md1
```

## 9. RAID0 故障处理

任意一块盘故障即整组不可用，**不要尝试通过降级/scrub 恢复**。原则流程：

1. 确认硬件故障（`smartctl -a`、`dmesg`），物理更换磁盘。
2. 重新建立 RAID0：`mdadm --create` 同前（先 `--zero-superblock` 清残留）。
3. 重新 `mkfs.xfs` 并挂载。
4. 触发对应内容重新同步 / 重新发布。

## 10. 监控

主机层监控至少覆盖 `docs/02-os-host/architecture.md` §13 的健康项。具体 exporter、dashboard 与 alert policy 留给 `docs/09-observability/`；本节只写 OS 层可直接执行的部分。

### 10.1 SMART 与固件版本

```bash
apt install smartmontools
systemctl enable --now smartd

smartctl --scan
smartctl -a /dev/sdX        # SATA / SAS 盘
smartctl -a /dev/nvme0      # NVMe
```

**HBA 直通模式下，SMART 直接对真实块设备读取，不需要 RAID 控制器参数。** 若以后盘改由硬件 RAID / 非 HBA 暴露，才需要 `-d megaraid,N` 之类的写法——当前设计不涉及。

按盘类型重点关注：

- **SATA/SAS SSD**（Intel S46xx 系统盘）：`Reallocated_Sector_Ct(5)`、`Available_Reservd_Space(170/232)`、`Program_Fail_Count(171)`、`Erase_Fail_Count(172)`、`End-to-End_Error_Count(184)`、`Uncorrectable_Error_Cnt(187)`、`CRC_Error_Count(199)`、`Media_Wearout_Indicator(233)`、`Power_Loss_Cap_Test(175/235)`、温度 `190/194`。
- **HDD**（WDC / HGST）：`Reallocated_Sector_Ct(5)`、`Current_Pending_Sector(197)`、`Offline_Uncorrectable(198)`、`UDMA_CRC_Error_Count(199)`、`Raw_Read_Error_Rate(1)`、温度 `194`。
- **SAS**：`Elements in grown defect list`、`Non-medium error count`、`Total uncorrected errors`、`Current Drive Temperature`。
- **NVMe**：`Critical Warning`、`Available Spare` / 阈值、`Percentage Used`、**`Media and Data Integrity Errors`（非 0 需立即处理）**、`Warning/Critical Composite Temperature Time`。

判断方法：`Pre-fail` 类预示临期故障，`Old_age` 类反映老化；`VALUE` 低于 `THRESH` 即视为异常。

自检与日志：

```bash
smartctl -t short /dev/sdX
smartctl -t long  /dev/sdX
smartctl -l selftest /dev/sdX
```

大容量 HDD 的 long 测试耗时长且影响性能，安排在维护窗口；可用 smartd 的 `-s` 计划执行。

**固件版本必须核对。** 数据中心 SSD 有过“达到某个通电小时数后批量损坏”的固件缺陷先例；同批次同型号盘会同时爆发，RAID 冗余救不了。系统盘 Intel S46xx 已约 6.8 万小时、Percent Life 74%，尤其要先确认固件并关注公告。能用 `fwupd` 的优先用 `fwupd`，否则用厂商官方工具（属非 APT 工具，需走包管理策略）。

smartd 默认配置：

```conf
DEVICESCAN -d removable -n standby -m root -M exec /usr/share/smartmontools/smartd-runner
```

`-M test` 可在启动时发测试邮件验证告警链路。

### 10.2 mdraid 状态

```bash
cat /proc/mdstat
mdadm --detail /dev/md1
mdadm --examine /dev/disk/by-id/<SSD_A>-part3
systemctl enable --now mdmonitor
```

在 `/etc/mdadm/mdadm.conf` 配置 `MAILADDR`（或 `PROGRAM` 事件脚本），关注 `Fail`、`DegradedArray`、`SpareMissing`、`RebuildStarted/Finished` 等事件。

- RAID1：定期 scrub，`echo check` / `repair` 后看 `mismatch_cnt`（见 §7）。
- RAID0：无冗余、无可修复状态，监控重点落在单盘 SMART，而不是阵列一致性。

### 10.3 接入 Observability

主机层只需先暴露指标来源：

- SMART / 固件版本（`smartd` / `smartctl`）
- mdraid 状态（`/proc/mdstat`、`mdadm --detail`）
- NVMe 健康、文件系统容量、温度（`docs/02-os-host/architecture.md` §13）

最终 exporter、dashboard、alert 路由由 `docs/09-observability/` 决定。可参考 telegraf 的 `mdstat` / `smart` 插件，以及 USTC 的 [raid-telegraf](https://github.com/ustclug/raid-telegraf) 与 Grafana dashboard 20645，但仅作参考，不预设选型。

### 10.4 待决策

- 主机是否运行完整 MTA 发告警邮件，还是只把告警送 Observability（减少主机攻击面）。
- smartd 自检计划、温度阈值与各项告警阈值。
- 系统盘固件升级方式（`fwupd` vs 厂商工具 / 带外）。

## 11. 待冻结项

- chunk size 与 RAID0 条带参数
- mkfs 参数与 mount options
- `/boot` 布局与 ESP 镜像方法
- pool 到 `/srv/*` 的最终映射
- `perccli` 是否纳入、固件升级走 OS 还是带外
- 主机监控/告警链路（MTA vs Observability，见 §10.4）
