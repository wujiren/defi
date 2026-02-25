## 模块 6：全链路 Trace ID 日志规范

### 1. Trace ID 生命周期流转 (The Lifecycle)

为了覆盖 PRD 要求的全生命周期，Trace ID 必须在以下关键节点进行物理传递：

1. **产生阶段 (AI Agent)**：Python 策略引擎在生成交易信号时，立即创建一个 `UUID v4` 格式的 `trace_id`。
2. **传输阶段 (gRPC)**：在调用 `ExecuteTrade` 时，`trace_id` 作为 `Context` 字段通过 gRPC 传递给 Go 执行层。
3. **持久化阶段 (PostgreSQL)**：Go 执行层接收后，将其写入 `trade_tasks` 表的 `trace_id` 列。
4. **签名阶段 (MPC)**：Go 发送 Payload 给签名机时，必须在日志中携带该 ID，以便审计签名权限释放的依据。
5. **链上匹配阶段 (Event Indexer)**：当交易确认后，Indexer 通过 `tx_hash` 关联回数据库中的 `trace_id`，完成闭环。

---

### 2. 标准日志格式规范 (Structured Logging)

系统统一采用 **JSON 结构化日志** 格式，以便 ELK (Elasticsearch, Logstash, Kibana) 或 Grafana Loki 快速检索。

| 字段 (Key) | 类型 (Type) | 说明 (Description) |
| --- | --- | --- |
| `trace_id` | `string` | 必填，全局唯一标识。 |
| `span_id` | `string` | 选填，用于区分模块内部子任务（如跨链的一跳）。 |
| `component` | `string` | 模块标识：`Strategy_Agent`, `Go_FSM`, `Simulator`, `MPC_Signer`。 |
| `level` | `string` | `INFO`, `WARN`, `ERROR`, `CRITICAL`。 |
| `sync_block` | `int64` | 水位线区块号，用于诊断“过时数据”导致的拦截。 |
| `internal_tx_id` | `string` | 对应数据库 `trade_tasks` 的主键。 |
| `msg` | `string` | 描述性文本。 |
| `error_code` | `string` | 错误码，如 `STALE_DATA_ERROR`, `SIMULATION_REVERT`。 |

---

### 3. 跨语言/组件的传递实现

#### 3.1 Python (Agent) 侧日志示例

```python
# 决策触发日志
logger.info("AI Decision Generated", extra={
    "trace_id": "550e8400-e29b-41d4-a716-446655440000",
    "agent_id": "trend_follower_01",
    "sync_block": 19238410,
    "msg": "Moving average crossover detected, proposing SWAP"
})

```

#### 3.2 Go (FSM) 侧仿真日志

```go
// 仿真结果日志
log.Info("Simulation Completed",
    zap.String("trace_id", req.Ctx.TraceId),
    zap.String("component", "Simulator"),
    zap.Uint64("gas_used", result.GasUsed),
    zap.Bool("success", true),
)

```

---

### 4. 异常监控与告警触发 (Alerting)

通过 Trace ID 关联，系统需实现以下自动告警逻辑：

* **延迟告警**：若同一 `trace_id` 从 `INIT` 到 `SIGNED` 的跨度超过 500ms，触发性能预警。
* **状态卡死告警**：若 `trade_tasks` 中某记录停留在 `BROADCASTED` 状态超过 5 个区块且无 `tx_hash`，自动触发“阶梯 Gas 加速”并记录关联日志。
* **反向收割审计**：若 **模块 4.3** 审计发现“毒性流”，通过 `trace_id` 自动定位是哪一个 AI 模型的策略参数导致了亏损，并更新 **模块 6.2** 的反馈飞轮。

---

### 5. 评价标准 (Evaluation Standards)

| 维度 | **好日志架构 (Observability)** | **坏日志架构 (Traditional)** |
| --- | --- | --- |
| **故障定位** | 数秒内通过 Trace ID 还原交易在仿真区、签名区的全路径。 | 必须登录不同服务器，手动对比时间戳来猜测交易失败原因。 |
| **精度对齐** | 日志中明确记录了决策时的水位线 `sync_block`。 | 无法确定 AI 决策时参考的价格是否已过时。 |
| **闭环能力** | 实际盈亏数据自动通过 Trace ID 反馈给 AI Agent。 | 交易完成后，AI 引擎不知道自己的决策在链上的真实表现。 |
