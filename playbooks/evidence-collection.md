# Evidence collection

## 目标

尽量从用户已经给出的内容中主动找到完整日志；不要把"请重新提供所有信息"作为第一反应。

## 证据等级

| 等级 | 内容 | 允许的结论 |
|---|---|---|
| A | 完整日志 + ServerArgs + 版本 + 可复现实验 | 可评估 confirmed |
| B | 完整日志，但缺版本或排他实验 | suspected |
| C | 只有聊天框粘贴片段 | 仅初步方向；不得写入 Wiki |

## 用户只粘贴报错时

### 1. 先从对话内容找线索

识别：

- `/absolute/path/*.log`、`workspace/*.log`、`logs/*.txt`；
- `bash /path/start.sh`、`./launch.sh` 等启动脚本；
- `LOG=...`、`LOG_FILE=...`、`LOG_DIR=...`；
- `tee <file>`、`> <file> 2>&1`、`&> <file>`、`nohup ... -o <file>`；
- traceback 的源码/工作目录路径。

### 2. 能访问启动脚本就直接读取

只读解析：

```bash
rg -n 'LOG(_FILE|_DIR)?=|tee |2>&1|&>|nohup|--log-file|--log-dir' /path/to/start.sh
```

处理变量展开和时间戳命名。例如：

```bash
LOG="$HOME/workspace/run_$(date +%Y%m%d_%H%M%S).log"
python ... 2>&1 | tee "$LOG"
```

在对应目录按 mtime、启动时间和粘贴 signature 选日志，不要只取文件名排序的最后一个。

### 3. 没有脚本路径时做小范围搜索

只搜索当前项目和常见日志目录；不要无界扫描整个文件系统：

```bash
find "$PWD" "$HOME/workspace" "$HOME/logs" -maxdepth 3 -type f \
  \( -name '*.log' -o -name '*.out' -o -name '*.txt' \) \
  -printf '%T@ %s %p\n' 2>/dev/null | sort -nr
```

多个候选时，用粘贴片段里的 timestamp、PID、model basename 或强 signature 交叉确认；不要猜一个路径。

### 4. 仍找不到

追问一次并同时工作：

- 说明当前片段中 strongest signature；
- 标明它是上游强错误还是 `diagnostic-value:none` 的尾部噪声；
- 给 suspected 初判；
- 一次性请求：启动脚本路径或完整日志路径、模型配置、版本信息。

## 拿到完整日志后

```bash
rg -n 'server_args=ServerArgs\(' "$LOG"
rg -n 'sglang|sglang_kernel|lightop|torch|DTK|ROCm|version|commit|build' "$LOG"
```

ServerArgs 通常已包含 TP/DP/EP/PP、量化、graph、memory、DSpark 和 PD 配置，不再向用户重复索要启动命令。

## 版本必须来自同一运行环境

在实际测试容器执行：

```bash
python -c 'import sglang; print(sglang.__version__)'
python -c 'import sgl_kernel; print(sgl_kernel.__version__)'
python -c 'import torch; print(torch.__version__, torch.version.hip)'
python -c 'import lightop; print(lightop.version, lightop.git_hash, lightop.dtk, lightop.abi, lightop.torch_version)'
cat /opt/dtk/.info/rocm_version
```

命令不可用时记录 `ENV-MISMATCH`，不要用另一台机器的结果替代。

## 请求相关故障

若问题只被某类请求触发，在重启或改配置前保存脱敏后的实际请求、请求长度/并发和触发时序。能 replay 的证据优先于一次性现场观察。

## 敏感信息

复制到 Wiki 前删除 token、密钥、业务请求正文、无必要内网地址和用户身份信息。
