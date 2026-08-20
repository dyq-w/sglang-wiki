---
id: hip-out-of-memory
kind: symptom
signatures:
  - "torch.OutOfMemoryError: HIP out of memory"
  - "HIP out of memory. Tried to allocate"
signature-sources:
  - "external:pytorch:runtime allocator error"
diagnostic-value: strong
means: "某次 HIP 显存分配失败；错误事实可靠，但根因仍取决于失败阶段、分配点和内存余量。"
action: "执行 playbooks/memory.md；先找第一个报 OOM 的 rank 和具体分配调用，再做 graph / batch / mem_fraction 二分。"
possible-causes:
  - dspark-moe-runtime-hip-oom
---

# HIP out of memory

## 不能只看什么

- 不能只看最后一个 rank。
- 不能只看 `reserved but unallocated` 就认定是碎片化。
- 不能因为日志建议 `expandable_segments` 就直接把它当修复。

## 必须提取

- 第一个 OOM 的时间、rank、分配大小；
- `allocated / private pools / reserved / free`；
- 分配点对应的 Python/C++ 调用；
- ServerArgs 中 `mem_fraction_static`、graph、batch、量化、投机解码和并行配置；
- 是否所有 rank 同秒失败。

## 结论边界

OOM 本身可标为 confirmed；导致余量不足的具体配置只有经过最小二分实验后才能 confirmed。
