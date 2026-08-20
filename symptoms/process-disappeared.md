---
id: process-disappeared
kind: symptom
signatures:
  - "process exited with code 0 but service is not ready"
  - "child process disappeared without traceback"
signature-sources:
  - "observation:process supervisor / shell exit status"
diagnostic-value: weak
means: "进程已退出但没有足够错误信息；可能是 signal、abort、OOM、低层 exit(0) 或初始化顺序问题。"
action: "检查最后一条未闭合的低层日志、shell 返回码、dmesg/coredump（如获授权）和启动脚本实际日志路径。"
possible-causes:
  - lightop-hip-call-exit-zero
  - lightop-asm-dir-null
---

# 进程直接消失

## 高价值线索

如果 stdout 最后一条是：

```text
[lightop] hipModuleLoad: <path> GetFunction: <name>
```

且没有后续 ` Success`，优先进入 LightOp ASM 加载路径。

## 取证

1. 从启动脚本解析 `tee`、`LOG_FILE`、`>file 2>&1`、`nohup -o`。
2. 检查 supervisor / shell 记录的返回码。
3. 查看最后 200 行，但根因搜索仍从全日志最早异常开始。
4. 仅在用户授权并且必要时读取系统 coredump / kernel log。
