# LZU Mirror Documentation

兰州大学镜像站重构项目的设计、规划与运维文档仓库。

本仓库是整个镜像站重构项目的**设计事实源（source of truth）**。代码仓库只保存对应组件的实现细节；跨组件的系统架构、基础设施规划、迁移方案和长期运维约定统一记录在这里。

## 当前阶段

目前已经完成：

- 整体重构范围与第一版架构划分；
- 主服务器现有硬件与存储基线摸底；
- LZU Mirror Tools（LMT）M1–M4 开发与 v0.9.0-rc.1 发布冻结；
- 存储架构第一版方向；
- Ubuntu 26.04 OS / Host 第一版设计。

尚未开始或尚未完成的主要模块包括 Nginx Serving、同步策略、网络、安全、可观测性、前端、备份与灾难恢复、迁移、验证和长期运维。

## 文档导航

- [00 Overview](docs/00-overview/README.md)
- [01 Hardware](docs/01-hardware/README.md)
- [02 OS / Host](docs/02-os-host/README.md)
- [03 Storage](docs/03-storage/README.md)
- [04 LMT](docs/04-lmt/README.md)
- [05 Sync Policy](docs/05-sync/README.md)
- [06 Serving](docs/06-serving/README.md)
- [07 Network](docs/07-network/README.md)
- [08 Security](docs/08-security/README.md)
- [09 Observability](docs/09-observability/README.md)
- [10 Frontend](docs/10-frontend/README.md)
- [11 Backup / DR](docs/11-backup-dr/README.md)
- [12 Migration](docs/12-migration/README.md)
- [13 Validation](docs/13-validation/README.md)
- [14 Operations](docs/14-operations/README.md)

另外：

- [Architecture Decisions](decisions/README.md)：记录值得长期保留背景的重要架构决策。
- [Runbooks](runbooks/README.md)：记录实际部署后可执行的运维与故障恢复流程。

## 相关仓库

- [Degeneracy-Evil/lzu-mirror-tools](https://github.com/Degeneracy-Evil/lzu-mirror-tools)：LZU Mirror Tools（LMT）实现。

## 文档约定

- 主体文档维护“当前正确状态”，历史变化交给 Git。
- 不按日期堆积设计快照。
- 未冻结设计必须明确标记为 provisional。
- 临时排查命令、一次性测试输出不进入长期架构文档。
- 真正影响长期架构的选择，必要时在 `decisions/` 中单独记录。
