---
id: dsv4-hcu-garbled-output
kind: failure
layer: accuracy
components: [deepseek-v4, hcu, aiter, mhc]
signatures: []
signature-sources: []
detection: |
  服务正常启动并返回 HTTP 200，但 /v1/chat/completions 及 /v1/completions 和
  /generate 全部端点 content 为无语义字符流（长串 `_` / `|` / 空格构成的 ASCII
  框线，夹杂零散 token 或重复片段 `Hom nay Hom nay` / `{/cableimplication` /
  `1.1.1.1.1.`），finish_reason=length 而非 stop。日志无 traceback、无 NaN 告警。
  纯静默精度故障，只能由输出内容判定。
masquerades-as:
  - silent-performance-regression
repro-condition:
  - "observed: gfx936 + DeepSeek-V4-Flash INT8 w8a8 + tp8/ep8/attn_cp8 + dsv4 attention backend"
  - "触发条件：export SGLANG_OPT_USE_TILELANG_MHC_PRE=0（人工设置，覆盖了 environ 默认 True）"
  - "SGLANG_ROCM_USE_AITER_TILELANG_MHC=1 时该 export 会跳过唯一验证过的 MHC pre kernel"
  - "temperature=0 稳定复现；反证：单独 export SGLANG_OPT_USE_AITER_INDEXER=false 不复现"
applies-to:
  sglang: "0.5.15.post2+das.opt1.dtk2604.torch2100.2608132257.g97a193 (wheel)"
  sgl-kernel: "0.4.4+das.opt1.dtk2604.torch2100.2608132334.g97a193 或 0.4.6+das.opt1..."
  tilelang: "0.1.9+das185.dtk2604.torch2100.2608121523.gf631c9"
  aiter: "0.1.5+das185.dtk2604.torch2100.2608151919.g672e2d"
  gpu: "gfx936 (HCU/DCU BW) x8"
status: confirmed
first-seen: 2026-08-21
fixed-in:
verify: |
  # PASS: 删掉 prefill.sh 中的 `export SGLANG_OPT_USE_TILELANG_MHC_PRE=0` 后复测：
  curl -s http://127.0.0.1:30001/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{"model":"m","messages":[{"role":"user","content":"请介绍一下自己。"}],
         "temperature":0,"max_tokens":128}' | python3 -m json.tool
  # 预期 content 为通顺自然语言，finish_reason 可为 length 或 stop。
  # FAIL(仍复现): content 仍为无语义框线字符 → 尚有其他 env 覆盖了 HCU 默认路径。
  # NOT-APPLICABLE: 非 gfx936 或非 DeepSeek-V4。
  # ENV-MISMATCH: sglang / sgl-kernel / aiter 版本与 applies-to 不同。
sources:
  - "raw/20260821-dsv4-prefill-hcu-gfx936.md"
---

# DeepSeek-V4 在 gfx936 上因 export 关闭 TILELANG_MHC_PRE 输出乱码

## 现象

服务完整启动，请求返回 200，但 `content` 无语义（`/v1/chat/completions`、
`/v1/completions`、`/generate` 全端点复现）：

```text
#  _________________________________________________________________
# |\/
# |\/
# |\/|/|  |\/|/| |  |/| |/| |/| ...
```

- `finish_reason=length`（撞满 `max_tokens`，模型从未生成 EOS）；
- `temperature=0` 稳定复现；
- 日志无 traceback、无 NaN/inf 告警；
- 8 rank 吞吐正常。

## 根因

`prefill.sh` 中 `export SGLANG_OPT_USE_TILELANG_MHC_PRE=0` 关闭了 gfx936 上
**唯一验证过的 MHC pre 路径**。

`environ.py:1303` 默认 `True`；`server_args.py:4529-4548` 中 HCU 分支
（`elif is_hip(): if not is_hcu(): ...`）**不会**关它，即默认保持 True。
但 `EnvBool` 让外部 env 优先，用户 export=0 直接把 HCU 默认路径推翻。

`deepseek_v4.py:1456` 的 MHC pre 分岔（wheel 版）按顺序：

```python
if envs.SGLANG_OPT_USE_TILELANG_MHC_PRE.get():
    if _is_hcu and _use_aiter_tilelang_mhc:            # ← 默认命中，正确路径
        post, comb, y = mhc_pre_big_fuse(...)          # aiter.ops.tilelang.pre_big_fuse_tilelang
    else:
        from sglang.kernels.ops.layernorm.mhc import mhc_pre
        post, comb, y = mhc_pre(...)
    return ...
if _is_hip and envs.SGLANG_OPT_USE_AITER_MHC_PRE.get():
    from aiter.ops.mhc import mhc_pre
    ...
if envs.SGLANG_OPT_DEEPGEMM_HC_PRENORM.get():
    ...
# fallthrough: torch 手写 hc_pre_torch_impl
```

`_use_aiter_tilelang_mhc = get_bool_env_var("SGLANG_ROCM_USE_AITER_TILELANG_MHC")`
在本次环境中为 `True`（prefill.sh 已 export=1）。

用户 export=0 之后：
- 第一个 `if` 被跳过；
- `SGLANG_OPT_USE_AITER_MHC_PRE` 未 export，走 `environ` 默认值——在**当前 wheel**
  里 `deepseek_v4.py:1456-1500` 附近**没有等价的 `_use_aiter_tilelang_mhc` 兜底**，
  该分支要求 `_is_hip and envs.SGLANG_OPT_USE_AITER_MHC_PRE.get()`；
- `SGLANG_OPT_DEEPGEMM_HC_PRENORM=0`（用户 prefill.sh 已 export=0，deepgemm _C.so
  也不可用）；
- **落到最后的 torch fallback**。该 fallback 在 DSv4 + INT8 w8a8 + attn_cp=8 组合
  上从未被验证，数值精度不足以维持 attention，输出结构性乱码。

## 定位过程

**反证实验**（wheel 版 sglang，只改 prefill.sh 的 env，其余保持一致）：

| 变体 | `SGLANG_OPT_USE_TILELANG_MHC_PRE` | `SGLANG_OPT_USE_AITER_INDEXER` | 输出 |
|---|---|---|---|
| A（两者 export） | `0` | `false` | 乱码 `Hom nay Hom nay` / `{/cableimplication` |
| B（只 export MHC_PRE） | `0` | (不 export) | 乱码 `|\/|/|` / `1.1.1.1.1.` |
| C（只 export AITER_INDEXER） | (不 export) | `false` | **正确**：自然中文自我介绍 |
| D（都不 export） | (不 export) | (不 export) | 正确 |

- B/C 对比锁定：**MHC_PRE 是主因，AITER_INDEXER 与正确性无关**（HCU indexer 有多条
  可用路径，切换不影响 attention 数值）。
- 排除挂载源问题：本次全程使用 wheel `0.5.15.post2+das.opt1...g97a193`，无 editable pth。
- 排除 sgl_kernel 版本：0.4.4 / 0.4.6 均能在正确 env 下产出通顺输出；乱码与 kernel
  版本无关。0.4.6 曾观察到 `lightop int8_utils.matmul_int8` VMFault，是同一根因
  引发下游 kernel 越界（错路径的 tensor 视图不满足 kernel 前置条件），修复主因后
  该崩溃亦消失。
- 排除 chat template：`/v1/completions` 与 `/generate` 同样乱码，且 `resolve_chat_encoding_spec`
  已按 architecture 命中 dsv4 原生编码器。见
  [dsv4 编码器对照数据](../raw/20260821-dsv4-prefill-hcu-gfx936.md#阶段三-升级-wheel-后启动成功但输出乱码)。

之前记录的怀疑项已排除：`SGLANG_NSA_FUSE_TOPK=false` 并非根因，本次修复未变动
该 env；重新验证 MHC 数值差异（bf16 量级）也证实 aiter tilelang 路径本身无问题。

## 修复 / 规避

删除 `prefill.sh` 中的两行：

```bash
export SGLANG_OPT_USE_TILELANG_MHC_PRE=0
export SGLANG_OPT_USE_AITER_INDEXER=false
```

（`AITER_INDEXER` 那行不是主因但也无必要，保留会掩盖真正的 HCU 默认路径。）

原则：**凡是 `environ.py` 里为 `EnvBool` 的 flag，不知道正确值就不要 export。**
`server_args.py` 里根据 HCU / SM120 / is_hip 做的自动 override，只要外部 env 一
export 就被压过；一次覆盖会引发全链路重定向到未验证路径。

## 复发判据

- 服务 ready、无 traceback，`/v1/chat/completions`、`/v1/completions`、
  `/generate` 全端点乱码，`finish_reason=length`；
- gfx936 + DeepSeek-V4 + `SGLANG_ROCM_USE_AITER_TILELANG_MHC=1`；
- 环境中存在 `SGLANG_OPT_USE_TILELANG_MHC_PRE=0` 或等价把该 EnvBool 覆盖为
  false 的写法（含 `.env`、shell rc、Docker `-e`）；
- 删除该 export 后单条 curl 即恢复。
