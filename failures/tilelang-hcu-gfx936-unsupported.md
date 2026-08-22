---
id: tilelang-hcu-gfx936-unsupported
kind: failure
layer: kernel
components: [tilelang, mhc, deepseek-v4, hcu]
signatures:
  - 'HCU arch gfx[0-9a-z]+ not supported for MLS/GEMM_MLS'
  - 'tvm\.error\.InternalError.*Check failed.*supported\.count\(mcpu\)'
signature-sources:
  - "external:tilelang:src/target/utils.cc:151-157"
detection: |
  首次 warmup extend forward 时全部 TP rank 同秒退出；traceback 终点在 tilelang
  JIT 编译阶段而非 kernel 执行阶段。日志前一行通常是
  `TileLang begins to compile kernel ...`。
masquerades-as:
  - process-disappeared
  - eoferror-dataparallelcontroller
repro-condition:
  - "GPU arch 不在 {gfx938, gfx92a, gfx946} 内（observed: gfx936）"
  - "SGLANG_OPT_USE_TILELANG_MHC_PRE=true（默认值）"
  - "模型走 DeepSeek-V4 MHC pre 路径"
applies-to:
  sglang-das: "deepseek-v4-opt @ 9ca1b25f7f"
  tilelang: "容器内 dist-packages 版本，含 GetHcuArchString 白名单；未取到版本号"
  gpu: "gfx936 (HCU/DCU BW)"
status: confirmed
first-seen: 2026-08-21
fixed-in:
verify: |
  # PASS: 白名单仍不含当前卡 arch。
  grep -n 'supported = {' <tilelang>/src/target/utils.cc
  # 预期输出 {"gfx938", "gfx92a", "gfx946"}；与 rocminfo 的 gcnArchName 对比。
  # NOT-APPLICABLE: tilelang 已加入该 arch，或模型不走 MHC pre。
  # ENV-MISMATCH: 宿主与实际推理容器 tilelang 版本不同。
sources:
  - "raw/20260821-dsv4-prefill-hcu-gfx936.md"
---

# tilelang GEMM 后端不支持 gfx936，MHC pre 编译期崩溃

## 现象

- 权重加载、量化方法选择、分布式初始化全部通过；
- 第一次 warmup extend forward 时 8 个 TP rank 同秒退出；
- traceback 终点在 tilelang JIT 编译，不是 kernel 执行；
- 报错原文：

```text
tvm.error.InternalError: Check failed: (supported.count(mcpu)) is false:
HCU arch gfx936 not supported for MLS/GEMM_MLS; supported: gfx938, gfx92a, gfx946
```

## 根因

`hc_pre` 默认走 tilelang 路径，而该路径的 stage-0 GEMM 是 tilelang JIT kernel，
在 gfx936 上编译期即被白名单拒绝。

调用链：

- `python/sglang/srt/models/deepseek_v4.py:1636` — `if envs.SGLANG_OPT_USE_TILELANG_MHC_PRE.get():`
- `python/sglang/srt/environ.py:1303` — `SGLANG_OPT_USE_TILELANG_MHC_PRE = EnvBool(True)`，默认开
- → `sglang/kernels/ops/layernorm/mhc.py` 的 `mhc_pre`，stage-0 GEMM 用 `T.gemm`
- → tilelang LayoutInference → `GetHcuArchString`
- → `external:tilelang:src/target/utils.cc:154-157`：

```cpp
static const std::set<std::string> supported = {"gfx938", "gfx92a", "gfx946"};
ICHECK(supported.count(mcpu))
    << "HCU arch " << mcpu
    << " not supported for MLS/GEMM_MLS; supported: gfx938, gfx92a, gfx946";
```

本机 8 卡 `gcnArchName` 均为 `gfx936`，不在白名单内。`ICHECK` 是 hard assert，
无 fallback 分支，所以每个 rank 在各自首次编译该 kernel 时同时失败。

## 定位过程

**排除「HCU 适配开关没开」。** 启动脚本已设 `SGLANG_ROCM_USE_AITER_TILELANG_MHC=1`
但无效。读 `mhc.py` 确认 `mhc_pre` 分两段，该开关只替换 stage-1 big-fuse
（切到 aiter 的 `pre_big_fuse_tilelang`），stage-0 GEMM 恒走 tilelang `T.gemm`。
对比 `mhc_post`：`deepseek_v4.py:1744` 有完整 `_is_hcu and _use_aiter_tilelang_mhc`
分支彻底绕开 tilelang，而 `hc_pre` 没有对应分支。这是 fork 中 HCU 适配只做了一半，
不是配置错误。

**排除 deep_gemm 替代路线。** 另一条 stage-0 路径 `SGLANG_OPT_DEEPGEMM_HC_PRENORM=1`
不可用：容器内 `deep_gemm/_C.so` 是 CUDA 构建，加载即失败

```text
RuntimeError: Failed to load .../deep_gemm/_C.so
libcudart.so.13: cannot open shared object file
```

脚本原本设为 `0`，是正确的，不应改动。

**确认 aiter 路径在同卡可用。** 实测 `aiter.ops.mhc.mhc_pre`（hc_mult=4,
hidden=7168）返回 `(8,4,1) (8,4,4) (8,7168) bf16`，`finite: True`，形状与
`hc_pre` 期望的 `post.squeeze(-1)` 一致。

## 修复 / 规避

```bash
export SGLANG_OPT_USE_TILELANG_MHC_PRE=0
```

`hc_pre` 落到下一分支 `deepseek_v4.py:1665` 的
`_is_hip and envs.SGLANG_OPT_USE_AITER_MHC_PRE.get()`（`environ.py:1281` 默认 True），
使用 aiter 预编译算子，不触发 tilelang JIT。

语义等价：两条路径在 HCU 上 `norm_fused` 均为 `False`，`input_layernorm` 仍由调用方
施加，不会漏做或重复做 norm。

prefill 与 decode 使用同一组 MHC 开关，需同时设置。

正式修复方向：为 `hc_pre` 补 `_is_hcu` 分支（对齐 `mhc_post` 的处理），或在 tilelang
白名单中加入 gfx936 并验证 MMAC 正确性。均不在本次记录范围。

### ⚠️ 该 workaround 仅适用于挂载旧 commit，不适用于 wheel 版 sglang

**wheel 版 sglang（0.5.15.post2+das.opt1...g97a193 及之后）里 `deepseek_v4.py`
在 `SGLANG_OPT_USE_TILELANG_MHC_PRE.get()` 内部有 `_is_hcu and _use_aiter_tilelang_mhc`
兜底**，走 `aiter.ops.tilelang.pre_big_fuse_tilelang`；此路径不触发 MLS/GEMM_MLS
编译，gfx936 上稳定。因此 wheel 版**必须保持 `SGLANG_OPT_USE_TILELANG_MHC_PRE=True`（默认）**，
不能 export 关掉；关掉会造成 [dsv4-hcu-garbled-output](dsv4-hcu-garbled-output.md)
所述的静默乱码。

**挂载源** `deepseek_v4.py:1636` 及 `mhc.py:960 mhc_pre` 目前也有相同兜底代码
（`_is_hcu and _use_aiter_tilelang_mhc`），因此 wheel 与挂载源在 MHC pre 上应
一致；本页原 workaround 是老 commit 上写的，需在当前挂载源上确认。

若挂载源上确实必须 `TILELANG_MHC_PRE=0`（例如更旧的 commit），**同时不能关
`SGLANG_OPT_USE_AITER_INDEXER`**——挂载源的
`sglang/srt/layers/attention/dsv4/indexer.py` 缺少 wheel 版
`indexer.py:730-742` 处的 HCU lightop 兜底分支（`elif _is_hcu → lightop.attention.paged_mqa_logits`），
一旦把 aiter 分支也关掉，indexer 会落到 `from deep_gemm import fp8_paged_mqa_logits`，
其内部同样拉起 tilelang MLS codegen，再次撞 `GetHcuArchString` 白名单，
scheduler 死亡：

```text
InternalError: Check failed: (supported.count(mcpu)) is false
HCU arch gfx936 not supported for MLS/GEMM_MLS
```

因此挂载源上的**安全组合**是：让 `SGLANG_OPT_USE_AITER_INDEXER` 保持默认（HCU 上
`server_args.py` 会自动 `set(True)`），只调整 MHC pre。

## 复发判据

- 日志出现 `not supported for MLS/GEMM_MLS`，且 arch 与 rocminfo 一致；
- 崩溃点在 tilelang 编译而非执行；
- 全部 rank 同秒失败，无单 rank 先行；
- 关闭 `SGLANG_OPT_USE_TILELANG_MHC_PRE` 后该点不再复现。
