---
id: sgl-kernel-version-op-missing
kind: failure
layer: kernel
components: [sgl-kernel, dsv4-indexer, deepseek-v4]
signatures:
  - "'_OpNamespace' 'sgl_kernel' object has no attribute"
  - 'torch\.ops\.sgl_kernel\.[a-z0-9_]+.*AttributeError'
signature-sources:
  - "python/sglang/kernels/ops/attention/dsv4/topk.py:57"
detection: |
  挂载源码覆盖 site-packages 时，Python 层调用了当前 wheel 未注册的算子名。
  同一 .so 内旧名算子存在、新名缺失即可确诊，与「扩展未加载」区分。
masquerades-as:
  - process-disappeared
  - eoferror-dataparallelcontroller
repro-condition:
  - "python/ 源码挂载覆盖 site-packages，但 sgl_kernel wheel 未同步"
  - "observed: sgl_kernel 0.4.4 + sglang-das @ 9ca1b25f7f 调用 deepseek_v4_topk_transform_512"
applies-to:
  sgl-kernel: "0.4.4 (dist-packages)"
  sglang-das: "deepseek-v4-opt @ 9ca1b25f7f"
status: confirmed
first-seen: 2026-08-21
fixed-in: "sglang-kernel 0.4.4+das.opt1.dtk2604.torch2100.2608132334.g97a193"
verify: |
  # PASS: 新名缺失而旧名存在 → 版本错配。
  python3 -c "
  import torch, sgl_kernel
  for n in ('deepseek_v4_topk_transform_512','fast_topk_transform_fused'):
      try: getattr(torch.ops.sgl_kernel, n).default; print(n,'PRESENT')
      except AttributeError: print(n,'MISSING')"
  # 注意 dir(torch.ops.sgl_kernel) 是延迟枚举，不反映已注册算子，不能用于计数或判空。
  # 版本务必用 pip list 看 local version；sgl_kernel.__version__ 修复前后都是裸 0.4.4。
  # NOT-APPLICABLE: wheel 与源码版本已对齐。
  # ENV-MISMATCH: 宿主 pip 环境与推理容器不同。
sources:
  - "raw/20260821-dsv4-prefill-hcu-gfx936.md"
---

# sgl_kernel wheel 落后于挂载源码，算子名缺失

## 现象

- MHC pre 问题修复后前向继续推进，随即在 DSA indexer 崩溃；
- 8 个 rank 同秒退出，报错一致：

```text
AttributeError: '_OpNamespace' 'sgl_kernel' object has no attribute
              'deepseek_v4_topk_transform_512'
```

- `py-spy` 无法 dump（`Failed to find a python interpreter in the .data section`），
  是噪声，不影响定位。

## 根因

容器内 `sgl_kernel` 是 0.4.4，挂载的 `sglang-das` 源码调用了该 wheel 未注册的算子名。
算子在上游被重命名/重构，Python 层已更新而 C++ 扩展未同步。

调用链：

- `python/sglang/srt/models/deepseek_v4.py:1349` — `_forward_prepare`
- `python/sglang/srt/layers/attention/dsv4/indexer.py:1002` — `forward_c4_indexer`
- `python/sglang/srt/layers/attention/dsv4/indexer.py:879` — `topk_transform_512(...)`
- `python/sglang/kernels/ops/attention/dsv4/topk.py:57` —
  `torch.ops.sgl_kernel.deepseek_v4_topk_transform_512(...)`

符号级探测结果（同一 `common_ops.cpython-310-x86_64-linux-gnu.so`）：

| 算子 | 状态 |
|---|---|
| `fast_topk_transform_fused` | PRESENT（旧 API） |
| `deepseek_v4_topk_transform_512` | MISSING（新 API） |

旧名在、新名缺，是版本错配的判据；若全部缺失则应怀疑扩展未加载，修复方向不同。

## 定位过程

**排除「扩展根本没加载」。** `.so` 可被 `torch.ops.load_library` 成功加载，HIP 6.3
匹配，且加载后 `torch.ops` 无新增命名空间（说明 import 时已注册完毕）。

**排除误判陷阱。** `dir(torch.ops.sgl_kernel)` 是延迟枚举，看不到已注册算子。本次
定位中曾据此误判「只注册了 1 个算子、基本是空壳」，随后由 `getattr(...).default`
逐名探测纠正。该方法缺陷已写入 `skills/sglang-triage/SKILL.md`。

**排除 torchao 干扰。** 日志中
`Skipping import of cpp extensions due to incompatible torch version`
来自 torchao，与 `sgl_kernel` 无关。

## 修复 / 规避

```bash
export SGLANG_TOPK_TRANSFORM_512_TORCH=true
```

`indexer.py:845` 该 env 是第一优先分支，优先级高于 `dsa_topk_backend`，无需改
`--dsa-topk-backend`。前置条件 `SGLANG_NSA_FUSE_TOPK=false` 需满足（本次已满足）。
落到 `indexer.py:361` 的 `topk_transform_512_pytorch_vectorized`。

实测（B=4, S=2048, page=64, topk=512）：

```text
torch fallback OK  pages: (4, 512) raw: (4, 512)
all valid: True   raw in [0,S): True
matches torch.topk: True
```

与 `torch.topk` 逐元素一致，无数值折损。代价是性能：走 vectorized PyTorch 而非融合
kernel。该实现有 helper tensor 缓存以兼容 graph capture；本次 `disable_cuda_graph=True`，
不涉及。

同路径第二个 `sgl_kernel` 依赖 `dsv4_fused_q_indexer_rope_hadamard_quant`
（`python/sglang/kernels/ops/attention/dsv4/elementwise.py:195`）位于 `_is_hip` 分支内，
HCU 走 `_is_hcu` 的 TVM-FFI JIT 分支，不会撞上。

根治：重装与 `sglang-das` 匹配的 `sgl_kernel`，或从 `python/sglang/kernels/` 就地编译
AOT 扩展。env 开关是当下可跑通的规避。

## 根治已验证（2026-08-21）

装上更新的 wheel 后算子注册，服务正常启动：

```text
sglang-kernel  0.4.4+das.opt1.dtk2604.torch2100.2608132334.g97a193
deepseek_v4_topk_transform_512  REGISTERED
fast_topk_transform_fused       REGISTERED
```

**版本号陷阱：`sgl_kernel.__version__` 修复前后都返回裸 `0.4.4`**，不带 `+das.opt1...`
后缀。只有 `pip list` / dist-info 的 local version 能区分两个构建。判断 wheel 是否匹配
必须查 `pip list`，或直接探测算子名，不能查 `__version__`。

启动通过不等于结果正确：本次修复后端到端输出为乱码，见
[dsv4-hcu-garbled-output](dsv4-hcu-garbled-output.md)。

## 复发判据

- `_OpNamespace 'sgl_kernel' object has no attribute <op>`；
- 同 `.so` 内存在同族旧名算子；
- 源码挂载覆盖 site-packages 且两者版本不一致；
- 切到 torch 后端后该点不再复现。
