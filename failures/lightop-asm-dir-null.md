---
id: lightop-asm-dir-null
kind: failure
layer: kernel
components: [lightop, hip-module-loader, hsaco]
signatures:
  - 'Get GPU arch from rocminfo failed'
signature-sources:
  - "lightop/csrc/aiter_hip_common.h:54-60"
  - "lightop/python/lightop/utils.py:17-34"
detection: |
  `hipModuleLoad` 行之后没有 ` Success`，或加载低层扩展时进程段错误且 LIGHTOP_ASM_DIR 未设置。
masquerades-as:
  - process-disappeared
  - eoferror-dataparallelcontroller
repro-condition:
  - "ASM kernel 构造时 LIGHTOP_ASM_DIR 为空、路径错误、gfx 目录不匹配或 hsaco 缺失"
applies-to:
  lightop: "commit 3797bcc4 and compatible AiterAsmKernel implementation"
status: confirmed
first-seen: 2026-08-20
fixed-in:
verify: |
  # PASS: 目录存在、与实际 gfx 一致、关键 hsaco 可读，加载行后出现 Success。
  printf 'LIGHTOP_ASM_DIR=%s\n' "$LIGHTOP_ASM_DIR"
  test -n "$LIGHTOP_ASM_DIR" && test -d "$LIGHTOP_ASM_DIR"
  # ENV-MISMATCH: 当前节点没有 LightOp/rocminfo/GPU。
sources:
  - "baseline/2026-08-20-bootstrap.md"
---

# LightOp ASM 路径为空或不匹配

## 现象

```text
[lightop] hipModuleLoad: <path> GetFunction: <name>
```

本行正常情况下会以 ` Success` 结束。没有 Success 时，失败发生在 `hipModuleLoad` 或 `hipModuleGetFunction`。

## 根因

C++ 构造器直接读取 `std::getenv("LIGHTOP_ASM_DIR")` 并构造 `std::string`。如果变量未设置，行为未定义；即使变量非空，gfx/hsaco 不匹配也会进入 `HIP_CALL` 的 `exit(0)` 路径。

当前标准 Python import 路径会在 `utils.py` 中设置该变量，但以下情况仍可能失败：

- 直接加载低层扩展，未经过 `lightop.utils`；
- Python 包与 `.so` 来自不同构建；
- `rocminfo` 返回错误架构；
- 包内 `hsa/<gfx>` 资源不完整。

## 定位过程

1. 检查最后一条 LightOp 日志是否缺 `Success`。
2. 打印真实 gfx 与 `LIGHTOP_ASM_DIR`。
3. 检查具体 hsaco 文件存在性，而不是只看目录非空。
4. 检查导入的 `lightop.__file__` 和扩展 `.so` 是否来自同一安装。

## 修复 / 规避

- 通过正常 `import lightop` 路径初始化资源；
- 修正包资源或环境变量，确保路径以目录分隔符结束；
- 对齐真实 gfx 与打包 hsaco；
- 正式修复应在 C++ 对 nullptr 和文件不存在给出明确非零错误。

## 复发判据

`hipModuleLoad` 日志无 Success，且目录/hsaco/gfx 检查至少一项失败。
