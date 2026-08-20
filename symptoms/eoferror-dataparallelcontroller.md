---
id: eoferror-dataparallelcontroller
kind: symptom
signatures:
  - "raise EOFError"
  - "DataParallelController hit an exception"
  - "Received sigquit from a child process"
signature-sources:
  - "sglang/python/sglang/srt/managers/data_parallel_controller.py:867-870"
  - "sglang/python/sglang/srt/entrypoints/engine.py:1672-1680"
diagnostic-value: none
means: "子进程已经退出，控制进程读取管道失败或收到清理信号；这些行本身不包含根因。"
action: "立即执行 playbooks/find-first-error.md，从本组消息之前定位最早异常。"
possible-causes:
  - deepep-rocshmem-ipc-invalid-pointer
  - dspark-moe-runtime-hip-oom
  - lightop-hip-call-exit-zero
  - lightop-asm-dir-null
---

# DataParallelController EOFError / SIGQUIT

## 如何识别

常见于日志尾部：

```text
DataParallelController hit an exception
raise EOFError
Received sigquit from a child process
```

## 为什么没有诊断价值

SGLang 的 DP controller 捕获异常后通知父进程；主进程的 SIGQUIT handler 再清理整个进程树。这里描述的是**清理链**，不是最早失败点。

真实案例中，RocSHMEM 的 `hipIpcOpenMemHandle` 错误比 EOFError 早约 440 行。

## 下一步

1. 查找本消息之前最早的 `Traceback / Fatal / Error / HIP error / abort`。
2. 按时间戳和 rank 重建时间线。
3. 若用户只粘贴尾部，尝试从启动脚本的 `tee / LOG_FILE / 重定向` 找到完整日志。
4. 找不到完整日志时只能给 `suspected` 结论。
