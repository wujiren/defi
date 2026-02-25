## 数据层：基于 Redis Stream 的多 Agent 高频推送架构

### 1. Go 执行层：高频数据推送逻辑 (The Producer)

Go 服务作为生产者，利用其原生的高并发能力（Goroutines）将链上数据实时压入 Redis Stream。

* **多源并行摄取 (Multi-Source Ingestion)**：Go 端通过多个 Goroutine 分别订阅不同 RPC 节点的 `NewHeads` 和 `Logs`，确保数据摄取的冗余与速度。
* **水位线打标 (Watermark Tagging)**：在每一条数据进入 Stream 之前，Go 逻辑强制注入当前已确认的区块号（`sync_block`）和摄取时间戳。
* **消息裁剪策略 (Memory Management)**：
* 使用 `XADD` 的 `MAXLEN ~ 10000` 参数，将 Stream 长度限制在最近 10000 条记录内。
* 由于 DeFi 交易的时效性极强，过旧的数据没有存储价值，此举可确保 Redis 内存始终处于健康水位。



---

### 2. Redis Stream：消息总线拓扑 (The Broker)

利用 Redis Stream 的 **Consumer Group (消费者组)** 机制，实现 50+ Agent 的高效订阅。

* **独立消费者组 (Isolated Consumer Groups)**：为每个 Python Agent 分配一个唯一的 `Group_ID`（如 `Agent_001_Group`）。
* **优点**：每个 Agent 拥有独立的 `Last_Delivered_ID` 指针，互不干扰。
* **容错**：若某个 Agent 宕机，重启后可以从其最后一次 `ACK` 的位置继续读取，不会丢失关键行情或交易反馈。


* **扇出模式 (Fan-out Distribution)**：一条链上事件进入 Stream 后，Redis 负责将其瞬间分发给所有活跃的消费者组，实现真正的 Pub/Sub 扇出效果。

---

### 3. Python 策略层：异步高频消费 (The Consumer)

Python Agent 采用 `asyncio` 结合 `redis-py` 实现非阻塞的高频消费逻辑。

* **异步拉取循环 (Async Read Loop)**：
* 使用 `XREADGROUP` 以阻塞模式读取消息。
* 设置 `BLOCK = 100`（100ms），在没有新数据时释放 CPU，降低功耗。


* **本地水位线熔断 (Stale Data Kill-switch)**：
* Agent 接收到数据后，立即对比 `current_time` 与消息中的 `ingest_timestamp`。
* 若延迟超过预设阈值（如 500ms），或者区块高度落后，则该条数据仅用于更新 K 线状态，禁止触发任何交易决策。


* **ACK 与可靠性闭环**：
* 策略计算完成后，Agent 必须向 Redis 发送 `XACK`。
* 通过监控 `XPENDING` 列表，系统可以实时感知哪些 Agent 处理数据过慢，从而动态调整其负载或发出性能预警。



---

### 4. 物理性能指标 (Quantitative Targets)

为了满足工业级要求，该架构需达到以下性能底线：

| 指标 | 目标值 (Target) | 决策因素 |
| --- | --- | --- |
| **分发延迟 (L2 -> L4)** | < 10ms | 确保 AI 推理逻辑有足够的响应窗口。 |
| **并发承载能力** | 100+ Consumers | 预留未来 2 倍的扩展空间。 |
| **消息丢失率** | 0% | 基于 Redis Stream 的持久化特性。 |
| **水位线漂移** | < 1 Block | 物理拦截过时决策的硬指标。 |

---

### 5. 架构演进：从单机 Redis 到 NATS/Kafka

虽然 Redis Stream 在 50+ Agent 规模下表现优异，但若未来资产管理规模进一步扩大（如达到千万美金以上且策略数超 200）：

* 建议将 L3 消息总线平滑迁移至 **NATS JetStream** 或 **Kafka**，以获得更强的吞吐能力和更精细的分区逻辑。
* **迁移方案**：由于 Go 端采用了接口抽象，届时仅需更换 `Data-Processor` 的驱动模块，无需修改 Python 端的业务策略。
