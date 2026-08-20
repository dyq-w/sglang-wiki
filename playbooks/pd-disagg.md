# PD disaggregation playbook

## 进入条件

问题仅出现在 prefill/decode 分离，涉及 Mooncake、RDMA、bootstrap、KV transfer 或角色注册。

## 同时收集两端

必须同时拿到 prefill 和 decode 端：

- 完整日志和 ServerArgs；
- 时间同步情况；
- `disaggregation_mode`、transfer backend、bootstrap endpoint；
- RDMA device 配置和实际 active 状态；
- request id / bootstrap id / transfer id（脱敏）；
- 两端 sglang、torch、DTK、Mooncake 版本。

只拿 decode 端 EOFError 无法诊断 PD。

## 分阶段定位

1. **启动/注册**：角色是否成功注册、endpoint 是否可达；
2. **资源初始化**：RDMA device、memory registration、Mooncake adapter；
3. **KV 元数据**：page size、dtype、slot/token 数一致；
4. **数据传输**：send / receive / polling / timeout；
5. **消费阶段**：decode 是否拿到正确 KV。

## 最小二分

1. 相同模型和请求改为 IFB；
2. 保持 PD，使用已知可用的 transfer backend 或环境预检；
3. 单节点 PD vs 跨节点 PD；
4. 单一 RDMA device vs topo 配置；
5. 缩短请求，减少 transfer 量；
6. 关闭量化/投机解码等非必要变量。

## 现成预检

优先复用：

```bash
python -m pytest -q test/registered/hcu/disaggregation/test_pd_environment_hcu.py
```

该测试用于设备可见性、RDMA active、router 和 Mooncake adapter 导入检查。预检通过不代表实际 transfer 正确，但可排除一批 launch 环境问题。

## 输出

明确 PD 失败阶段、哪一端先错、是否 IFB 正常、下一项最小 transfer 实验。
