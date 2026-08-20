---
id: quant-fused-shards-mixed-precision
kind: failure
layer: weights-quant
components: [sglang, w8a8, fused-linear]
signatures:
  - "Detected some but not all shards of .* are quantized"
  - "All shards of fused layers to have the same precision"
signature-sources:
  - "sglang/python/sglang/srt/layers/quantization/w8a8_int8.py:164-193"
detection: ""
masquerades-as: []
repro-condition:
  - "融合映射（如 QKV 或 gate/up）中部分 shard 为 FLOAT，其他 shard 为量化权重"
applies-to:
  sglang: "commit bd1f2823 checkout; w8a8_int8 implementation"
status: confirmed
first-seen: 2026-08-20
fixed-in:
verify: |
  # PASS: 同一 fused layer 的所有 shard 在 quant_description 中精度一致。
  # FAIL: 一部分为 FLOAT、一部分为量化；代码应稳定抛出该 ValueError。
sources:
  - "baseline/2026-08-20-bootstrap.md"
---

# 融合层 shard 精度不一致

## 现象

权重加载/量化方法选择阶段报：

```text
Detected some but not all shards of <prefix> are quantized.
All shards of fused layers to have the same precision.
```

## 根因

融合层由多个逻辑投影组成；配置把其中一部分标为 FLOAT、另一部分标为量化。当前实现要求融合后的所有 shard 精度一致。

## 定位过程

1. 从 `<prefix>` 找到 fused mapping 中的所有逻辑 shard。
2. 对每个 `<shard_prefix>.weight` 打印 `quant_description`。
3. 确认是否仅部分命中 ignore / FLOAT 规则。
4. 此错误发生在权重/量化阶段，不进入 kernel 或 distributed 路由。

## 修复 / 规避

- 调整量化描述或 ignore 规则，使融合层所有 shard 一致；
- 如果业务必须混合精度，需先拆除融合或实现显式混合精度加载，不能绕过检查后继续运行。

## 复发判据

同一 fused prefix 的 shard 精度集合包含 FLOAT 和量化两类。
