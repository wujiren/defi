# Phase 3 详细分解：虚拟实盘与混沌测试（修订版 v1.1）

**⏱️ 预计耗时**：3-4 周（联调与压测期）
**🎯 核心目标**：接入真实 RPC 实时流，激活水位线与预言机双重熔断；上线双分录账本并核算影子收益；在真实高并发下通过拔网线测试验证 FSM 的极致自愈能力。**同步实现策略异常路径（PAUSED / DEPRECATED 状态机）与非功能性指标专项验收。**

> **v1.1 修订说明：**
> - 新增任务 4：PAUSED / DEPRECATED 异常路径状态机（原计划完全缺失，是 LIVE 阶段策略自保的核心）
> - 新增任务 5：NFR 专项验收（非功能性量化指标——两档延迟、可用性 SLA、数据可复现性）
> - 任务 3 与 MtM 估值联动说明（依赖 Phase 2 任务 6）

---

## 任务 1：真实数据流摄取与水位线注入

**主导团队**：数据工程组
**协办团队**：AI 与量化策略组（Zone A）

废弃 Phase 1 的假数据推送脚本，全面接入真实区块链网络。

* **子任务 1.1：开发 `Chain-Adapter` 接入真实 RPC**
  * **工作内容**：使用 Go 编写服务，通过 WebSocket 订阅以太坊主网及 L2 的区块头与 `Event Log`（如 Uniswap Swap / Mint / Burn 事件）。
  * **硬性约束**：具备 RPC 断线自动重连（指数退避）和备用节点切换机制（主节点 100ms 无响应立即切换）；实现链重组（Reorg）事件感知，重组深度 ≤ 3 个区块时自动处理，> 3 个区块触发 P0 告警。

* **子任务 1.2：开发 `Data-Processor` 与水位线注入**
  * **工作内容**：接收 `Chain-Adapter` 的原始日志，进行协议语义解析（L1 原始数据 → L2 协议解析 → L3 聚合计算）；在每条消息中强制注入 `sync_block`（当前已同步区块高度）和 `sync_timestamp`（UTC 毫秒时间戳）。
  * **交付物**：标准化的 `ChainEvent` Protobuf 消息，高频推入 Redis Stream `stream:chain:events`，写入延迟 P99 < 50ms。

* **子任务 1.3：多链数据标准化对齐**
  * **工作内容**：统一时间锚点（将 L2 链事件映射到 `anchor_eth_block`）；映射 Canonical Token ID（主网地址 + chain_id 二元组），确保下游消费无感知跨链差异。
  * **数据连续性保障**：数据工程组负责确保 WebSocket 流不漏推、区块重组事件被正确标记和回滚。如发现漏推，第一责任人是数据工程组。

---

## 任务 2：硬熔断防线与预言机就绪

**主导团队**：财务风控组（Zone B）与区块链底层组

数据一旦是真实的，风控就必须是致命的。

* **子任务 2.1：激活 `Sync Sentinel` 水位线硬拦截**
  * **工作内容**：在 Go gRPC 拦截器中，对比当前链上最新区块与 Python 传来的 `ctx.last_sync_block`。
  * **硬性约束**：如果 `lag > watermark_max_lag`（默认 2 个区块），强制返回 `StaleDataError` 并拦截交易，触发 `CriticalWatermarkLag` P0 级 CRITICAL 告警（15 分钟内须响应）。
  * **熔断闭环**：`StaleDataError` 同时写入 Redis Stream `stream:agent:proposals`（供 AI 反馈飞轮学习），更新 `defi_watermark_lag_blocks` Prometheus Gauge。

* **子任务 2.2：开发 `Triple-Oracle Guard` 三源预言机**
  * **工作内容**：并行查询 Chainlink 喂价（HTTPS Pull）、Uniswap V3 TWAP（30 分钟时间加权）和当前 DEX Slot0 现货价格；取三源中位价作为资产估值基准（MtM NAV 计算唯一合法价格来源）。
  * **硬性约束**：任意两源价格偏差 > 0.5%（50 bps），立即抛出 `ORACLE_DEVIATION_TOO_HIGH` 并熔断交易，触发 `OraclePriceSkew` P0 告警；更新 `defi_oracle_deviation_bps` 指标。
  * **特别说明**：MtM NAV 计算**必须**使用三源中位价，禁止直接使用 Slot0 现货价（防止 MEV 操控价格影响风控决策）。

---

## 任务 3：影子状态机与对账基础

**主导团队**：区块链底层组（FSM）、财务风控组（对账）

* **子任务 3.1：实现影子记账引擎**
  * **工作内容**：当 FSM 推进至 `SIMULATED`（影子模式中视同最终状态），将预估 Token 变动转化为 `ledger_entries` 中的 Debit/Credit 分录；严格区分 `TRADE`（交易本金）、`GAS_FEE`（执行损耗，来自历史 Gas 价格估算）和 `PROTOCOL_FEE`（手续费）。
  * **与 MtM 联动**：每条影子分录写入后，通知 `MtMCalculator`（Phase 2 任务 6 交付）更新 NAV 快照；影子 PnL = 累计 NAV 变化量，数据展示在 Admin Web Portal 的策略总览页面。

* **子任务 3.2：实现跨链在途资金（Transit）追踪**
  * **工作内容**：利用 Saga 模式，发跨链交易时将资金记入 `BRIDGE_IN_TRANSIT` 账户；目标链到账时执行平账操作（Debit `BRIDGE_IN_TRANSIT`，Credit 目标链策略账户）。
  * **超时保障**：预留 30 分钟超时后自动触发告警并转入人工处理队列，Saga 回滚逻辑在 Phase 4 正式上线（本阶段在影子模式下只模拟，不真实跨链）。

* **子任务 3.3：对账第一责任人明确**
  * 任何对账告警（`diff != 0`），工单第一处理人为**财务风控组**。财务组须提供证据证明双分录借贷逻辑无误，再转给**数据工程组**排查 WebSocket 漏推或区块重组导致的底层事件缺失。

---

## 任务 4：PAUSED / DEPRECATED 异常路径状态机 *(新增)*

**主导团队**：区块链底层与执行组（Zone B）
**协办团队**：财务风控组（MtM 数据联动）、AI 与量化策略组（策略生命周期 API）
**依据**：`策略生命周期定义说明文档` §3（7 条自动触发规则）、§4（DEPRECATED 退役流程）

**为什么必须在 Phase 3 而非 Phase 4 实现？** LIVE 阶段的自动风控（MDD 超限自暂停、治理哨兵联动暂停）依赖 PAUSED 状态机，如果在 Phase 4 才实现，真金白银上线的 STAGED 阶段就缺乏异常路径保护。

### 4.1 PAUSED 状态：自动触发逻辑

实现以下 **7 条自动触发规则**（PRD 定义，任一触发即自动推进至 PAUSED）：

| 规则编号 | 触发条件 | 数据来源 |
|---|---|---|
| R1 | 实时 MDD > 回测基准 MDD × 1.5 | MtM NAV 序列（Phase 2 任务 6） |
| R2 | 连续 N 笔（配置项，默认 5）交易亏损 | `ledger_entries` 统计 |
| R3 | 毒性流检测：连续 3 次 `ToxicFlowDetected` | `4_2_资产估值与审计` 毒性流模块 |
| R4 | 对账差异：`diff != 0` 触发 `RECONCILIATION_FAILURE` | `Reconciler` 模块 |
| R5 | 水位线滞后超过 `pause_lag_threshold`（默认 5 区块，高于熔断阈值） | `Sync Sentinel` |
| R6 | 治理哨兵将依赖协议标记为 `HIGH_RISK` 或 `SUSPENDED` | Adapter Registry 状态变更事件 |
| R7 | G7 影子模式收益偏差 > 15%（仅在 SHADOWED → STAGED 审核期） | G7 验收报告 |

* **子任务 4.1：实现自动触发 PAUSED 逻辑**
  * **工作内容**：在 FSM 的每次状态推进前，轮询检查 R1-R7 触发条件；任一触发时，停止接受新的 `generate_proposals`，等待当前已进入 `SIGNED` 或 `BROADCASTED` 的任务自然完成（不强制中断），FSM 状态置为 `PAUSED`。
  * **原子性要求**：PAUSED 状态写入 `trade_tasks` 与 PAUSED 原因写入审计日志必须在同一数据库事务中完成。触发原因（`pause_reason`）须写入结构化字段，供 Admin Web Portal 展示。

* **子任务 4.2：实现 PAUSED → LIVE 人工确认恢复流程**
  * **工作内容**：策略进入 PAUSED 后，在 Admin Web Portal 显示暂停原因及关联 `trace_id`；管理员排查完毕后手动触发"恢复审批"，经管理员多签确认后 FSM 退回 STAGED 或 LIVE（依触发来源决定）。
  * **保护性约束**：R4（对账差异）触发的 PAUSED，须由财务风控组先关闭对应工单（证明账本逻辑无误）才允许提交恢复申请。

### 4.2 DEPRECATED 状态：自动退役逻辑

* **子任务 4.3：实现自动退役触发条件**
  * **工作内容**：实现以下退役条件（任一满足即自动推进至 DEPRECATED）：
    1. 协议适配器失效：Adapter Registry 中对应 `protocol_id` 被标记为 `SUSPENDED` 且超过 7 天未恢复
    2. 熔断超次：策略在 30 天内累计触发 PAUSED ≥ 5 次
    3. 手动退役：管理员在 Admin Web Portal 主动触发，需多签确认

* **子任务 4.4：实现退役资金归还与数据归档流程**
  * **工作内容**：策略进入 DEPRECATED 时：
    1. **资金归还**：停止所有新交易，等待已广播交易确认；通过 Adapter 的 `exit_all_positions()` 接口逐步退出所有持仓（顺序由协议优先级决定），将资金归还到资金池账户（写 `ledger_entries`）
    2. **数据归档**：将该策略所有 `trade_tasks`、`ledger_entries`、`mtm_snapshots`、G5/G6 评估报告压缩归档至冷存储（S3/MinIO），保留至少 2 年（合规要求）
    3. **AI 飞轮归档数据投递**：将退役策略的全量历史数据（含失败原因）写入 Redis Stream `stream:internal:accounting`，供 AI 反馈飞轮提取学习

---

## 任务 5：自愈状态机与混沌工程攻坚

**主导团队**：架构与 SRE 组（攻击方）vs 区块链底层组（防守方）

* **子任务 5.1：实现 `Recovery Manager` 自愈管理器**
  * **工作内容**：Go-Execution-Engine 启动时，阻塞扫描 PostgreSQL 中处于 `SIGNED` 和 `BROADCASTED` 状态的孤儿任务，根据链上 `tx_hash` 真实状态决定：
    - 链上已确认 → FSM 推进至 `CONFIRMED`，触发对账
    - 链上已回滚 → FSM 推进至 `REVERTED`，释放 Nonce 锁
    - 链上不存在 → 重新广播，实施阶梯 Gas 加速
  * **Nonce 锁释放**：进程重启后须原子释放所有属于该进程的 `nonce_locks` 行级锁，防止死锁。

* **子任务 5.2：执行持续混沌测试（Chaos Engineering）**
  * **工作内容**：SRE 编写混沌脚本，在 50+ Agent 疯狂发送交易期间随机执行：
    1. `kill -9` 杀掉 Go 进程（测试启动时自愈扫描）
    2. `iptables` 切断 Go 与 PostgreSQL 的连接（测试 WAL 日志保护）
    3. 重启 Redis Cluster 触发主从切换（测试 Stream 消费者组恢复）
    4. 停止 RPC 节点同步（测试水位线熔断精准度，须在 24 秒内触发 `StaleDataError`）
  * **验收标准**：1000 次随机中断后，`Recovery Manager` 每次在 **15 秒内**接管，0 笔交易重复发送，0 笔已签名交易丢失。

---

## 任务 6：NFR 专项验收 *(新增)*

**主导团队**：架构与 SRE 组
**依据**：`非功能性量化指标说明文档`

**为什么在 Phase 3 做 NFR 验收？** Phase 3 是第一个接入真实数据、运行真实高并发的阶段，此前在沙盒环境中的延迟测量不具备生产参考价值。NFR 验收必须在真实负载下进行。

* **子任务 6.1：端到端延迟两档验收**
  * **测试方法**：全链路 Trace ID 在每个阶段打时间戳（数据获取 → 策略计算 → EVM 仿真 → 签名请求 → 广播），数据写入 Prometheus 滚动 1 小时窗口计算 P50/P99。两档分别统计，不混合计算。
  * **通过标准**：
    - 轻量路径（常规交易）：P50 < 180ms，P99 < 510ms
    - 完整路径（大额/STAGED 交易）：P50 < 250ms，P99 < 680ms
    - 任一路径 P99 超过 800ms，记录为"异常执行"，须有对应告警触发

* **子任务 6.2：核心链路可用性 SLA 验收**
  * **测试方法**：统计 Phase 3 影子模式 72 小时运行期间，Go-Execution-Engine 和 Data Pipeline 的计划外停机时间；混沌工程导致的主动中断不计入可用性统计。
  * **通过标准**：核心执行链路（Go-Execution-Engine + Zone B 关键组件）可用性 ≥ 99.9%/月（月度换算：停机时间 ≤ 43.8 分钟）。

* **子任务 6.3：历史数据可复现性验收**
  * **测试方法**：取同一策略、同一时间段，使用相同随机数种子，执行两次独立回测，比较输出结果。
  * **通过标准**：两次回测产出的 `strategy_version_hash` + 数据集 SHA-256 哈希相同，所有交易的时间戳、金额、Gas 成本精确一致（差值 = 0）。

* **子任务 6.4：最大并发策略扩展性验收**
  * **测试方法**：启动 50 个 Python Agent 容器，每个运行独立策略，同时向 Go-Execution-Engine 发送交易。监控 CPU 使用率、内存使用量、数据库连接池饱和度。
  * **通过标准**：50 个策略并发时，系统稳定运行不崩溃，P99 延迟满足上述两档标准，数据库连接池使用率 < 80%（`max_connections = 200` 情况下，活跃连接 < 160）。

---

## 🏁 Phase 3 阶段合并与验收标准（DoD）

1. **影子模式持续运行（G7 Passed）**：AI 策略在真实市场数据下连续运行 **72 小时**不崩溃；产生的影子 PnL 与纯本地历史回测的假设收益偏差 **≤ 15%**（使用 MtM NAV 序列计算，非简单累加）。

2. **异常路径验收（新增）**：
   - R1-R7 全部触发规则经过单元测试覆盖，测试用例库可供审计
   - 手动触发 R1（构造 MDD 超限场景），FSM 在 2 个区块内自动推进至 PAUSED，Admin Web Portal 展示触发原因
   - 手动触发 DEPRECATED 流程，策略资金正确归还资金池，历史数据归档至冷存储路径验证可读

3. **零重复幂等验收（Chaos Passed）**：1000 次随机断网和 Kill 进程后，`Recovery Manager` 每次在 **15 秒内**接管；数据库最终对账结果：**0 笔交易被重复发送，0 笔已签名交易丢失（diff == 0）**。

4. **熔断精准度验收**：停止 RPC 节点同步，水位线在 **24 秒**（2 个以太坊区块）内精准触发 `StaleDataError`，P0 级电话告警成功拨出。

5. **NFR 延迟验收（新增）**：轻量路径 P99 < 510ms，完整路径 P99 < 680ms（均在 50 并发策略、真实市场数据下实测）。Grafana 仪表盘可展示两档延迟的独立分布曲线。

6. **可用性与可复现性验收（新增）**：72 小时影子运行期间核心链路可用性 ≥ 99.9%；同参数重复回测结果完全一致。
