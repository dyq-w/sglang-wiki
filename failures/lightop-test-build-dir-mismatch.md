---
id: lightop-test-build-dir-mismatch
kind: failure
layer: launch
components: [lightop, test-runner, build]
signatures:
  - "no build/lib\\.\\* directory found; rerun with --build"
  - "compiled lightop/op\\.\\*\\.so is missing"
  - "Ninja is required to load C\\+\\+ extensions"
signature-sources:
  - "lightop/test/category_test_runner.py:43-58"
  - "lightop/setup.py:109-123"
detection: |
  当前 checkout 存在 `build/lib`，但 runner 只 glob `build/lib.*`；或目标目录中没有 `lightop/op*.so`。
masquerades-as: []
repro-condition:
  - "直接运行 category-level runner，未传有效 --build-dir"
  - "当前 build 只有 Python 文件或使用 build/lib 而非 build/lib.*"
applies-to:
  lightop: "commit 3797bcc4 current checkout"
status: confirmed
first-seen: 2026-08-20
fixed-in:
verify: |
  # PASS: runner 指向的目录存在 lightop/op*.so，并且 import 来自该目录。
  find "$LIGHTOP_ROOT/build" -maxdepth 3 -type f -name 'op*.so' -print
  # NOT-APPLICABLE: 使用已安装包测试而不走 category runner。
sources:
  - "baseline/2026-08-20-bootstrap.md"
---

# LightOp 测试 runner 与 build 目录不匹配

## 现象

```text
no build/lib.* directory found; rerun with --build
```

或者：

```text
compiled lightop/op*.so is missing
```

## 根因

category runner 默认只查 `build/lib.*<abi>` / `build/lib.*`，并要求其中存在 `lightop/op*.so`。当前调查 checkout 只有 `build/lib` 下的 Python 文件，没有发现扩展 `.so`。

这是**测试脚本/构建环境问题**，不是被测试算子的数值错误。

## 定位过程

1. 检查 runner 的实际 `PYTHONPATH`。
2. 列出 build 目录和 `op*.so`。
3. 确认 import 的 `lightop.__file__` / `lightop.op.__file__`。
4. 区分"已安装包可用"和"源码 build tree 可用"。

## 修复 / 规避

- 完整执行 build 后将 `--build-dir` 指向实际含 `lightop/op*.so` 的目录；
- 或在隔离测试中使用已安装包；
- 不要把仅含 Python 文件的 `build/lib` 放到最前面的 `PYTHONPATH`。

## 复发判据

runner 仍找不到预期目录/扩展，或者 import 路径与待测构建不一致。
