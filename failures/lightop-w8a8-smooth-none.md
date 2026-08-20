---
id: lightop-w8a8-smooth-none
kind: failure
layer: kernel
components: [lightop, w8a8]
signatures: []
signature-sources: []
detection: |
  `lightop.gemm_w8a8_smooth(...)` 返回 `(False, None)`；调用方忽略 status 后在下游对 None 报错或静默回退。
masquerades-as:
  - process-disappeared
repro-condition:
  - "LIGHTOP_GEMM_W8A8_SMOOTH != 1"
  - "M<=16 且没有精确 tuned config"
  - "M>16 且 ASM 目录不存在"
  - "M>16 且 bias 非空"
  - "M>16 且 K<128"
applies-to:
  lightop: "commit 3797bcc4; python/lightop/gemmopt.py:75-101"
status: confirmed
first-seen: 2026-08-20
fixed-in:
verify: |
  # PASS: 对每个受支持/不支持分支断言 status 和 output 一致；
  # 不支持时调用方不得使用 output。
  # ENV-MISMATCH: 当前节点无 torch/lightop/GPU。
sources:
  - "baseline/2026-08-20-bootstrap.md"
---

# gemm_w8a8_smooth 返回 `(False, None)`

## 现象

函数自身不抛错。调用方若只取 output，可能在更下游才出现 `NoneType`、shape 或算子错误，使错误位置远离真正决策点。

## 根因

`gemm_w8a8_smooth` 用 `(status, output)` 表达是否支持当前输入；多个正常分支会返回 `(False, None)`。

## 定位过程

记录：

- M/N/K；
- bias 是否为空；
- `LIGHTOP_GEMM_W8A8_SMOOTH`；
- ASM 目录；
- M<=16 时 tuned config 是否精确命中；
- 调用方是否检查 status。

## 修复 / 规避

- 调用方必须按 `status` 路由到 fallback，禁止直接使用 output；
- 在 fallback 时记录一次结构化原因；
- 对每个返回 False 的条件建立单元测试。

## 复发判据

函数返回 False/None，但调用方继续使用 None，或无记录地切到意外 backend。
