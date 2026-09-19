# LMT Status

## Milestones

```text
M1 complete
M2 complete
M3 complete
M4 complete
M4 development frozen
M5 not authorized
```

## Current Release

```text
v0.9.0-rc.1
```

Release commit:

```text
4e5f55078070288832b9e098cbf5f9951fd18c19
```

## Current Scope

LMT 已具备：

- centralized Server / Agent architecture
- Mirror / Run / Attempt state management
- deterministic scheduling
- process supervision
- cancellation and reconciliation
- crash recovery
- structured logging / metrics integration points
- M4 Atomic Publication

M4 Atomic Publication 已完成：

- fresh generation semantics
- hard-link reuse
- `RENAME_EXCHANGE`
- write-ahead / durability ordering
- full-writer fencing
- GC / admission coordination
- crash recovery
- Move semantics
- forward-only M3 → M4 compatibility

Atomic publication 的 `mirror_root` 与 `publication_root` 必须位于同一 mounted local filesystem。

## Validation

LMT 已经过：

- unit / integration / installer / release tests
- bare-metal Server / Agent deployment tests
- cancel / restart / reconciliation tests
- controlled production trial
- real XFS Atomic publication smoke test

## Freeze Policy

当前进入 **production release candidate / stabilization** 阶段：

- 不进行 speculative M5 development；
- 只有真实生产证据暴露 correctness、security、compatibility 或 installation blocker 时才重新进入功能修改；
- 后续重点是 production rollout 与整个镜像站的系统集成。
