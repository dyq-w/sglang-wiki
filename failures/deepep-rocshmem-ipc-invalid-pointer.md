---
id: deepep-rocshmem-ipc-invalid-pointer
kind: failure
layer: distributed
components: [deepep, rocshmem, hip-ipc]
signatures:
  - 'hipIpcOpenMemHandle.*invalid device pointer'
  - 'backend_ipc\.cpp:[0-9]+'
signature-sources:
  - "external:rocshmem:third-party/rocshmem/src/ipc/backend_ipc.cpp:414"
detection: ""
masquerades-as:
  - eoferror-dataparallelcontroller
  - process-disappeared
repro-condition:
  - "observed: nnodes=2, TP=16, DP=16, EP=16"
  - "observed: moe_a2a_backend=deepep"
  - "observed: decode PD role"
applies-to:
  sglang: "0.5.15 DAS family (observed)"
  dtk: "2604 runtime from deployment record"
  deep_ep: "unknown build; log path contains bundled RocSHMEM"
status: confirmed
first-seen: 2026-08-15
fixed-in:
verify: |
  # PASS: 最早决定性错误是 hipIpcOpenMemHandle invalid device pointer，
  # 且后续 EOFError/SIGQUIT 发生在其后。
  rg -n 'hipIpcOpenMemHandle|DataParallelController|raise EOFError|Received sigquit' "$LOG"
  # ENV-MISMATCH: 当前节点没有相同多节点/EP/RocSHMEM 环境。
  # NOT-APPLICABLE: 未启用 DeepEP/RocSHMEM。
sources:
  - "raw/20260815-deepep-ipc-invalid-pointer.md"
  - "local-log:/public/home/duanyongqiang/workspace/eplb_20260815_181502.log#L628-L1085"
---

# DeepEP / RocSHMEM HIP IPC invalid pointer

## 现象

日志最早的决定性错误：

```text
hipIpcOpenMemHandle(...): invalid device pointer (17)
... rocshmem/src/ipc/backend_ipc.cpp:414
```

几百行后才出现 `DataParallelController`、`EOFError` 和 `Received sigquit`。

## 根因

已确认到的根因边界：**DeepEP 初始化或使用 RocSHMEM IPC backend 时，RocSHMEM 无法在当前进程打开某个 HIP IPC memory handle。**

尚未确认更底层是哪一种环境条件导致 handle 无效；候选包括设备可见性/rank 映射不一致、peer access/IPC 生命周期问题、不同节点或进程看到的设备集合不一致。不得把任一候选写成 confirmed。

## 定位过程

1. 全日志中最早错误位于 628 行，而不是尾部 1085 行。
2. 错误路径明确落在 RocSHMEM `backend_ipc.cpp`。
3. 随后的多份 `Fatal Python error: Aborted` 是同一失败在多 rank 的扩散。
4. DP controller 的 EOFError 是子进程死亡后读取 pipe 失败，已排除为根因。

## 修复 / 规避

按顺序验证：

1. 对每个 rank 打印 `HIP_VISIBLE_DEVICES`、本地 rank、实际 device id、node rank。
2. 检查两个节点的设备枚举和 RocSHMEM/DeepEP 构建是否一致。
3. TP/EP 二分：先缩到单节点和更小 EP；确认失败是否只在跨节点出现。
4. 临时切换非 DeepEP dispatcher 仅用于归因；能运行不代表修复 IPC。
5. 修复后必须恢复原 TP/EP/PD 组合复测。

## 复发判据

- 首错重新命中 `hipIpcOpenMemHandle ... invalid device pointer`；
- 后续可能仍伪装成 EOFError / SIGQUIT；
- 仅看到 EOFError 不能判定复发，必须找到上游 IPC signature。
