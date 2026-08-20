---
id: lightop-unsupported-device-relabel
kind: failure
layer: perf
components: [lightop, device-detection]
signatures: []
signature-sources: []
detection: |
  真实 `device_name_numberCU` 不在 SUPPORTED_DEVICES 中，但 `LMSLIM_GPU_NAME` 输出为 `gfx936_80cu`。
masquerades-as:
  - silent-performance-regression
repro-condition:
  - "GPU 架构/CU 组合不在 lightop.envs.SUPPORTED_DEVICES"
applies-to:
  lightop: "commit 3797bcc4; python/lightop/envs.py:7-22"
status: confirmed
first-seen: 2026-08-20
fixed-in:
verify: |
  # PASS: actual_name == LMSLIM_GPU_NAME，或 unsupported device 明确禁用而非改名。
  python -c 'import torch; from lightop import envs; p=torch.cuda.get_device_properties("cuda"); print(p.gcnArchName, p.multi_processor_count, envs.LMSLIM_GPU_NAME, envs.LMSLIM_USE_LIGHTOP)'
  # ENV-MISMATCH: 当前节点无 torch/lightop/GPU。
sources:
  - "baseline/2026-08-20-bootstrap.md"
---

# 不支持的 GPU/CU 被静默改名

## 现象

服务能运行，但性能显著偏低或 tuned config 命中异常；日志里可能没有任何错误。

## 根因

`lightop.envs` 先构造真实名 `<gfx>_<cu>cu`，不在支持列表时无警告地改成 `gfx936_80cu`。随后 LightOp 可能按错误设备名选择开关和 tuned 参数。

当前支持列表：

```text
gfx936_80cu
gfx928_128cu
gfx928_120cu
gfx938_64cu
```

列表会随版本变化，以上只对应调查 commit。

## 定位过程

同时打印：

- `gcnArchName`；
- `multi_processor_count`；
- 拼出的真实 `<gfx>_<cu>cu`；
- `LMSLIM_GPU_NAME`；
- `LMSLIM_USE_LIGHTOP`。

若真实名与映射名不同，即命中本条；无需先跑 profiler。

## 修复 / 规避

- 未支持设备默认禁用 LightOp 或显式报错，比伪装成另一设备安全；
- 添加该设备的真实配置和测试后再加入支持列表；
- 临时规避必须记录实际硬件，不得把错误映射的性能数据作为正式 baseline。

## 复发判据

真实设备名不等于 `LMSLIM_GPU_NAME`，且后者固定为 `gfx936_80cu`。
