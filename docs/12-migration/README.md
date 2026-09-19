# Migration / Cutover

**Status: Not started**

本模块后续设计：

- maintenance / shutdown procedure
- data freeze
- local migration
- ops node as temporary buffer
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
