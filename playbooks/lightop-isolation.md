# LightOp isolation playbook

## 目标

脱离 SGLang 判断故障属于 LightOp 自身、构建/资源环境，还是 SGLang 调用契约。

所有命令在**实际运行容器**中执行。

## 1. 包与版本

```bash
python -c 'import lightop; print(lightop.__file__); print(lightop.version, lightop.git_hash, lightop.dtk, lightop.abi, lightop.torch_version)'
python -c 'import lightop; print(lightop.op.__file__)'
```

检查 Python 包和 `.so` 是否来自同一安装。

## 2. 设备与 ASM

```bash
python -c 'import lightop; print(lightop.gfx, lightop.LIGHTOP_ASM_DIR)'
python -c 'from lightop import envs; print(envs.device_name, envs.number_cu, envs.LMSLIM_GPU_NAME, envs.LMSLIM_USE_LIGHTOP)'
find "$LIGHTOP_ASM_DIR" -maxdepth 2 -type f -printf '%p\n'
```

不硬编码文件数；与当前 wheel/commit 的 manifest 对比。`hipModuleLoad` 行后无 `Success` 时检查具体 hsaco。

## 3. 最小无业务依赖测试

当前源码中优先选择不依赖 vllm/lmslim 的测试：

```bash
python /public/home/duanyongqiang/oplib/lightop/test/w8a8/test_w8a8_block.py
python /public/home/duanyongqiang/oplib/lightop/test/moe_w8a8/w8a8_quant.py
python /public/home/duanyongqiang/oplib/lightop/test/moe_groupgemm_w4a8/w4a8_moe_groupgemm_correctness.py
```

运行前先确认文件在当前 checkout 存在。缺 `vllm` / `lmslim` 的测试失败属于测试依赖问题，不等于算子失败。

## 4. 重现同一调用契约

从 SGLang 保存并打印：

```text
shape, dtype, stride, contiguous
M/N/K, topk, expert count
weight/scale layout
bias
stream and graph state
```

用相同输入构建最小脚本，并与 torch/reference 比较。

## 5. Build tree 陷阱

category runner 只查 `build/lib.*` 且要求 `lightop/op*.so`。当前调查 checkout 的 `build/lib` 不满足。使用：

- 完整 build 产生的实际目录并传 `--build-dir`；或
- 已安装包；
- 不要让仅含 Python 文件的 build 目录遮蔽已安装扩展。

## 归因

| 结果 | 边界 |
|---|---|
| import 失败 | ABI / DTK / package path / build |
| hsaco 加载失败 | gfx / ASM 资源 / 环境变量 |
| 单算子数值失败 | LightOp kernel / contract |
| 单算子正常、SGLang 失败 | SGLang layout / stream / graph / parallel split |
| 只 category runner 失败 | 测试 build 环境 |
