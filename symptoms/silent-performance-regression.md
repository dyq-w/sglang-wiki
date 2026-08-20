---
id: silent-performance-regression
kind: symptom
signatures: []
signature-sources: []
diagnostic-value: weak
means: "服务可以启动且结果可能正确，但吞吐、延迟或算子命中情况明显劣化。"
action: "执行 playbooks/perf.md；先核对 baseline、真实设备映射和 tuned config 命中，再考虑 profiler。"
possible-causes:
  - lightop-unsupported-device-relabel
  - lightop-config-nearest-fallback
---

# 静默性能回退

## 判定方式

满足以下至少一项：

- 同配置、同模型、同硬件相对已知 baseline 有稳定回退；
- 日志未显示错误，但预期 LightOp / tuned kernel 没有命中；
- 真实 GPU/CU 与 `LMSLIM_GPU_NAME` 不一致；
- tuned JSON 已修改，但重启前后实际命中项没有变化。

## 下一步

1. 固定模型、请求和并行配置，先得到可重复 baseline。
2. 打印真实设备、LightOp 映射名和开关。
3. 检查配置 key 与实际 M/N/K/EP size。
4. 清理进程级 cache 只能通过重启或明确调用 cache_clear；不要先做 profiler。
