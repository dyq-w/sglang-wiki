# Launch / config playbook

## 进入条件

权重尚未开始加载就失败，或服务进程无法完成初始化。

## 先确认

- 实际启动脚本和日志路径；
- 当前 shell / 容器中的 `HIP_VISIBLE_DEVICES`；
- node rank、nnodes、dist init addr、base GPU id 和 step；
- 包是否来自预期环境；
- 失败在 import、参数校验、进程 spawn、distributed init 还是设备初始化。

## 最小实验

1. `python -c 'import sglang'`；
2. `python -c 'import torch; print(torch.cuda.is_available(), torch.version.hip)'`；
3. `python -c 'import lightop; print(lightop.__file__)'`（仅目标 GPU 环境）；
4. 单卡最小启动；
5. 保持模型不变，去掉 PD / DeepEP / DSpark 等非必要功能；
6. 多节点问题先验证每个节点单独启动，再验证 rendezvous。

## 常见分流

- `undefined symbol` / `.so` 不存在 → ABI / build / load path；
- `ROCM_HOME environment variable is not set` → build 环境，不是运行时模型问题；
- `no build/lib.*` → [LightOp test build mismatch](../failures/lightop-test-build-dir-mismatch.md)；
- 子进程直接消失 → [process-disappeared](../symptoms/process-disappeared.md)；
- 只有 EOF / SIGQUIT → 回到 [find-first-error](find-first-error.md)。

## 输出

明确失败边界：import / argument validation / process spawn / device init / distributed init。不要笼统写"启动失败"。
