# Accuracy playbook

## 进入条件

服务能运行，但输出乱码、NaN、重复、明显偏离 reference，或只在某个并行/量化/backend 组合下错误。

## 先固定可比条件

- 相同 prompt / token ids；
- greedy 或固定 seed；
- 相同 tokenizer/chat template；
- 相同模型 revision；
- 记录 logits/hidden state 的比较点，而不是只看文本。

## 二分矩阵

1. TP=1 vs TP>1；
2. EP off vs on；
3. quant vs non-quant；
4. eager vs graph；
5. attention backend A vs B；
6. speculative off vs on；
7. SGLang kernel vs reference / 单算子。

## 定位首个偏离层

1. 保存 reference 的层输入/输出摘要；
2. 在目标配置逐层比较；
3. 找第一个超阈值或出现 NaN 的层；
4. 再对该层检查 weight/scale/layout/rank 分片；
5. 不要从最终乱码反推具体 kernel。

## 量化重点

- per-channel / per-tensor / group-wise 选择；
- scale shape 和切分方向；
- packed weight 的 K/N/layout；
- `process_weights_after_loading` 重排；
- EP expert ownership 和 scale 对齐；
- KV cache dtype 与 attention backend 支持。

参考：[quant-scale-tp-split](../failures/quant-scale-tp-split.md)。

## NaN

找到首个产生 NaN/Inf 的算子和输入，而不是在最终 logits 才检查。记录输入范围、scale 范围、dtype 和是否发生溢出/未初始化。

## 输出

- reference 配置；
- 首个偏离层/算子；
- 已排除的维度；
- 当前状态；
- 下一项最小数值实验。
