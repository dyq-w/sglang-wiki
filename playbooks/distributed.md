# Distributed playbook

## 进入条件

- 无报错卡住；
- collective timeout；
- 只在 TP/DP/EP/PP>1 复现；
- DeepEP / RocSHMEM / rank 映射问题。

## 时间线原则

报 timeout 的 rank 可能已经到达等待点，真正失败的是未到达的 rank。必须收集所有 rank 同一时间窗的日志和栈。

## 证据

每个 rank 记录：

```text
node_rank / local_rank / global_rank
TP/DP/EP/PP group membership
HIP_VISIBLE_DEVICES / actual device id
last completed collective
next collective type + tensor shape/dtype
last application-level event
```

## 最小二分

1. 1 GPU；
2. 单节点多 GPU；
3. 多节点；
4. TP=1 / 目标 TP；
5. EP off / on；
6. DeepEP off / on；
7. 保持拓扑不变，关闭 PD；
8. 保存同一个触发请求做 replay。

## Hang

在授权范围内采集每个进程栈。只采一张 rank0 栈通常不足；重点对比 rank divergence：谁进入了不同分支、谁没有发起同一个 collective。

若环境已安装 SGLang 的 `debug-distributed-hang` Skill，可在完成日志/请求保存后切换使用，不在本 playbook 重复其深层流程。

## DeepEP / RocSHMEM

命中 `hipIpcOpenMemHandle` 时进入 [deepep-rocshmem-ipc-invalid-pointer](../failures/deepep-rocshmem-ipc-invalid-pointer.md)。先核对设备可见性和 rank 映射，再怀疑上层 MoE。

## 输出

- 首个发生分歧的 rank；
- 最后一个一致事件；
- collective/IPC 边界；
- 已排除的拓扑维度；
- 下一项最小二分。
