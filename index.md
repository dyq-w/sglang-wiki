# 知识库索引

> 本索引只包含诊断知识。`skills/` 是随 Git 分发的工具流程，问题分析、signature 匹配和全文检索时必须忽略，也不会加入 `index.tsv`。

## 诊断入口

1. [证据采集](playbooks/evidence-collection.md)
2. [定位最早异常](playbooks/find-first-error.md)
3. 按阶段进入对应 playbook
4. 用 [index.tsv](index.tsv) 匹配症状或根因
5. 执行最小判别实验后再给 `confirmed / suspected`

## Playbooks

| 阶段 | 页面 | 典型边界 |
|---|---|---|
| launch | [launch-config.md](playbooks/launch-config.md) | 权重尚未开始加载 |
| weights-quant | [weight-quant.md](playbooks/weight-quant.md) | key / shape / scale / layout |
| kernel | [kernel.md](playbooks/kernel.md) | 首次 forward、HIP graph、算子调用 |
| distributed | [distributed.md](playbooks/distributed.md) | hang、collective、仅多卡复现 |
| pd | [pd-disagg.md](playbooks/pd-disagg.md) | Mooncake、RDMA、prefill/decode 边界 |
| memory | [memory.md](playbooks/memory.md) | HIP OOM、运行一段时间后退出 |
| accuracy | [accuracy.md](playbooks/accuracy.md) | 乱码、NaN、数值漂移 |
| perf | [perf.md](playbooks/perf.md) | 无报错但性能下降、静默回退 |
| LightOp | [lightop-isolation.md](playbooks/lightop-isolation.md) | 脱离 SGLang 验证算子 |

## Symptoms

| ID | 诊断价值 | 含义 |
|---|---|---|
| [eoferror-dataparallelcontroller](symptoms/eoferror-dataparallelcontroller.md) | none | 子进程已死，父进程读管道失败 |
| [hip-out-of-memory](symptoms/hip-out-of-memory.md) | strong | 显存分配失败，但仍需定位发生阶段和首个 rank |
| [process-disappeared](symptoms/process-disappeared.md) | weak | 进程消失；可能是 signal、abort、OOM 或 LightOp `exit(0)` |
| [silent-performance-regression](symptoms/silent-performance-regression.md) | weak | 服务可用但算子配置或设备映射可能静默回退 |
| [slimquant-w4a8-logger-misleading](symptoms/slimquant-w4a8-logger-misleading.md) | none | import-time 横幅，不代表 w4a8 生效；改看 MoE method 类名 |

## Failures

| ID | Layer | 状态 | 一句话结论 |
|---|---|---|---|
| [deepep-rocshmem-ipc-invalid-pointer](failures/deepep-rocshmem-ipc-invalid-pointer.md) | distributed | confirmed | RocSHMEM 打开 HIP IPC handle 失败，后续 EOFError 只是噪声 |
| [dspark-moe-runtime-hip-oom](failures/dspark-moe-runtime-hip-oom.md) | memory | suspected | DSpark + INT8 MoE 运行时临时 tensor 分配耗尽显存，多 rank 同秒失败 |
| [lightop-hip-call-exit-zero](failures/lightop-hip-call-exit-zero.md) | kernel | confirmed | HIP API 失败后宏调用 `exit(0)`，导致无 traceback 且返回码错误 |
| [lightop-asm-dir-null](failures/lightop-asm-dir-null.md) | kernel | confirmed | C++ 直接用未设置的 `LIGHTOP_ASM_DIR` 构造 string，可能段错误 |
| [lightop-unsupported-device-relabel](failures/lightop-unsupported-device-relabel.md) | perf | confirmed | 未支持的 GPU/CU 组合被静默映射为 `gfx936_80cu` |
| [lightop-config-nearest-fallback](failures/lightop-config-nearest-fallback.md) | perf | confirmed | tuned config 未精确命中时静默选择最近 token 配置，并被缓存 |
| [lightop-w8a8-smooth-none](failures/lightop-w8a8-smooth-none.md) | kernel | confirmed | 多个条件下返回 `(False, None)`，调用方必须检查 status |
| [lightop-test-build-dir-mismatch](failures/lightop-test-build-dir-mismatch.md) | launch | confirmed | 测试 runner 只找 `build/lib.*`，当前 checkout 只有 `build/lib` 且无 `op*.so` |
| [quant-fused-shards-mixed-precision](failures/quant-fused-shards-mixed-precision.md) | weights-quant | confirmed | 融合层各 shard 精度不一致时主动报错 |
| [quant-scale-tp-split](failures/quant-scale-tp-split.md) | accuracy | suspected | per-channel scale 应按 output dim 切，per-tensor scale 不按 TP 切 |
| [tilelang-hcu-gfx936-unsupported](failures/tilelang-hcu-gfx936-unsupported.md) | kernel | confirmed | tilelang GEMM 白名单不含 gfx936，MHC pre 编译期 hard assert |
| [sgl-kernel-version-op-missing](failures/sgl-kernel-version-op-missing.md) | kernel | confirmed | wheel 落后于挂载源码，算子被重命名导致 `_OpNamespace` AttributeError |
| [dsv4-hcu-garbled-output](failures/dsv4-hcu-garbled-output.md) | accuracy | suspected | gfx936 上启动正常但输出无语义；已排除 prompt 编码与 MHC aiter 路径 |

## Components

- [LightOp 代码基线](components/lightop.md)
- [量化与并行切分基线](components/quantization.md)

## Evidence

- [2026-08-15 DeepEP / RocSHMEM IPC 证据](raw/20260815-deepep-ipc-invalid-pointer.md)
- [2026-08-14 DSpark HIP OOM 证据](raw/20260814-dspark-hip-oom.md)
- [2026-08-20 技术基线调查](baseline/2026-08-20-bootstrap.md)
- [2026-08-21 DeepSeek-V4 prefill on gfx936 证据](raw/20260821-dsv4-prefill-hcu-gfx936.md)
