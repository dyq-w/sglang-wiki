---
name: sglang-triage
description: 当用户贴出或描述 SGLang/SGLang-DAS 启动、权重加载、推理报错、进程消失、hang、HIP OOM、PD/DeepEP 通信故障、精度异常或性能回退，并询问“为什么、怎么解决、下一步怎么查”时使用。面向海光 DCU/HCU、DTK/ROCm、LightOp、量化模型、TP/DP/CP/EP/PP、IFB/PD 和 DSpark；优先主动定位启动脚本写出的完整日志，而不是只分析粘贴尾部。
---

# SGLang-DAS Triage

## 目标

把用户的一段报错或异常现象转成：

- 问题类别和失败阶段；
- 最早有诊断价值的错误；
- 当前最可信的根因候选；
- 已排除项；
- 一个最小判别实验；
- `confirmed` 或 `suspected` 结论。

只负责查询和诊断，**不修改知识库**。沉淀使用 `sglang-kb-record`。

## 硬约束

1. 阶段优先，signature 次之。
2. 不因 signature 命中直接确认根因。
3. `EOFError / DataParallelController / SIGQUIT` 默认是下游清理噪声。
4. 完整日志可用时，必须读取日志并找首错，不只分析用户粘贴片段。
5. 仅有粘贴片段时，结论强制为 `suspected (pasted excerpt only)`。
6. 不用另一台机器的版本或验证结果冒充当前运行环境。
7. 一次只推荐一个最有区分度的主实验；可列后续顺序，但不要让用户同时改很多变量。
8. `$KB/skills/` 仅用于随 Wiki 分发 Skill，不是诊断知识；问题分析、索引匹配和全文搜索时必须忽略该目录。只有用户明确询问 Skill 本身时才读取。

## 解析知识库路径

```bash
if [ -n "$SGLANG_KB_ROOT" ] && [ -d "$SGLANG_KB_ROOT" ]; then
  KB="$SGLANG_KB_ROOT"
elif [ -d "$HOME/sglang-wiki" ]; then
  KB="$HOME/sglang-wiki"
else
  echo "SGLang KB 未安装；请 clone sglang-wiki 并设置 SGLANG_KB_ROOT"
fi
```

知识库不存在时仍可做通用取证，但要明确说明未使用团队知识库，不能假装匹配过历史结论。

## 主流程

### 1. Evidence Gate：从用户现有信息主动找日志

用户通常只会粘贴报错问“这是什么”。不要先要求他们整理材料。

#### 1.1 从粘贴内容识别

- 绝对/相对日志路径：`*.log / *.out / *.txt`；
- 启动脚本：`bash x.sh`、`./launch.sh`、`source x.sh`；
- `LOG / LOG_FILE / LOG_DIR`；
- `tee`、`>file 2>&1`、`&>file`、`nohup ... -o file`；
- traceback 和命令中的工作目录。

粘贴中已有可访问日志路径时，直接读取，不再询问。

#### 1.2 有启动脚本时读取脚本

```bash
rg -n 'LOG(_FILE|_DIR)?=|tee |2>&1|&>|nohup|--log-file|--log-dir' /path/to/start.sh
```

解析变量和时间戳文件名，进入其日志目录。脚本只给了相对路径时，以脚本目录和实际工作目录分别验证。

#### 1.3 没有明确路径时小范围查找

只查当前 workspace、脚本目录、`$HOME/workspace`、`$HOME/logs`，不无界扫描整机：

```bash
find "$PWD" "$HOME/workspace" "$HOME/logs" -maxdepth 3 -type f \
  \( -name '*.log' -o -name '*.out' -o -name '*.txt' \) \
  -printf '%T@ %s %p\n' 2>/dev/null | sort -nr
```

用粘贴片段中的时间戳、PID、模型 basename 或 strong signature 验证候选。多个候选无法区分时再问用户，不擅自选。

#### 1.4 仍无完整日志

同时做两件事：

- 基于片段给初步方向，并明确 evidence level C；
- 一次性请求启动脚本路径或完整日志路径、model/quant config、运行容器版本。

命中 `diagnostic-value:none` 时直说“这是下游症状，根因通常在之前”，不要围绕它猜。

详细规则：读取 `$KB/playbooks/evidence-collection.md`。

### 2. 提取 ServerArgs 和版本

拿到日志后先查：

```bash
rg -n 'server_args=ServerArgs\(' "$LOG"
rg -n 'sglang|sglang_kernel|lightop|torch|DTK|ROCm|version|commit|build' "$LOG"
```

ServerArgs 已有 TP/DP/EP/PP、quant、graph、memory、PD、DSpark 时，不重复向用户索要启动命令。

版本缺失时，一次性要求在实际容器执行：

```bash
python -c 'import sglang; print(sglang.__version__)'
python -c 'import sgl_kernel; print(sgl_kernel.__version__)'
python -c 'import torch; print(torch.__version__, torch.version.hip)'
python -c 'import lightop; print(lightop.version, lightop.git_hash, lightop.dtk, lightop.abi, lightop.torch_version)'
cat /opt/dtk/.info/rocm_version
```

### 3. Find First Error

读取 `$KB/playbooks/find-first-error.md`。

```bash
rg -n -i 'traceback|fatal|abort|assert|out of memory|invalid device|illegal memory|undefined symbol|fail(ed)? to|exception|segmentation|timeout' "$LOG"
```

给候选分 strong / weak / none，读取前后上下文，输出：

```text
T0 最后一个成功事件
T1 最早 strong error
T2 重复错误和 cleanup
```

多 rank 同秒同错只诊断一份。无 rank 前缀 traceback 按 `Fatal Python error / Current thread / Traceback` 重建。

### 4. Determine Failure Stage

| 观察 | Stage / playbook |
|---|---|
| 权重尚未加载 | `launch` → `launch-config.md` |
| key/shape/scale/layout 加载失败 | `weights-quant` → `weight-quant.md` |
| 首次 forward / graph / 算子失败 | `kernel` → `kernel.md` |
| 仅 TP/DP/EP/PP>1、hang、collective | `distributed` → `distributed.md` |
| 仅 IFB/PD 分离、Mooncake/RDMA | `pd` → `pd-disagg.md` |
| OOM 或运行一段时间后退出 | `memory` → `memory.md` |
| 运行但乱码/NaN/数值漂移 | `accuracy` → `accuracy.md` |
| 无报错但吞吐/延迟回退 | `perf` → `perf.md` |
| 进程返回 0 但服务未 ready | 查 `process-disappeared` + LightOp |

HCU 代码/日志里仍可能沿用 `cuda_graph` 命名；诊断时按 HIP graph 理解，不因字段名误判平台。

### 5. Search Wiki

1. 读取 `$KB/index.tsv`，用 T1 的归一化错误串匹配；
2. 若目标是 symptom，先读 symptom 的 `diagnostic-value` 和 `possible-causes`；
3. 再读候选 failure 的 `applies-to / status / verify / 定位过程`；
4. 未命中时读 `$KB/heuristics.md` 和对应 stage playbook；
5. 不加载无关页面，避免旧知识污染当前判断；
6. 所有针对 Wiki 的 `find / rg / grep` 和递归读取都必须排除 `$KB/skills/**`。Skill 文本不是故障证据，不能参与 signature、根因或状态判断。

版本不适用、verify 为 ENV-MISMATCH 或条目 `fixed-in` 早于当前版本时，降低候选优先级。

### 6. Minimum Discriminating Experiment

从下列矩阵选区分度最高的一项：

| 维度 | 对照 |
|---|---|
| TP | TP=1 vs TP>1 |
| EP | off vs on |
| Graph | eager vs HIP graph |
| Quant | quant vs non-quant |
| Attention | backend A vs B |
| Kernel | SGLang vs 单算子/reference |
| GPU | 1 GPU vs multi-GPU |
| PD | IFB vs PD |
| Memory | 小 batch/短请求 vs 当前 workload |
| Speculative | DSpark off vs on |

保持其他变量不变。关闭功能后正常仅说明边界，不自动等于根因已修复。

#### 6.1 探测算子是否注册

判断 `torch.ops.<ns>.<op>` 是否存在，必须用 `getattr(...).default` 逐名探测并捕获
`AttributeError`。`dir(torch.ops.<ns>)` 是延迟枚举，不反映已注册算子，不能用于计数
或判空 —— 据此得出的「扩展是空壳」是误判。

怀疑算子缺失时，同时探测同模块的旧名算子：

- 新名缺失 + 旧名存在 → 版本错配（wheel 落后于挂载源码）；
- 全部缺失 → 扩展未加载。

两者修复方向不同，先区分再动手。

LightOp 归因读取 `$KB/playbooks/lightop-isolation.md`。

### 7. 必要时切换专用流程

已保存日志/触发请求并明确边界后：

- distributed hang → 可使用 `debug-distributed-hang`；
- kernel crash → 可使用 `debug-cuda-crash`（平台命令需适配 HCU）；
- compute 性能路径已确认 → 再使用 profiler 分析；
- 不要在证据和 replay 之前直接 profile。

## 输出格式

```markdown
## 诊断结论
- 证据等级：A / B / C
- 问题类别：
- 失败阶段：
- 最早有效错误：<file:line 或 pasted excerpt>
- 最强信号：
- 当前判断：confirmed / suspected
- 候选根因：
- 已排除：

## 下一步
1. <唯一主实验：命令/改动/预期 A 与 B 分别说明什么>
2. <后续顺序，可选>

## 风险与说明
- 是否只是临时规避：
- 是否需要恢复原 TP/EP/PD/graph 配置复测：
- 缺失证据：
```

没有完整日志时，第一段必须注明 `suspected (pasted excerpt only)`，第一项下一步是如何拿到对应完整日志；但仍应回答用户当前最可能是什么，不能只回复“请提供日志”。
