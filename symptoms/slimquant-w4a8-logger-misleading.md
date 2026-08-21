---
id: slimquant-w4a8-logger-misleading
kind: symptom
signatures:
  - '\[slimquant_w4a8_marlin\] requested_backend=.*resolved_backend='
signature-sources:
  - "python/sglang/srt/layers/quantization/slimquant_w4a8_marlin.py:110-114"
diagnostic-value: none
means: |
  模块级 import-time 横幅，不表示 w4a8 scheme 生效。
  `--quantization slimquant_marlin` 会导入该模块，横幅无条件打印。
action: |
  改看 MoE method 类名确定实际方案，例如
  `quant_method=CompressedTensorsW8A8Int8MarlinMoEMethod` 即 INT8 W8A8。
  不要据此判断精度问题。
possible-causes: []
---

# `[slimquant_w4a8_marlin]` 日志不代表 w4a8 生效

## 含义

跑 w8a8 模型时日志出现：

```text
INFO sglang.srt.layers.quantization.slimquant_w4a8_marlin:
     [slimquant_w4a8_marlin] requested_backend=auto, resolved_backend=lightop
```

这容易被读成「w8a8 模型被当成 w4a8 跑了」。不是。

该 `logger.info` 位于 `slimquant_w4a8_marlin.py:110`，处于模块顶层（无缩进，位于首个
`class` 之前），import 该模块即执行。`--quantization slimquant_marlin` 会导入它，
所以无论最终选中哪个 scheme，这行都会打印。它报告的是 w4a8 TPMoE backend 的解析结果
（lightop / aiter 可用性），与本次请求实际使用的量化方案无关。

## 怎么确认实际方案

看 MoE method 类名：

```text
FlashInfer TRTLLM MoE deferred finalize is disabled
  (moe_runner_backend=auto, quant_method=CompressedTensorsW8A8Int8MarlinMoEMethod)
```

`CompressedTensorsW8A8Int8MarlinMoEMethod` 即 INT8 W8A8。该类定义在**另一个文件**
`python/sglang/srt/layers/quantization/compressed_tensors/compressed_tensors_moe_marlin.py:229`，
不在 `slimquant_w4a8_marlin.py` 内。

代码层是单一强制通路
（`compressed_tensors/compressed_tensors_moe_marlin.py:221-225`）：

```python
if quant_config._is_dynamic_token_w8a8(weight_quant, input_quant):
    return CompressedTensorsW8A8Int8MarlinMoEMethod(quant_config)
else:
    raise RuntimeError(
        f"Slimquant_marlin does not support the FusedMoe scheme: ..."
    )
```

非 w8a8 直接抛异常，不存在静默降级到 w4a8 的路径。
`CompressedTensorsW8A8Int8MarlinMoEMethod.__init__` 另有两道校验：权重必须
channelwise、激活必须 dynamic per-token，静态 scale 会被拒。服务能推进到 warmup
即说明三道检查全过。

## 为什么 diagnostic-value 是 none

这行日志既不指示故障，也不指示配置错误。出现它时不要据此排查精度或量化问题；
按阶段继续定位真实首错。
