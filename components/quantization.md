# SGLang 量化与并行切分基线

调查基线：SGLang commit `bd1f2823e5743437f4e1773d9b0ea5a167de30ca`，工作树存在用户未提交修改。因此本页描述参数框架的稳定契约；涉及当前修改的 EP/DSpark 细节需复核 diff。

## 参数类型

来源：`python/sglang/srt/layers/parameter.py`。

| 参数类型 | 并行行为 | 典型用途 |
|---|---|---|
| `ModelWeightParameter` | row + column parallel | 普通线性权重 |
| `GroupQuantScaleParameter` | row + column parallel | group-wise scale |
| `ChannelQuantScaleParameter` | column parallel | per-output-channel scale |
| `BlockQuantScaleParameter` | row + column parallel | block-wise scale |
| `PerTensorScaleParameter` | 不按 TP rank 切；融合层按 shard id 映射 | per-tensor scale |
| `PackedColumnParameter` | column parallel + packed index 调整 | GPTQ/Marlin 等 packed 权重 |

## w8a8 / w4a8 线性层

- `w8a8_int8.py:229-244`
- `slimquant_w4a8.py:281-296`

两者都创建：

```text
weight:       output_dim=0, input_dim=1
weight_scale: ChannelQuantScaleParameter(output_dim=0)
```

因此 per-channel scale 必须与输出通道分片一致。

## per-tensor scale

`parameter.py:404-420` 对 row/column parallel loader 移除 `tp_rank`，即每个 rank 加载同一 logical scale；QKV/融合层只通过 shard id 选择对应 scalar。

## 融合层一致性

`w8a8_int8.py:164-193` 会遍历 fused mapping 的各 shard。部分 FLOAT、部分量化会主动抛：

```text
Detected some but not all shards ... are quantized
```

不要绕过此检查后继续运行。

## 诊断打印清单

遇到 TP/EP + 量化问题时，每个相关 rank 至少记录：

- 参数完整名和类名；
- weight / scale shape、dtype、stride；
- output_dim / input_dim / packed_dim；
- TP rank / TP size / EP rank / local expert id；
- loader 的 shard id；
- scale 前若干值或统计值；
- `process_weights_after_loading` 前后 layout。

## 最小实验

1. TP=1 vs TP>1；
2. EP off vs on；
3. quant vs non-quant；
4. SGLang 路径 vs 单算子 reference；
5. 首个数值偏离层，而不是只比较最终文本。
