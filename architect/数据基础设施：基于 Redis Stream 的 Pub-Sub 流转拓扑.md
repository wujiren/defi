## 数据基础设施：基于 Redis Stream 的 Pub/Sub 流转拓扑

### 1. 数据流转四层模型 (L1-L4 Layering)

根据《数据基础设施规范》，我们将数据流拆分为以下四个物理层级：

* **L1 - 原始接入层 (Raw Ingestion)**：
* **组件**：由 Go 编写的 **Chain-Adapter**。
* **职责**：通过 WebSocket/gRPC 订阅多节点 RPC 的 `newHeads` 和 `Logs`。
* **物理特性**：极轻量，仅负责解析 RLP 编码并打上接收时间戳。


* **L2 - 标准化与水位线层 (Normalization & Watermark)**：
* **组件**：Go **Data-Processor**。
* **职责**：将原始数据转化为标准的 Protobuf 格式，并注入 **模块 4.1** 定义的 `last_sync_block`。
* **写入**：将处理后的事件写入 Redis Stream 的指定 Topic。


* **L3 - 消息总线层 (Redis Stream Bus)**：
* **组件**：Redis 集群。
* **职责**：作为持久化消息中心，支持多消费者组（Consumer Groups）并发读取。


* **L4 - 消费与推理层 (Consumer Layer)**：
* **组件**：50+ 个 **Python Agent 实例**。
* **职责**：作为独立的消费者组订阅 L3。每个 Agent 维护自己的 `Pending_Entries`，确保消息不丢失且处理有序。



---

### 2. Redis Stream 拓扑结构设计

我们不使用传统的 Redis Pub/Sub（因为其不具备持久化能力），而是采用 **Redis Stream** 以保证交易指令的可靠性。

#### 2.1 Topic (Stream) 划分

* `stream:chain:events`：实时的 Block Header、Logs、Mempool 数据。
* `stream:internal:accounting`：**模块 4.2** 生成的双分录对账事件。
* `stream:agent:proposals`：Python Agent 发出的交易提议（用于监控与影子模式审计）。

#### 2.2 消费者组逻辑 (Consumer Groups)

* **并行化设计**：每个 Python Agent 属于一个独立的 `Consumer Group`。
* **负载均衡**：如果某个 Agent 需要横向扩展（如运行多个相同的套利实例），它们可以共享同一个 Group ID，Redis 会自动平衡消息分发。
* **ACK 机制**：Agent 处理完逻辑并成功向 Go 发送 `ExecuteTrade` 后，向 Redis 发送 `XACK`，确保每条数据都被有效利用。

---

### 3. 如何解决 PRD 中的关键痛点？

#### 3.1 极低延迟响应 (P99 < 500ms)

* **二进制传输**：在 Stream 中直接传输 Protobuf 字节流，避免了 JSON 解析的开销。
* **内存级速度**：Redis 作为内存数据库，其 `XADD/XREAD` 操作通常在微秒级，为后续的 **G2 仿真** 预留了充足的时间窗口。

#### 3.2 确定性水位线 (Sync Sentinel)

* **逻辑强制**：每一条流出的 L3 消息都包含 `sync_block` 标签。
* **自愈能力**：如果 Python Agent 的处理速度慢于区块产生速度，消息会在 Redis 中堆积。Agent 在读取消息时，若发现 `Current_Block - Message_Block > 2`，会主动触发 **StaleDataError** 并跳过该决策，防止基于过期状态发单。

#### 3.3 跨链原子性的数据支撑 (Saga 模式)

* **在途资金追踪**：当跨链操作发生时，Go 端向 `stream:internal:accounting` 写入一条“在途”记录。
* **超时触发**：独立的监控进程订阅此 Stream，若 30 分钟内未收到目标链的 `CONFIRMED` 事件，则自动触发 **Saga 回滚流程**。

---

### 4. 评价标准

| 维度 | **好架构 (Event-Driven)** | **坏架构 (Polling)** |
| --- | --- | --- |
| **扩展性** | 增加 100 个 Agent 只需增加 Redis 消费者组，不增加 RPC 负担。 | 每个 Agent 都去轮询 RPC，导致节点被 Rate Limit 封禁。 |
| **一致性** | 所有 Agent 看到的是同一套带水位线的时间序列数据。 | 不同 Agent 采集到的区块高度不一，导致对账混乱。 |
| **容错性** | Agent 重启后可以从 `PEL (Pending Entry List)` 恢复未处理完的消息。 | 重启后数据丢失，无法判断当前市场状态是否已发生变化。 |

