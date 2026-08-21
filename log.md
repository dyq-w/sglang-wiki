# 知识库时间线

## [2026-08-20] bootstrap | 初始化 SGLang-DAS 故障诊断知识库

- 建立 symptom / failure 多对多模型
- 建立按阶段组织的 playbooks
- 从两份真实日志沉淀 DeepEP IPC 和 DSpark HIP OOM 证据
- 从当前 SGLang / LightOp checkout 提取首批真实 signature 与静默失败机制
- 建立 `index.md` + `index.tsv` 双索引
- 建立 `sglang-triage` 与 `sglang-kb-record` 双 Skill

## [2026-08-20] workflow | 将双 Skill 纳入 Wiki 分发并支持持续改进

- 新增 `skills/` 目录，保存两个 Skill 的 Git 规范副本和安装说明
- 规定诊断时忽略 `skills/`，避免工具流程参与知识匹配
- `sglang-kb-record` 在每次沉淀后复盘 Skill；有具体证据时可在同一 diff 和 commit 中完善流程
- commit 后安全同步用户级安装副本；不覆盖本地定制，不自动 push 或创建 PR

## [2026-08-21] record | DeepSeek-V4 prefill 在 gfx936 上的两处 kernel 层失败

- 新增 `tilelang-hcu-gfx936-unsupported`：tilelang GEMM 白名单
  `{gfx938, gfx92a, gfx946}` 不含 gfx936，`hc_pre` 默认走 tilelang 导致编译期
  hard assert；`SGLANG_ROCM_USE_AITER_TILELANG_MHC` 只覆盖 stage-1，无法规避
- 新增 `sgl-kernel-version-op-missing`：sgl_kernel 0.4.4 未注册
  `deepseek_v4_topk_transform_512`，同 `.so` 内旧名 `fast_topk_transform_fused`
  存在，据此区分版本错配与扩展未加载
- 新增 symptom `slimquant-w4a8-logger-misleading`：import-time 横幅不代表 w4a8 生效
- 记录 `raw/20260821-dsv4-prefill-hcu-gfx936.md`，含噪声项清单
- Skill 改进：`sglang-triage` 6.1 明确 `dir(torch.ops.<ns>)` 延迟枚举不可用于判空，
  须 `getattr(...).default` 逐名探测；本次定位曾因此误判「扩展是空壳」

## [2026-08-21] record | 升级 sgl-kernel 后启动成功，但输出乱码

- `sgl-kernel-version-op-missing` 标记 `fixed-in`：
  `0.4.4+das.opt1.dtk2604.torch2100.2608132334.g97a193` 已注册
  `deepseek_v4_topk_transform_512`；补记版本号陷阱 ——
  `sgl_kernel.__version__` 升级前后都是裸 `0.4.4`，须查 `pip list` local version
- 新增 `dsv4-hcu-garbled-output`（`suspected`）：启动正常但 content 无语义，
  `finish_reason=length`，无 traceback
- 证伪原「chat template 缺失」假设：DeepSeek-V4 由
  `resolve_chat_encoding_spec` 命中 architecture 走 dsv4 原生编码器，
  绕开 `apply_chat_template`；实测 8 tokens 与服务端 `prompt_tokens` 一致，
  `TokenizersBackend` / `No chat template found` 均为噪声，已加入 index.tsv
- 排除 MHC aiter 替换：对照 torch 参考实现 `y` 仅 bf16 量级误差，
  `post`/`comb` 近似精确
- 记录最高怀疑项：`prefill.sh:39` 的 `SGLANG_NSA_FUSE_TOPK=false` 是调试残留，
  该 env 已被默认 `True` 的 `SGLANG_DSA_FUSE_TOPK` 取代
