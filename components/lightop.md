# LightOp 代码基线

调查基线：commit `3797bcc4c97c1b08bd4d564e1be81e150e8d9b51`。

## 注册与导入

| 环节 | 位置 | 说明 |
|---|---|---|
| 扩展定义 | `setup.py:563-571` | `CUDAExtension(name="lightop.op", ...)` |
| C++ 导出 | `csrc/export.cpp:771` | `PYBIND11_MODULE(TORCH_EXTENSION_NAME, m)` |
| Python 导入 | `python/lightop/__init__.py:20` | 导入 `.op` |
| Python API | `python/lightop/gemmopt.py` 等 | 包装 `op.<function>` |

## 资源加载

`python/lightop/utils.py:41-55` 的优先级：

1. 已安装包中的资源；
2. 源码 checkout 根目录；
3. `third_party/`。

随后设置：

- `LIGHTOP_GPU_TARGET`
- `LIGHTOP_ASM_DIR`

`setup.py:584-620` 把 `configs`、`hsa`、`lib` 等复制到安装包。

当前 checkout 的直接 hsaco 文件数：

- `hsa/gfx936`：39
- `hsa/gfx938`：37

只用于本 commit 的完整性对比，不是跨版本常量。

## 高价值契约

### GEMM layout

`gemmopt.py` 多个接口要求：

- A / B 为 2D；
- A contiguous；
- 部分 col-major B 明确要求 `b.is_contiguous() == False`；
- w4a8 / marlin 路径对 packed K/N 和 tile 整除有额外要求。

遇到 shape/layout 问题，先打印：shape、stride、dtype、contiguous、transpose 历史和 scale shape。

### ASM 成功标记

`AiterAsmKernel` 的输出只有在 `hipModuleLoad` 和 `hipModuleGetFunction` 都成功后才追加 ` Success`。因此"加载行未闭合"是无 traceback 场景的重要 detection。

### 返回值契约

`gemm_w8a8_smooth` 返回 `(status, output)`；`status=False` 时 output 为 None。调用方必须检查 status。

## 版本

`python/lightop/version.py` 包含：

- `version`
- `git_hash`
- `git_branch`
- `abi`
- `dtk`
- `torch_version`
- `hcu_version`

由 `get_version.py` 在构建时生成。应在**实际测试容器内**采集，不能只看源码 checkout。

## 已知测试边界

`test/category_test_runner.py` 默认要求 `build/lib.*` 中存在 `lightop/op*.so`。当前 checkout 的 `build/lib` 只有 Python 文件；这类错误属于测试构建环境，不代表算子数值失败。
