# Weight / quantization playbook

## 进入条件

- 权重加载期间 key / shape 不匹配；
- 缺 scale / zero point；
- pack/layout 错误；
- 只在量化模型或 TP/EP 切分下异常。

## 证据

记录 model config、quant config、量化方法、backend，以及报错参数完整名。每个相关 rank 打印：

```text
parameter class
weight/scale shape dtype stride
input_dim/output_dim/packed_dim
TP rank/size, EP rank/size
shard id and loader
process_weights_after_loading 前后 layout
```

## 阶段判断

- 加载时直接 shape/key/scale 错 → weights-quant；
- 加载成功、首次 forward kernel 报错 → kernel，但先验证 layout；
- 仅 TP>1 数值错 → accuracy + scale split；
- 仅 EP 打开后错 → expert ownership、weight repack、dispatcher/backend 契约。

## 固定二分矩阵

1. quant vs non-quant；
2. TP=1 vs TP>1；
3. EP off vs on；
4. fused layer vs 对应未融合/reference；
5. LightOp backend vs Triton/reference（若功能等价）；
6. 权重加载前后分别检查 shape/stride。

## 代码契约

见 [components/quantization.md](../components/quantization.md)：

- per-channel scale 随 output dim；
- per-tensor scale 不按 TP rank 切；
- fused shard 精度必须一致；
- packed weight 必须校验 tile / pack factor。

## 高价值条目

- [融合 shard 混合精度](../failures/quant-fused-shards-mixed-precision.md)
- [scale TP 切分错误](../failures/quant-scale-tp-split.md)

## 最小正确性实验

固定一个小输入，保存 TP=1 reference；逐层找 TP>1 的**首个数值偏离层**。不要只比较最终文本，否则定位粒度过粗。
