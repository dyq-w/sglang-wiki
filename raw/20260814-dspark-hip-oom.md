# 2026-08-14 DSpark HIP OOM 证据快照

原始文件（仅当前机器）：

```text
/public/home/duanyongqiang/workspace/ifb_decode_dspark_20260814_131243.log
```

## 必要配置子集

```text
tp_size=8
dp_size=8
ep_size=1
quantization=slimquant_marlin
speculative_algorithm=DSPARK
mem_fraction_static=0.96
decode HIP graph enabled
```

模型绝对路径、请求正文和无关 ServerArgs 已省略。

## 时间线

```text
13:54:35 DP5/TP5 Scheduler traceback begins
... lightop/.../fused_moe/int8_marlin.py
... torch.empty(cache13)
torch.OutOfMemoryError: HIP out of memory (768 MiB allocation)

13:54:35 其他多个 rank 同秒报同类错误
... per_token_quant_int8
... torch.empty_like(..., dtype=torch.int8)
torch.OutOfMemoryError: HIP out of memory (48 MiB allocation)

13:54:38 scheduler process crashed; cleanup SIGQUIT
```

## 证据解释

- OOM 发生在服务已运行后的 forward，不是权重加载。
- 多 rank 同秒失败应视为一个根因的重复噪声。
- GPU free 仅 0-204 MiB，而临时 tensor 需要 48-768 MiB。
- `full token usage` 约 0.13-0.16，不代表总显存充足；权重、draft/target、graph pool 和 workspace 不在该比率中。
- 尚未做单变量二分，因此具体配置根因保持 suspected。
