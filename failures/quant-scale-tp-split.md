---
id: quant-scale-tp-split
kind: failure
layer: accuracy
components: [sglang, tensor-parallel, quantization]
signatures: []
signature-sources: []
detection: |
  TP=1 输出正确，TP>1 稳定出现乱码、NaN 或显著数值漂移；权重加载无报错。各 rank 的 scale shape/value 与参数类型契约不一致。
masquerades-as: []
repro-condition:
  - "量化模型"
  - "TP>1"
  - "scale 的并行切分方向或 shard 映射错误"
applies-to:
  sglang: "parameter.py scale parameter model; specific incident not yet captured"
status: suspected
first-seen: 2026-08-20
fixed-in:
verify: |
  # PASS requires a controlled TP=1 vs TP>1 comparison and per-rank scale dump.
  # ChannelQuantScaleParameter: scale follows output_dim partition.
  # PerTensorScaleParameter: same logical scalar on every TP rank; fused layers map by shard id only.
  # ENV-MISMATCH: target model or multi-GPU environment unavailable.
sources:
  - "sglang/python/sglang/srt/layers/parameter.py:358-420"
  - "sglang/python/sglang/srt/layers/quantization/w8a8_int8.py:229-244"
  - "sglang/python/sglang/srt/layers/quantization/slimquant_w4a8.py:281-296"
---

# 量化 scale 的 TP 切分错误

## 现象

- TP=1 正常；
- TP>1 加载成功但输出乱码、NaN 或数值漂移；
- 非量化模型正常；
- 错误可能没有任何 exception。

## 根因

候选机制：scale 使用了错误参数类型或错误切分维度。

代码契约：

- `ChannelQuantScaleParameter` 是 column-parallel scale，随 output dim 切分；
- `PerTensorScaleParameter` 不按 TP rank 切分，融合层只按逻辑 shard id 选择 scalar；
- w8a8 / slimquant w4a8 线性层的 per-channel weight scale 使用 `output_dim=0`。

尚无具体真实故障日志和排他实验，因此保持 suspected。

## 定位过程

1. 固定输入和随机种子，比较 TP=1 / TP=2 / 目标 TP 的首个偏离层。
2. 对偏离层按 rank dump weight shape、scale shape、scale 前若干值和参数类名。
3. 验证 per-channel scale 拼接后是否等于 TP=1 全量 scale。
4. 验证 per-tensor scale 是否在所有 rank 相同。
5. 用非量化权重复测，排除并行通信本身。

## 修复 / 规避

- per-channel scale 使用正确 `output_dim` 和与权重一致的 loader；
- per-tensor scale 不按 TP 切分；
- fused layer 按 shard id 映射 scale，而不是按 rank 截断；
- 增加 TP=1 vs TP>1 的层级数值一致性测试。

## 复发判据

TP>1 首个偏离层的 scale 与上述契约不一致，修正后相同输入恢复一致。
