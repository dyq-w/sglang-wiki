---
id: lightop-hip-call-exit-zero
kind: failure
layer: kernel
components: [lightop, hip-runtime]
signatures:
  - '\[lightop\].*fail to call.*\[HIP error\]'
signature-sources:
  - "lightop/csrc/aiter_hip_common.h:7-15"
detection: |
  服务未就绪或进程消失，shell/supervisor 记录返回码 0；stdout 最后出现 LightOp HIP API 失败。
masquerades-as:
  - process-disappeared
  - eoferror-dataparallelcontroller
repro-condition:
  - "任一被 HIP_CALL 包裹的 HIP API 返回非 hipSuccess"
applies-to:
  lightop: "commit 3797bcc4 and compatible code containing exit(0) macro"
status: confirmed
first-seen: 2026-08-20
fixed-in:
verify: |
  # PASS: 宏的错误分支仍包含 exit(0)。
  rg -n -U 'hipGetErrorString\(err\).*\n.*exit\(0\)' "$LIGHTOP_ROOT/csrc/aiter_hip_common.h"
  # NOT-APPLICABLE: 当前 LightOp 已将该分支改成非零退出或异常。
sources:
  - "baseline/2026-08-20-bootstrap.md"
---

# LightOp HIP API 失败后 exit(0)

## 现象

- 进程直接消失；
- 可能没有 Python traceback；
- supervisor 误认为退出成功；
- 多进程上层随后只看到 EOFError / SIGQUIT。

## 根因

`HIP_CALL` 宏在任何 HIP API 返回错误时打印一行后调用 `exit(0)`。返回码 0 把失败伪装成正常退出。

## 定位过程

静态代码检查确认：

```cpp
if (err != hipSuccess) {
    printf("... [HIP error](%s) ...", hipGetErrorString(err));
    exit(0);
}
```

## 修复 / 规避

- 短期：监控"服务未 ready + 进程已退出"，不要只依赖返回码；保存 stdout/stderr。
- 短期：命中 `[lightop] ... fail to call` 即按失败处理。
- 正式修复：将错误分支改为非零退出或抛出可传播异常，并增加回归测试；该改动不在本次知识库搭建范围。

## 复发判据

- 任一 LightOp HIP error 后进程返回 0；
- 上层只有 child failed / EOFError；
- 静态 verify 仍能找到 `exit(0)`。
