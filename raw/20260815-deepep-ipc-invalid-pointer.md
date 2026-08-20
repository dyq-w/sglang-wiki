# 2026-08-15 DeepEP / RocSHMEM IPC 证据快照

原始文件（仅当前机器）：

```text
/public/home/duanyongqiang/workspace/eplb_20260815_181502.log
```

## 必要配置子集

```text
nnodes=2
node_rank=1
tp_size=16
dp_size=16
ep_size=16
moe_a2a_backend=deepep
disaggregation_mode=decode
```

内网地址、模型绝对路径和无关 ServerArgs 已省略。

## 时间线

```text
L628 Error: hipIpcOpenMemHandle(...): invalid device pointer (17)
     at ... DeepEP/third-party/rocshmem/src/ipc/backend_ipc.cpp:414
L629 Fatal Python error: Aborted
...
L1068 DataParallelController hit an exception: Traceback ...
L1082 raise EOFError
L1083 EOFError
L1085 Received sigquit from a child process. It usually means the child failed.
```

## 证据解释

- `hipIpcOpenMemHandle` 是最早的强诊断错误。
- 多个 rank 后续重复 `Fatal Python error`，日志发生字符级交错。
- EOFError / SIGQUIT 比首错晚约 440 行，是清理链而非根因。
