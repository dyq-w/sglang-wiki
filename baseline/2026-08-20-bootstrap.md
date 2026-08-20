# 2026-08-20 技术基线调查

## 调查范围

- SGLang：`/public/home/duanyongqiang/sglang`
- LightOp：`/public/home/duanyongqiang/oplib/lightop`
- 真实日志：
  - `/public/home/duanyongqiang/workspace/eplb_20260815_181502.log`
  - `/public/home/duanyongqiang/workspace/ifb_decode_dspark_20260814_131243.log`

## Checkout 状态

| 项 | 状态 |
|---|---|
| SGLang 分支 | `dspark_0.515_test` |
| SGLang commit | `bd1f2823e5743437f4e1773d9b0ea5a167de30ca` |
| SGLang 工作树 | **dirty**：量化、MoE、Scheduler、DSpark 等文件存在用户未提交修改；调查只读，未改动这些文件 |
| LightOp 分支 | `master` |
| LightOp commit | `3797bcc4c97c1b08bd4d564e1be81e150e8d9b51` |
| LightOp 工作树 | clean |
| 当前宿主 DTK 文件 | `/opt/dtk/.info/rocm_version = 25.04.4` |
| 当前宿主 Python | 未安装 sglang / sglang-kernel / lightop / torch；运行日志来自另一运行容器，版本验证在该容器中执行 |

当前宿主与目标测试容器存在环境差异，因此任何需要导入 torch/lightop 的验证在本机应返回 `ENV-MISMATCH`，不能据此把知识条目标为失效。

## 1. LightOp 注册、构建与加载

### 注册

- `setup.py:563-571` 构建 Python 扩展 `lightop.op`。
- `csrc/export.cpp:771` 使用 `PYBIND11_MODULE(TORCH_EXTENSION_NAME, m)` 注册导出函数。
- `python/lightop/__init__.py:20` 通过 `importlib.import_module('.op', __name__)` 加载扩展。

### 资源路径

- `setup.py:584-620` 将 `configs/`、`hsa/`、`lib/` 和两个第三方 library 目录打入包。
- `python/lightop/utils.py:41-55` 按 installed package → source checkout → third_party 的顺序解析资源，并设置 `LIGHTOP_GPU_TARGET`、`LIGHTOP_ASM_DIR`。
- 当前 checkout 的直接 hsaco 文件计数：`hsa/gfx936` 39 项，`hsa/gfx938` 37 项。此数字只作为当前 commit 的完整性参考，不能硬编码成跨版本常量。

### 已确认的静默行为

- `csrc/aiter_hip_common.h:7-16`：`HIP_CALL` 失败后打印并 `exit(0)`。
- `csrc/aiter_hip_common.h:54-60`：C++ 构造器直接读取 `LIGHTOP_ASM_DIR` 并拼路径，成功标记是末尾 ` Success`。
- `python/lightop/envs.py:7-22`：不在支持列表的设备会被静默改成 `gfx936_80cu`。
- `python/lightop/gemmopt.py:75-101`：`gemm_w8a8_smooth` 在开关关闭、无 tuned config、ASM 目录缺失、bias 非空或 K<128 时返回 `(False, None)`。
- `python/lightop/config.py:28-133`：配置查找使用 `lru_cache`；精确 token 未命中时选择最近 token；未命中警告在 `:117` 被注释。
- `test/category_test_runner.py:43-59`：runner 只查 `build/lib.*` 且要求 `lightop/op*.so`。当前 checkout 只有 `build/lib` Python 文件，未发现 `op*.so`。

## 2. 真实错误 signature

| signature | 来源 | 价值 |
|---|---|---|
| `DataParallelController hit an exception` | `python/sglang/srt/managers/data_parallel_controller.py:867-870` | none：控制器捕获下游异常并通知父进程 |
| `Received sigquit from a child process` | `python/sglang/srt/entrypoints/engine.py:1676-1680` | none：子进程失败后的进程树清理 |
| `Detected some but not all shards ... are quantized` | `python/sglang/srt/layers/quantization/w8a8_int8.py:183-188` | strong：融合层 shard 精度不一致 |
| `[lightop] ... fail to call ... [HIP error]` | `oplib/lightop/csrc/aiter_hip_common.h:7-15` | strong，但进程返回码错误地为 0 |
| `no build/lib.* directory found` | `oplib/lightop/test/category_test_runner.py:43-58` | strong：测试 runner 环境问题 |
| `hipIpcOpenMemHandle ... invalid device pointer` | 运行日志引用 `DeepEP/.../rocshmem/src/ipc/backend_ipc.cpp:414`；本 checkout 无该源码 | strong：RocSHMEM IPC 层 |
| `torch.OutOfMemoryError: HIP out of memory` | PyTorch runtime + 真实日志 | strong：显存分配失败，仍需按分配点细分根因 |

## 3. 量化和并行切分契约

- `python/sglang/srt/layers/parameter.py:358-364`：`ChannelQuantScaleParameter` 等价于 column-parallel 参数，随输出通道切分。
- `parameter.py:376-420`：`PerTensorScaleParameter` 对 row / column parallel 都移除 `tp_rank`，不按 rank 切分；融合层只按 shard id 映射 scaler。
- `w8a8_int8.py:229-244`：线性权重使用 `ModelWeightParameter(input_dim=1, output_dim=0)`，scale 使用 `ChannelQuantScaleParameter(output_dim=0)`。
- `slimquant_w4a8.py:281-296`：w4a8 线性层沿用相同的 output-channel scale 契约。
- `w8a8_int8.py:164-193`：融合映射中只允许全部 shard 量化或全部 FLOAT。
- 当前 SGLang 工作树的 `w8a8_int8.py` 有未提交修改；涉及 EP DeepGEMM / LightOp layout 的结论必须在提交或固定 diff 后复核。

## 4. 版本来源

| 组件 | 推荐采集方式 | 源码位置 |
|---|---|---|
| SGLang | `python -m sglang.cli.main --version` 或 `python -c 'import sglang; print(sglang.__version__)'` | `python/sglang/version.py` + setuptools_scm |
| sglang-kernel | `python -c 'import sgl_kernel; print(sgl_kernel.__version__)'` | `python/sglang/kernels/aot/python/sgl_kernel/version.py` |
| Torch / HIP | `python -c 'import torch; print(torch.__version__, torch.version.hip)'` | 运行时包 |
| LightOp | `python -c 'import lightop; print(lightop.version, lightop.git_hash, lightop.dtk, lightop.abi, lightop.torch_version)'` | `python/lightop/version.py`；由 `get_version.py` 生成 |
| DTK | `cat /opt/dtk/.info/rocm_version`（以实际容器路径为准） | 运行环境 |
| Git checkout | `git rev-parse HEAD && git status --short` | 每个源码仓库 |

## 5. 真实日志结论

### DeepEP / RocSHMEM

- 首个决定性错误在 `eplb_20260815_181502.log:628`。
- 尾部 `DataParallelController` / `EOFError` / `SIGQUIT` 在 `:1068-1085`，距首错约 440 行。
- 结论：IPC 失败为 confirmed；导致 IPC handle 无效的更底层环境原因仍需通过设备可见性和同 rank 复现确认。

### DSpark HIP OOM

- `ifb_decode_dspark_20260814_131243.log:898` 起多个 rank 在 `13:54:35` 同秒报错。
- 分配点至少包括 LightOp INT8 Marlin MoE 的 `torch.empty(cache13)`（768 MiB）和 per-token quant 的 `torch.empty_like`（48 MiB）。
- ServerArgs 显示 `mem_fraction_static=0.96`、TP/DP=8、DSpark、decode HIP graph enabled。
- OOM 事实 confirmed；"是哪一项配置导致余量不足"尚未做二分实验，因此根因页保持 suspected。

## 未完成项

- 本机没有测试容器中的 Python 包，无法执行 GPU / import 类 verify。
- DeepEP / RocSHMEM 对应源码未在当前 SGLang checkout 中，只有运行日志里的构建路径。
- 尚未扫描所有第三方二进制内部错误字符串；首期只录入与真实日志或高价值静默行为相关的条目。
