# Performance playbook

## 进入条件

无明确报错，但 TTFT、ITL、吞吐、GPU 利用率或某算子性能相对稳定 baseline 回退。

## 先有 baseline，再谈优化

固定：

- 模型/revision/量化；
- 请求长度、输出长度、并发；
- TP/DP/EP/PD；
- backend 和 graph；
- GPU/CU/DTK/torch/lightop commit；
- warmup 次数和采样窗口。

没有同条件 baseline 时，只能描述现状，不能确认 regression。

## 先排静默回退

1. 打印真实 GPU/CU 与 `LMSLIM_GPU_NAME`；
2. 打印 requested/resolved backend；
3. 记录真实 M/N/K 与 tuned config exact/selected key；
4. 修改 JSON 后重启，避免 `lru_cache`；
5. 检查是否进入 fallback / reference kernel；
6. 检查 PD/DeepEP/DSpark 是否改变 workload ownership。

相关条目：

- [unsupported device relabel](../failures/lightop-unsupported-device-relabel.md)
- [nearest config fallback](../failures/lightop-config-nearest-fallback.md)

## 阶段归因

- queue time 高：调度/容量；
- TTFT 高：prefill、权重/缓存 miss、PD transfer；
- ITL 高：decode kernel、graph、通信；
- throughput 低但 latency 正常：batching/并行利用率；
- 单算子慢：再进入 profiler。

## 不要先 profile

先用日志、配置和 exact backend 命中排除静默错误。只有问题已缩到 compute path，才使用 torch profiler 或现有性能分析 Skill。

## 最小实验

- graph off/on；
- backend A/B；
- exact tuned config vs fallback；
- IFB vs PD；
- TP/EP 单变量；
- LightOp 单算子 benchmark vs SGLang e2e。

## 输出

给出 baseline 差异、回退发生在哪个阶段、是否命中预期 backend/config、下一项单变量实验。
