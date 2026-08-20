---
id: lightop-config-nearest-fallback
kind: failure
layer: perf
components: [lightop, tuned-config]
signatures: []
signature-sources: []
detection: |
  实际 M/N/K 或 token 数没有精确配置项，但执行继续；选中的配置来自最近 token，且日志没有 fallback 告警。
masquerades-as:
  - silent-performance-regression
repro-condition:
  - "tuned config 文件存在，但当前 shape/token key 不精确命中"
  - "同一进程内修改 JSON 后再次调用"
applies-to:
  lightop: "commit 3797bcc4; python/lightop/config.py"
status: confirmed
first-seen: 2026-08-20
fixed-in:
verify: |
  # PASS: 记录 requested key、selected key 与是否 exact match。
  # 修改 JSON 后必须重启进程再验证；否则结果可能来自 lru_cache。
  # ENV-MISMATCH: 当前节点无法导入 LightOp。
sources:
  - "baseline/2026-08-20-bootstrap.md"
---

# LightOp tuned config 静默最近邻回退

## 现象

- 数值结果正确；
- 性能比已知 baseline 差；
- 修改配置 JSON 后当前进程表现没有变化；
- 日志没有 "config not found"。

## 根因

`get_moe_cuda_config` 等函数：

1. 精确 token key 不存在时，搜索与 M 最接近的 token；
2. 配置仍为空时只返回 `status=False`；
3. 原本的告警输出被注释；
4. 函数使用 `functools.lru_cache`，进程内结果不会因 JSON 修改自动失效。

## 定位过程

1. 记录实际 E/M/N/K/topk/EP size/gfx/CU。
2. 构造真实配置文件名，确认文件和 shape key。
3. 记录 exact key 与 selected nearest key。
4. 重启进程后再比较 JSON 修改效果。
5. 若重启后精确命中且性能恢复，才能 confirmed 为本条。

## 修复 / 规避

- 在配置选择路径打印 `requested / selected / exact`；
- 调试配置时每次修改后重启，或在受控测试中调用对应 cache 的 `cache_clear()`；
- 为真实 workload shape 补 tuned entry；
- 不把最近邻回退的结果作为正式性能基线。

## 复发判据

同一 shape 重复出现非精确命中，且精确配置/重启后性能恢复。
