# Migration / Cutover

**Status: Not started**

本模块后续设计：

- maintenance / shutdown procedure
- data freeze
- local migration
- 利用 ops 生产存储的可用空间作为迁移缓冲
- new RAID/filesystem creation
- Ubuntu 26.04 reinstall
- LMT deployment
- Nginx deployment
- mirror-by-mirror import
- DNS / traffic cutover
- rollback
- maintenance page
- production checklist

已确定：本次重构允许完整停站，并优先保留本地已有几十 TiB 镜像数据，避免不必要的公网全量重拉。

`lzu-mirrors-ops` 本身是正式主存储节点；迁移期间使用其空闲容量做 buffer 只是临时附加用途。
