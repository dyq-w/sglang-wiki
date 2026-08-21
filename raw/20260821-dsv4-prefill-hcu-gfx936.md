# 2026-08-21 DeepSeek-V4 prefill on gfx936 证据快照

已脱敏：去时间戳、PID、rank 前缀、`server_args` 全量 dump（含绝对路径与端口）。
只保留 traceback 骨架、算子探测结果与 arch 白名单。无 token / IP / 业务正文。

## 环境

| 项 | 值 |
|---|---|
| 容器 | `sglang-opt` |
| 源码挂载 | `sglang-origin/sglang-das` → `/sglang` |
| sglang-das | branch `deepseek-v4-opt`, commit `9ca1b25f7f` |
| sgl_kernel | 0.4.4 (dist-packages) |
| GPU | 8 × gfx936 (HCU/DCU BW) |
| 模型 | DeepSeek-V4-Flash INT8 w8a8（channel-wise） |
| 并行 | tp=8, ep=8, attn_cp=8, dp_attn=on |
| cuda graph | disabled |

宿主与推理容器为同一节点，pip 环境不同：所有算子探测均在容器内执行，无 ENV-MISMATCH。

## 阶段一：tilelang MHC pre 编译失败

前一行为 `TileLang begins to compile kernel ...`，说明失败在编译期而非执行期。

```text
tvm.error.InternalError: Check failed: (supported.count(mcpu)) is false:
HCU arch gfx936 not supported for MLS/GEMM_MLS; supported: gfx938, gfx92a, gfx946
```

白名单源码（容器内 tilelang，`src/target/utils.cc:154-157`）：

```cpp
static const std::set<std::string> supported = {"gfx938", "gfx92a", "gfx946"};
ICHECK(supported.count(mcpu))
    << "HCU arch " << mcpu
    << " not supported for MLS/GEMM_MLS; supported: gfx938, gfx92a, gfx946";
```

排除项：

- `SGLANG_ROCM_USE_AITER_TILELANG_MHC=1` 已设但无效 —— 只替换 stage-1 big-fuse，
  stage-0 GEMM 恒走 tilelang。
- `SGLANG_OPT_DEEPGEMM_HC_PRENORM=1` 不可用：

```text
RuntimeError: Failed to load .../deep_gemm/_C.so
libcudart.so.13: cannot open shared object file
```

aiter 路径同卡实测通过：

```text
AITER mhc_pre OK -> (8, 4, 1) (8, 4, 4) (8, 7168) torch.bfloat16
finite: True
```

## 阶段二：sgl_kernel 算子缺失

修复阶段一后前向推进到 DSA indexer，8 rank 同秒失败。

```text
  File "/sglang/python/sglang/srt/models/deepseek_v4.py", line 1349, in forward
    q, kv = self._forward_prepare(
  File "/sglang/python/sglang/srt/models/deepseek_v4.py", line 1262, in _forward_prepare
    self.indexer(
  File "/sglang/python/sglang/srt/layers/attention/dsv4/indexer.py", line 1002, in forward
    return attn_backend.forward_c4_indexer(
  File "/sglang/python/sglang/srt/layers/attention/dsv4/indexer.py", line 879, in forward_c4_indexer
    topk_transform_512(
  File "/sglang/python/sglang/kernels/ops/attention/dsv4/topk.py", line 57, in topk_transform_512
    torch.ops.sgl_kernel.deepseek_v4_topk_transform_512(
AttributeError: '_OpNamespace' 'sgl_kernel' object has no attribute
              'deepseek_v4_topk_transform_512'
```

符号级探测（同一 `common_ops.cpython-310-x86_64-linux-gnu.so`）：

```text
new namespaces after load: []
deepseek_v4_topk_transform_512: MISSING (AttributeError)
fast_topk_transform_fused: PRESENT
```

`new namespaces after load: []` 说明 import 时已注册完毕，排除「扩展未加载」。

torch 回退实测（B=4, S=2048, page=64, topk=512）：

```text
torch fallback OK  pages: (4, 512) raw: (4, 512)
all valid: True   raw in [0,S): True
matches torch.topk: True
```

## 噪声项

以下与本次根因无关，记录以免重复排查：

- `Skipping import of cpp extensions due to incompatible torch version` — 来自
  torchao，非 sgl_kernel。
- `py-spy dump ... Failed to find a python interpreter in the .data section` —
  dump 失败，不影响定位。
- `Tokenizer ... is still TokenizersBackend after retries` — 与崩溃无关。
- `--kv-cache-dtype auto` 但 server_args 显示 `kv_cache_dtype='fp8_e4m3'` —
  DSV4 模式覆写，非配置错误。
- `deepep-config.json` 传参与实际文件名 `deepep-config.josn` 不一致 —— 本次未触发
  （在 indexer 就崩了），但 MoE dispatch 初始化时会 FileNotFoundError。

## 阶段三：升级 wheel 后启动成功，但输出乱码

装上 `sglang-kernel 0.4.4+das.opt1.dtk2604.torch2100.2608132334.g97a193` 后
算子注册、服务 ready，但 `/v1/chat/completions` 返回无语义字符流，
`finish_reason=length`。详见
[dsv4-hcu-garbled-output](../failures/dsv4-hcu-garbled-output.md)（`suspected`）。

版本号陷阱：`sgl_kernel.__version__` 升级前后都返回裸 `0.4.4`，
只有 `pip list` 的 local version 能区分构建。

### 输入侧已排除（原「chat template」假设证伪）

checkpoint 确实无 `chat_template`，`AutoTokenizer` 也确实降级为
`TokenizersBackend`（transformers 不识别 `model_type: deepseek_v4`），
日志因此出现：

```text
Tokenizer for <model> is still TokenizersBackend after retries with --trust-remote-code.
No HuggingFace chat template found
No chat template found, defaulting to 'string' content format
```

但这些是噪声。`resolve_chat_encoding_spec` 按 architecture 命中 `DeepseekV4`
返回 `"dsv4"`，走原生编码器绕开 `apply_chat_template`：

```text
dsv4 n_tokens: 8
ids: [0, 128803, 2788, 70979, 1330, 320, 128804, 128822]
decoded: '<｜begin▁of▁sentence｜><｜User｜>请介绍一下自己。<｜Assistant｜></think>'
```

与服务端 `prompt_tokens=8` 一致 → 输入侧正确。

对照数据（同一 tokenizer）：raw 4 tokens / 正确 DeepSeek 模板 7 tokens /
dsv4 编码器 8 tokens。只有 8 与实测吻合。

### MHC aiter 路径数值对照

`SGLANG_OPT_USE_TILELANG_MHC_PRE=0` 后走 `aiter.ops.mhc.mhc_pre`，
对照 torch 参考实现（HC=4, HID=7168, N=16）：

```text
y      max_abs=0.031250  rel=0.004785  finite=True
post   max_abs=0.000006  rel=0.000003  finite=True
comb   max_abs=0.000002  rel=0.000002  finite=True
```

bf16 量级误差，非乱码来源。

### 环境补充

```text
tilelang 0.1.9+das185...  —— 白名单仍为 {gfx938, gfx92a, gfx946}，不含 gfx936
aiter    0.1.5+das185...
```

## 待验证

乱码根因未定位。`prefill.sh:39` 的 `SGLANG_NSA_FUSE_TOPK=false` 是调试残留
（注释自述 `temp: bypass lightop fused topk to isolate crash`），
为当前最高怀疑项，但尚未做对照实验。

两个 kernel 层 failure 的 `confirmed` 依据是根因排他（白名单直读 +
符号级新旧名对比），不含端到端输出正确性。
