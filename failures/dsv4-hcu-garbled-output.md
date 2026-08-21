---
id: dsv4-hcu-garbled-output
kind: failure
layer: accuracy
components: [deepseek-v4, hcu, lightop, aiter, dsa-indexer]
signatures: []
signature-sources: []
detection: |
  服务正常启动并返回 HTTP 200，但 /v1/chat/completions 的 content 是无语义字符流
  （长串 `_` / `|` / 空格构成的 ASCII 框线，夹杂零散 token），
  finish_reason=length 而非 stop。日志无 traceback、无 NaN 告警。
  这是纯静默精度故障：只能由输出内容判定，不能靠日志 signature 匹配。
masquerades-as:
  - silent-performance-regression
repro-condition:
  - "observed: gfx936 + DeepSeek-V4-Flash INT8 w8a8 + tp8/ep8/attn_cp8 + dsv4 attention backend"
  - "observed: SGLANG_OPT_USE_TILELANG_MHC_PRE=0（gfx936 上必须，见 tilelang-hcu-gfx936-unsupported）"
  - "observed: SGLANG_NSA_FUSE_TOPK=false（人工调试残留，非默认值）"
  - "temperature=0 下稳定复现；未做 TP=1 / 关融合的对照实验"
applies-to:
  sglang-das: "deepseek-v4-opt @ 9ca1b25f7f"
  sgl-kernel: "0.4.4+das.opt1.dtk2604.torch2100.2608132334.g97a193"
  tilelang: "0.1.9+das185.dtk2604.torch2100.2608121523.gf631c9"
  aiter: "0.1.5+das185.dtk2604.torch2100.2608151919.g672e2d"
  gpu: "gfx936 (HCU/DCU BW) x8"
status: suspected
first-seen: 2026-08-21
fixed-in:
verify: |
  # 判定当前是否仍复现（需服务在跑）：
  curl -s http://127.0.0.1:30001/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{"model":"m","messages":[{"role":"user","content":"请介绍一下自己。"}],
         "temperature":0,"max_tokens":128}' | python3 -m json.tool
  # FAIL(仍复现): content 为无语义字符流，finish_reason=length。
  # PASS: content 为通顺自然语言。
  # NOT-APPLICABLE: 非 gfx936 或非 DeepSeek-V4。
  # ENV-MISMATCH: sgl-kernel / tilelang / aiter 版本与 applies-to 不同。
sources:
  - "raw/20260821-dsv4-prefill-hcu-gfx936.md"
---

# DeepSeek-V4 在 gfx936 上启动正常但输出乱码

## 现象

服务完整启动（`The server is fired up and ready to roll!`），请求返回 200，
但 `content` 无语义：

```text
#  _____________________________________________________________________
# /                                                                     \|
... 长串框线字符 ...  a_n+! ., in
```

- `finish_reason=length`（撞满 `max_tokens=128`，模型从未生成 EOS）；
- `temperature=0` 下稳定复现；
- 日志无 traceback、无 NaN/inf 告警、无 kernel 报错；
- 8 个 rank 的 decode 吞吐正常（~4.4 token/s），无掉队 rank。

**未定位到根因，故为 `suspected`。** 本页记录已排除项，避免重复排查。

## 根因

未确定。已排除输入侧与两处 kernel 替换（见定位过程），
剩余怀疑集中在 gfx936 上的融合算子精度，尚未做对照实验。

## 定位过程

**排除 prompt 编码 / chat template。** 这是最初的怀疑方向，已证伪。
checkpoint 无 `chat_template`（`tokenizer_config.json` 无该键，
无 `chat_template.jinja`），且 `AutoTokenizer` 因 transformers 不识别
`model_type: deepseek_v4` 而降级为 `TokenizersBackend`，日志因此出现
`No HuggingFace chat template found` / `No chat template found, defaulting to
'string' content format`。但这些都是**噪声**：
`resolve_chat_encoding_spec`（`entrypoints/openai/chat_encoding.py:128`）
按 architecture 命中 `DeepseekV4` 返回 `"dsv4"`，走原生编码器，
完全绕开 `apply_chat_template`。实测该编码器输出：

```text
dsv4 n_tokens: 8
ids: [0, 128803, 2788, 70979, 1330, 320, 128804, 128822]
decoded: '<｜begin▁of▁sentence｜><｜User｜>请介绍一下自己。<｜Assistant｜></think>'
```

8 tokens 与服务端 `prompt_tokens=8` 完全一致，特殊 token 齐全。
**输入侧正确，不是模板问题。**

**排除 MHC pre 的 aiter 替换。** gfx936 上必须设
`SGLANG_OPT_USE_TILELANG_MHC_PRE=0`（tilelang 白名单不含 gfx936，
见 [tilelang-hcu-gfx936-unsupported](tilelang-hcu-gfx936-unsupported.md)），
这会把 MHC pre 从 tilelang 切到 `aiter.ops.mhc.mhc_pre`。
对照 `deepseek_v4.py` 的 torch 参考实现（`hc_pre_torch_impl` +
`hc_split_sinkhorn`）实测（HC=4, HID=7168, N=16）：

```text
y      max_abs=0.031250  rel=0.004785  finite=True
post   max_abs=0.000006  rel=0.000003  finite=True
comb   max_abs=0.000002  rel=0.000002  finite=True
```

`y` 的误差是 bf16 量级，`post`/`comb` 近似精确。**该替换不是乱码来源。**

**排除 sgl_kernel 版本错配。** 升级 wheel 后
`deepseek_v4_topk_transform_512` 已注册（见
[sgl-kernel-version-op-missing](sgl-kernel-version-op-missing.md)），
不再走 torch 回退。

**排除量化方案选错。** MoE 实际方法为
`CompressedTensorsW8A8Int8MarlinMoEMethod`，与 INT8 w8a8 checkpoint 一致；
`[slimquant_w4a8_marlin]` 横幅是 import-time 噪声，见
[slimquant-w4a8-logger-misleading](../symptoms/slimquant-w4a8-logger-misleading.md)。

**排除 unfused topk 路径缺少 page-table 变换。** `SGLANG_NSA_FUSE_TOPK=false`
使 `topk_transform`（`dsa/dsa_topk_backend.py:94-95`）直接返回
`_topk_unfused` 的**局部**索引，不做 page-table 变换。核对调用方确认
`dsa_backend.py:2275` / `:3206` 的 `else` 分支会调用
`transform_index_page_table_decode/prefill` 补上该变换，语义闭合。
**静态看无缺陷**，但该路径在 HCU 上未做数值对照。

## 修复 / 规避

尚无确认修复。按怀疑度排序的下一步（每次只改一项，其余不变）：

1. **恢复 `SGLANG_NSA_FUSE_TOPK` 默认值**：`prefill.sh:39` 的
   `export SGLANG_NSA_FUSE_TOPK=false` 注释写明是
   `temp: bypass lightop fused topk to isolate crash`，
   属调试残留；该 env 已被 `SGLANG_DSA_FUSE_TOPK` 取代（默认 `True`，
   `environ.py:157` 保留 deprecated alias）。删掉这行恢复默认融合路径，
   是成本最低且最可疑的一项。
2. **逐个关闭 gfx936 上的融合算子**，观察输出是否恢复：
   `SGLANG_OPT_USE_FUSED_HASH_TOPK` / `SGLANG_OPT_USE_JIT_KERNEL_FUSED_TOPK`
   / `SGLANG_USE_FUSED_DPSKV4_QNORM_ROPE_KV_ROPE_QUANT` /
   `SGLANG_ROCM_USE_AITER_MOE` / `SGLANG_GROUPGEMM` / `SGLANG_USE_LIGHTOP`。
3. **降并行维度**：TP=1 单卡跑同一 prompt。若 TP=1 正确而 TP=8 乱码，
   按 [quant-scale-tp-split](quant-scale-tp-split.md) 查 scale 切分。
4. **关 CP**：去掉 `--enable-nsa-prefill-context-parallel`，
   排除 `attn_cp=8` 的 round-robin-split 路径。

定位方法见 [accuracy playbook](../playbooks/accuracy.md)：先固定可比条件，
用二分矩阵定位首个偏离层，不要从最终乱码反推具体 kernel。

## 复发判据

- 服务 ready、无 traceback，但 content 无语义且 `finish_reason=length`；
- 输入侧已验证正确（`prompt_tokens` 与 dsv4 编码器一致）；
- gfx936 + DeepSeek-V4 + 上述融合算子组合。
