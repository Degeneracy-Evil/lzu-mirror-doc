# 重构路线

当前建议的推进顺序：

```text
总体架构
  ↓
硬件摸底
  ↓
LMT
  ↓
OS / Host
  ↓
Storage 最终冻结
  ↓
Network
  ↓
Security baseline
  ↓
Nginx / Serving
  ↓
Mirror Sync Policy
  ↓
Observability / Logging
  ↓
Frontend
  ↓
Backup / DR
  ↓
Migration Plan
  ↓
Validation / Benchmark
  ↓
Production Cutover
  ↓
Long-term Operations
```

实际执行允许局部并行，但依赖关系应尽量保持：

- OS 与 Storage 是所有生产服务的基础。
- Network 与 Security 应在公网 Serving 上线前闭环。
- Nginx、同步策略和 LMT 共同构成镜像站核心服务路径。
- Observability 应在正式迁移前部署完成，以便迁移和上线过程本身可观测。
- Migration、Validation 和 Runbook 必须在正式停站前完成，而不是上线后补写。
