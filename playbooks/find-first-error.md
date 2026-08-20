# Find first error

## 目标

找到**最早具有解释力的异常**，而不是最后一条 traceback。

## 1. 建立候选行

```bash
rg -n -i 'traceback|fatal|abort|assert|out of memory|invalid device|illegal memory|undefined symbol|fail(ed)? to|exception|segmentation|timeout' "$LOG"
```

不要机械地把第一条包含 `error` 的日志当首错；先查 `index.tsv` 中对应 symptom 的 `diagnostic-value`。

## 2. 给候选分级

- **strong**：HIP API 明确错误、OOM、undefined symbol、shape/assert、源文件:行号。
- **weak**：watchdog、timeout、进程退出、health fail。
- **none**：DP controller EOF、SIGQUIT 清理、父进程 pipe fail。

优先选择最早的 strong；若只有 weak，再沿其前置 rank/进程找原因。

## 3. 读取上下文

对候选行至少读取前后 40-100 行，确认：

- 发生阶段；
- rank / node / process；
- 前一个成功事件；
- 是否同秒多 rank 重复；
- traceback 最底层业务调用。

## 4. 处理交错日志

先抽带 rank 前缀的行：

```bash
rg -n '^\[.*(DP|TP|EP)[0-9]+' "$LOG"
```

无前缀 traceback 按这些锚点切块：

```text
Fatal Python error
Current thread
Thread 0x
Traceback (most recent call last)
```

字符级交错时不要依赖完整行；组合源文件路径、函数名和时间戳。

## 5. 同秒多 rank 聚合

- 找最早开始 traceback 的 rank；
- 对比每个 rank 的最底层调用是否相同；
- 相同则只保留一份主 traceback，其余记录为重复数；
- 不同则可能有一个首错和多个后续 collective/cleanup 错误。

## 6. 输出时间线

至少给三点：

```text
T0  最后一个明确成功事件
T1  最早 strong error（根因候选）
T2  下游重复错误 / cleanup
```

只有 T1 能进入 failure 匹配；T2 不得直接下结论。
