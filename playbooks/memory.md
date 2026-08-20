# Memory playbook

## 进入条件

HIP OOM、进程被 OOM killer 杀死、运行一段时间后显存持续增长，或只在 graph capture/replay 阶段内存失败。

## 先分类

| 阶段 | 典型含义 |
|---|---|
| 权重加载 | 模型/量化权重峰值、重复副本、loader workspace |
| graph capture | private pool、capture batch tier、预热峰值 |
| 首次请求 | lazy workspace / JIT / MoE cache |
| 运行一段时间 | KV/cache 增长、泄漏、请求形状峰值、fragmentation |
| 多 rank 同秒 | 同一 workload 触发的重复失败 |

## 从 OOM 行提取

```text
rank / GPU
attempted allocation
free
allocated by PyTorch
private pools (HIP Graphs)
reserved but unallocated
最底层分配调用
```

不要仅凭日志模板建议 `expandable_segments` 就认定碎片化。

## ServerArgs

至少提取：

- `mem_fraction_static`
- `max_running_requests / max_total_tokens`
- graph on/off 和 graph max batch
- TP/DP/EP
- quant / KV cache dtype
- speculative algorithm 和 draft model
- chunked prefill / max prefill tokens

## 最小二分顺序

1. 降低 `mem_fraction_static`；
2. graph off；
3. 小 batch / 短 prompt / 低并发；
4. speculative off；
5. quant vs baseline；
6. 单卡或更小并行；
7. 用相同请求连续 replay 观察是否单调增长。

每次只改一个主变量，记录峰值显存和是否复现。

## 已知条目

- [DSpark INT8 MoE runtime OOM](../failures/dspark-moe-runtime-hip-oom.md)
- [HIP OOM symptom](../symptoms/hip-out-of-memory.md)

## 输出

给出失败阶段、分配点、总余量解释、最可能的内存池，以及下一项单变量实验。没有实验时保持 suspected。
