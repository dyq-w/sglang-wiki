---
id: dspark-moe-runtime-hip-oom
kind: failure
layer: memory
components: [sglang, dspark, lightop, int8-moe, hip-graph]
signatures:
  - 'torch\.OutOfMemoryError: HIP out of memory'
  - 'lightop/.*/fused_moe/int8_marlin\.py'
signature-sources:
  - "external:pytorch:runtime allocator error"
  - "external:lightop-installed-package:_lmslim_native/layers/fused_moe/int8_marlin.py:217,283"
detection: |
  多个 DP/TP rank 在同一秒于 DSpark target forward 的 INT8 MoE 临时 tensor 分配处 OOM。
masquerades-as:
  - hip-out-of-memory
  - eoferror-dataparallelcontroller
repro-condition:
  - "observed: TP=8, DP=8, EP=1"
  - "observed: speculative_algorithm=DSPARK"
  - "observed: quantization=slimquant_marlin"
  - "observed: mem_fraction_static=0.96"
  - "observed: decode HIP graph enabled"
applies-to:
  sglang: "runtime represented by 2026-08-14 log"
  lightop: "installed _lmslim_native implementation; exact version not present in host"
status: suspected
first-seen: 2026-08-14
fixed-in:
verify: |
  # PASS only after controlled reproduction and one discriminating change.
  # Suggested matrix: mem_fraction_static 0.96 -> lower; DSpark on/off;
  # decode graph on/off; request/batch smaller; quant vs non-quant.
  # ENV-MISMATCH when the same model/runtime/GPU is unavailable.
sources:
  - "raw/20260814-dspark-hip-oom.md"
  - "local-log:/public/home/duanyongqiang/workspace/ifb_decode_dspark_20260814_131243.log#L898-L1367"
---

# DSpark INT8 MoE 运行时 HIP OOM

## 现象

多个 rank 在 `13:54:35` 同秒报 OOM：

- `torch.empty(cache13)` 尝试分配约 768 MiB；
- per-token quant 的 `torch.empty_like(..., dtype=int8)` 尝试分配约 48 MiB；
- 部分 GPU 只剩 0-204 MiB 可用；
- 日志显示约 122.78 MiB 在 HIP Graph private pools 中。

## 根因

已确认：运行时 INT8 MoE 前向所需临时 tensor 无法分配。

仍未完成排他验证：无法仅凭日志确认是 `mem_fraction_static=0.96`、DSpark 额外模型/缓存、graph pool、请求形状、临时 cache 策略或它们的组合导致余量不足。因此状态为 `suspected`。

## 定位过程

1. 日志在此前可正常服务和 decode，不是 launch 或权重加载阶段。
2. 7 个以上 rank 同秒 OOM，是同一工作负载触发的重复失败，不应逐 rank 分别诊断。
3. traceback 落到 DSpark target worker → MoE → LightOp INT8 Marlin 分配点。
4. `full token usage` 仅约 0.13-0.16，说明不能只用 KV token pool 使用率解释总显存余量；还要计入权重、draft/target、graph pool 和临时 workspace。

## 修复 / 规避

按单变量顺序复测：

1. 降低 `mem_fraction_static`，为运行时临时 tensor 留出余量；
2. 保持请求不变，关闭 decode HIP graph；
3. 保持目标模型不变，关闭 DSpark；
4. 降低请求长度 / 并发 / batch；
5. 用相同模型跑非量化或不同 MoE backend 做归因。

只有一个变量改变后稳定不复现，才能把对应原因升级为 confirmed。

## 复发判据

- 相同请求形状下，多个 rank 同秒在 MoE 临时 tensor 分配处 OOM；
- 仅出现任意 HIP OOM 不足以判定为本条复发，必须匹配 DSpark + INT8 MoE 调用链。
