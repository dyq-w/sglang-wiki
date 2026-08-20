# 领域启发式

这些规则用于确定调查方向，**不能单独把 failure 标成 confirmed**。

## 时间线与多进程

- 日志尾部的 `EOFError`、`SIGQUIT`、父进程 pipe 失败通常没有根因信息；从最早异常往后读。
- 同一秒多个 rank 报同一错误，优先找该组中最早开始 traceback 的 rank，只诊断一份，其余视为重复噪声。
- collective timeout 中，打印 timeout 的 rank 往往已经到达等待点；真正异常的可能是没有到达的 rank。
- 无 rank 前缀的多进程 traceback 会字符级交错；不要按完整行直接搜索唯一 traceback，应结合 `Fatal Python error`、`Current thread` 和带 rank 前缀区段重建时间线。

## 量化与并行

- TP=1 正确、TP>1 乱码或 NaN：优先检查 scale 切分和权重 layout。
- per-channel weight scale 应与输出通道同向切分；per-tensor scale 不应按 TP rank 切成不同数值。
- 融合 QKV / gate-up 层的 shard 必须使用一致精度；代码会对部分量化、部分浮点的组合主动报错。
- w4a8 shape 问题先确认 pack factor、K/N tile 整除和 scale shape，再怀疑 kernel。

## HIP graph 与异步错误

- HIP graph capture 失败而 eager 正常：检查动态 shape、host-device 同步、非法 stream 操作和 capture 期额外内存。
- `illegal memory access` 常在后续同步点才暴露；报错位置不一定是真实出错位置。使用平台支持的同步启动调试方式复现，并尽量缩成单算子用例。
- graph private pool 会单独占显存；OOM 日志中的 private-pool 数值必须与普通 allocated/reserved 一起看。

## LightOp 静默类

- `[lightop] hipModuleLoad: ... GetFunction: ...` 后没有 ` Success`：检查 hsaco 文件、gfx 目录和 `LIGHTOP_ASM_DIR`；`HIP_CALL` 目前会 `exit(0)`。
- 当前 checkout 中 `utils.py` 会自动设置 `LIGHTOP_ASM_DIR`；但直接加载低层扩展、初始化顺序异常或使用不一致构建产物时仍可能为空。
- 改 tuned config 后表现不变：多个配置函数使用 `functools.lru_cache`，必须重启进程或显式清 cache 才能验证。
- 未精确命中 tuned config 时，代码可能静默选择最近 token 数配置；数值正确但性能下降时要核对实际命中项。
- 未支持的 GPU/CU 组合会被映射为 `gfx936_80cu`；必须同时打印真实 `device_name / number_cu` 与 `LMSLIM_GPU_NAME`。
- `gemm_w8a8_smooth` 可能返回 `(False, None)`；调用方未检查 status 时，None 可能在更下游才报错。

## 证据等级

- **A**：完整日志 + ServerArgs + 版本 + 可复现实验；可进入 confirmed 评估。
- **B**：完整日志但缺版本或排他实验；最多 suspected。
- **C**：只有粘贴片段；只能给初步方向，不能写入 Wiki。
