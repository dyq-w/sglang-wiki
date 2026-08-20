# Kernel playbook

## 进入条件

权重已加载，首次 forward、预热、eager 或 HIP graph capture/replay 时失败。

## 先分 eager / graph

1. 相同输入 eager 是否复现；
2. 关闭 decode/prefill graph 后是否复现；
3. 只在 capture 失败还是 replay 后失败；
4. OOM 时记录 graph private pool。

**graph 关闭后正常只完成归因，不等于修复。**

## 再分框架 / 单算子

- 保存最小 shape、dtype、stride、layout 和 scalar 参数；
- 脱离 SGLang 调同一算子；
- 与 torch/reference 比数值；
- 单算子失败 → 算子/ABI/hsaco；
- 单算子正常 → SGLang 调用契约、stream、workspace 或并行切分。

LightOp 使用 [lightop-isolation.md](lightop-isolation.md)。

## 异步错误

HIP kernel 错误可能在后续同步点暴露。调试时：

- 缩到一个请求、一个 batch、一个算子；
- 使用当前平台支持的同步启动/错误检查机制；
- 记录最后一个成功 kernel；
- 不把 traceback 的最外层 Python 调用当真实 kernel 出错点。

## Shape / layout

至少记录：

```text
shape, dtype, device, stride, contiguous
A/B 是否转置
scale shape/dtype
M/N/K, topk, expert count
stream/capture state
```

## 已知条目

- [HIP_CALL exit(0)](../failures/lightop-hip-call-exit-zero.md)
- [ASM dir / hsaco](../failures/lightop-asm-dir-null.md)
- [w8a8 smooth None](../failures/lightop-w8a8-smooth-none.md)

## 输出

给出：eager/graph 边界、框架/单算子边界、首个失败 API/kernel 和下一项单变量实验。
