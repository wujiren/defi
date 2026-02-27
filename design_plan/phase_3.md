# Phase 3：虚拟实盘与混沌测试（修订版 v2.0）

**⏱️ 预计耗时**：3–4 周（联调与压测期）
**🎯 核心目标**：接入真实 RPC 实时流，激活水位线与预言机双重熔断；上线双分录账本并核算影子收益；在真实高并发下通过拔网线测试验证 FSM 的极致自愈能力。同步实现策略异常路径（PAUSED / DEPRECATED 状态机）与非功能性指标专项验收。

> **v2.0 修订说明（相较 v1.1）**
> - 新增：本阶段在整体架构中的位置图
> - 新增：各任务架构定位说明（📐）
> - 新增：各任务禁止事项（⛔）
> - 新增：各子任务输入/输出规范
> - 新增：任务依赖关系图
> - 新增：DoD 验收方法与责任人
> - 新增：外部依赖降级预案（Chainlink 长时间不更新、Snapshot/Tally API 限流）
> - 明确：MtM 联动说明（依赖 Phase 2 任务 6 交付）

> **全局参考**：接口所有权表、争议仲裁路径、环境隔离策略（DEV → STAGING 切换）、密钥证书生命周期（Phase 0 §C 约定的轮换窗口可能在本阶段到期，请提前确认）、灾难恢复预案均定义在 Phase 0 §A–§D，本阶段不重复。

---

## 📍 本阶段在整体架构中的位置

```
本阶段目标：废弃假数据，接入真实 RPC；激活水位线 + 预言机熔断；
           完成影子 FSM + 双分录影子对账；实现 PAUSED/DEPRECATED 异常路径；通过 NFR 验收

[Zone A]                      [Zone B]                              [Zone C]
──────────────────────        ──────────────────────────────────    ──────────────────
Python Agent(已有)            ★ Chain-Adapter(真实 RPC)              假签名(仍在用)
Admin Portal(已有)            ★ Data-Processor(水位线注入)
                              ★ Sync Sentinel(硬拦截激活)
                              ★ Triple-Oracle Guard(三源预言机)
                              ★ 影子 FSM(INIT→SIMULATED, 截断SIGNED)
                              ★ 双分录影子账本(影子 PnL 核算)
                              ★ PAUSED 自动触发(7条规则)
                              ★ DEPRECATED 退役流程
                              ★ Recovery Manager(15秒自愈)
                              ★ BRIDGE_IN_TRANSIT 跨链在途追踪
                              EVM 仿真器/MtM 估值(已有)
                              PostgreSQL / Redis(已有)

★ = 本 Phase 新增    括号 = 上阶段已有    — = 尚未存在（Phase 4 接入）
链上交互：真实 RPC 读取（影子模式不广播，Zone C 签名仍为 Mock）
环境切换：本阶段从 DEV → STAGING 环境（测试网），对应 Phase 0 §B.1
```

**本阶段不涉及**：真实链上广播、真实 MPC 签名、MEV 防夹路由、时间锁合约部署。FSM 在本阶段截断于 `SIMULATED`，`SIGNED` 之后的路径在 Phase 4 完成。

---

## ⛔ 本阶段全局禁止事项

```
数据流类：
  ⛔ 禁止 Chain-Adapter 使用 eth_call latest 标签（必须追踪最新区块 number 并显式传递）
  ⛔ 禁止 Data-Processor 省略 sync_block 字段注入（每条事件消息必须携带水位线）
  ⛔ 禁止水位线滞后超过 2 个区块时继续接受 TradeRequest（Sync Sentinel 硬拦截，不可绕过）
  ⛔ 禁止 MtM NAV 计算使用 DEX Slot0 即时价（必须三源预言机中位价）

影子模式类：
  ⛔ 禁止影子模式向任何链广播真实交易（FSM 必须截断于 SIMULATED，不允许推进至 SIGNED）
  ⛔ 禁止影子 PnL 计算不使用 MtM NAV 序列（不允许简单累加预估收益替代）

PAUSED/DEPRECATED 类：
  ⛔ 禁止 PAUSED 状态写入不记录触发原因（pause_reason 字段必须填写，不允许为 null）
  ⛔ 禁止 R4（对账差异）触发的 PAUSED 由非财务风控组关闭工单（必须财务组举证账本无误）
  ⛔ 禁止 DEPRECATED 流程跳过资金归还步骤（必须 exit_all_positions() 执行完成后才归档）

混沌测试类：
  ⛔ 禁止混沌测试在 PROD 环境执行（必须在 STAGING 环境，对应 §B 环境隔离）
  ⛔ 禁止混沌测试导致的中断计入 SLA 不可用时间统计（主动中断须在测试报告中标注排除）
```

---

## 外部依赖降级预案

> **为什么在 Phase 3 专节处理？** Phase 3 是第一个接入真实外部服务的阶段，Chainlink 延迟更新和 Snapshot/Tally 限流是两个最可能静默失效的风险点，必须在影子模式阶段就完成压力测试和降级验证。

### 降级预案 A：Chainlink 预言机长时间不更新

**风险描述**：Chainlink 喂价更新条件为"价格偏差 > 0.5% 或超过 1 小时"。在低波动率市场中，Chainlink 可能长达数小时不更新，而此时 Uniswap TWAP 和 DEX Slot0 已反映最新价格，三源偏差将持续触发熔断，策略被迫停止发单。

**降级策略**：

```
触发条件：Chainlink 同一喂价合约 > 2 小时未更新（defi_chainlink_last_update_age_seconds > 7200）

处理流程：
  Level 1（2–4 小时未更新）：
    → 将 Chainlink 权重降为"参考"，允许以 Uniswap TWAP + DEX Slot0 双源中位价作为估值基准
    → 偏差熔断阈值放宽至 1%（100 bps）
    → 写 WARNING 日志，触发 P1 告警通知财务风控组确认

  Level 2（> 4 小时未更新）：
    → 新建仓操作暂停（已有头寸保持），避免在价格基准不可靠时开新仓
    → 触发 P0 告警，通知 On-Call 工程师人工评估是否暂停全部策略
    → 记录事件至 oracle_degradation_log 表（含开始时间、持续时长、受影响策略数）

  恢复条件：Chainlink 再次更新后，熔断阈值自动恢复至 50 bps，恢复新建仓
```

**Prometheus 监控**：`defi_chainlink_last_update_age_seconds{pair}` Gauge，告警规则：`> 3600` 触发 P1，`> 14400` 触发 P0。

---

### 降级预案 B：Snapshot / Tally API 限流或响应异常

**风险描述**：治理哨兵每 5 分钟轮询 Snapshot.org 和 Tally.xyz API。若被限流（HTTP 429）或 API 架构变更导致解析失败，哨兵将静默停止更新，HIGH_RISK / SUSPENDED 状态变更无法及时触发，Adapter Registry 状态将出现滞后——这是一个无声的安全漏洞。

**降级策略**（在 Phase 3 验证，Phase 4 治理哨兵上线时实施）：

```
检测机制：
  → 记录每次 API 调用结果：defi_governance_api_poll_success{source="snapshot|tally"} Counter
  → 连续 3 次 API 失败（含 429/5xx/解析错误）触发 GoveranceAPIFailure P1 告警
  → API 静默超过 15 分钟（成功计数不增长）触发 GoveranceAPISilent P0 告警

降级行为（API 失败期间）：
  → 链上 Governor 合约事件（实时每区块订阅）继续正常运行，作为兜底数据源
  → Snapshot/Tally 失败不自动恢复协议状态（状态只降不升），防止因 API 恢复延迟导致
    HIGH_RISK 协议被误提前恢复
  → 在 Admin Web Portal 展示治理数据源健康状态（绿/黄/红），管理员可感知

恢复后补偿：
  → API 恢复后，立即执行全量历史提案补扫（补扫时间范围 = 最近一次成功轮询至当前）
  → 若补扫发现有 CRITICAL/HIGH 提案被漏处理，立即执行对应的 Adapter Registry 状态变更
  → 补扫结果写审计日志，通知安全组和财务风控组确认
```

---

## 任务 1：真实数据流摄取与水位线注入

### 📐 架构定位

| 项目 | 内容 |
|---|---|
| **架构位置** | Zone B 数据层，废弃 Phase 0 假数据脚本，接入真实以太坊主网/L2 |
| **对应架构文档** | `2.2_Redis_Stream实时管道.md` §2 消息结构；`2.3_多链数据统一规范.md` §3 Canonical ID |
| **在请求链路中的位置** | 以太坊节点 WebSocket → **Chain-Adapter** → **Data-Processor（注入水位线）** → Redis Stream `stream:chain:events` → Python Agent 消费 |
| **上游依赖** | Phase 0 Redis Cluster、Phase 1 `ChainAdapterInterface` 抽象基类 |
| **下游影响** | 阻塞任务 2（Sync Sentinel 需要真实 sync_block 才能拦截）；为 Python Agent 提供实时数据 |
| **接口 Owner** | 数据工程组 |

### ⛔ 本任务禁止事项

```
⛔ 禁止 Chain-Adapter 漏推链重组（Reorg）事件（必须检测并推送 reorg 类型事件，供下游回滚处理）
⛔ 禁止 Data-Processor 省略 sync_block 字段（每条 ChainEvent 必须携带，哪怕来源延迟）
⛔ 禁止主 RPC 节点 100ms 无响应时继续等待（必须立即切换至备用节点）
⛔ 禁止写入 Redis Stream 时超过 P99 50ms 延迟（须在 Prometheus 中实时监控）
```

---

### 子任务 1.1：开发 Chain-Adapter 接入真实 RPC

**输入：**
- `2.2_Redis_Stream实时管道.md` §2 消息结构与推送规范
- 以太坊主网及 L2（Arbitrum/Optimism）RPC 端点配置（STAGING 环境）
- Phase 1 `ChainAdapterInterface` 抽象基类

**输出：**
- `services/chain_adapter/main.go`：通过 WebSocket 订阅区块头 + Event Log（Uniswap Swap / Mint / Burn）
- 断线重连：指数退避（初始 1s，上限 30s），重连成功后从最后已知区块补推漏失事件
- 备用节点切换：主节点 100ms 无响应立即切换，切换事件写审计日志
- 链重组检测：比较新区块的 parentHash 与缓存，检测重组；重组深度 ≤ 3 个区块自动处理（推送 reorg 事件），> 3 个区块触发 P0 告警（对应 Phase 0 §D.3 预案）
- Prometheus：`defi_rpc_request_duration_ms{node="primary|fallback"}`、`defi_reorg_depth{chain_id}`

**不产出（明确排除）：**
- 不包含 L3/L4 数据聚合（由 Data-Processor 负责）
- 不包含历史数据回填（由 Phase 1 Archiver 负责）

---

### 子任务 1.2：开发 Data-Processor 与水位线注入

**输入：**
- Chain-Adapter 产出的原始 Event Log
- `2.1_Protobuf契约.md` `ChainEvent` 消息格式（含 `sync_block` 字段）

**输出：**
- `services/data_processor/main.go`：L1 → L2 协议语义解析（Swap 解析为 price/amount_in/amount_out，Mint/Burn 解析为流动性变化）
- 每条消息强制注入：
  - `sync_block`：当前已完成解析的最新区块高度
  - `sync_timestamp`：UTC 毫秒时间戳
- 写入 Redis Stream `stream:chain:events`，P99 写入延迟 < 50ms
- Prometheus：`defi_watermark_lag_blocks`（当前链头 - sync_block）持续监控

**不产出（明确排除）：**
- 不包含 L3 OHLCV 聚合（本阶段仅 L1→L2，L3 可按需扩展）

---

### 子任务 1.3：多链数据标准化对齐

**输入：**
- `2.3_多链数据统一规范.md` §3 Canonical Token ID 规范

**输出：**
- Canonical Token ID 映射表（主网地址 + chain_id 二元组 → 统一 token 标识符），写入 PostgreSQL `canonical_tokens` 表
- L2 链事件时间锚点映射（L2 区块 → `anchor_eth_block`，确保跨链时间基准一致）
- 单元测试：同一资产不同链的地址映射到相同 Canonical ID

---

## 任务 2：硬熔断防线与预言机就绪

### 📐 架构定位

| 项目 | 内容 |
|---|---|
| **架构位置** | Zone B 风控层，数据进入策略决策前的最后一道安全阀 |
| **对应架构文档** | `3.2_验证层_EVM仿真与三源预言机.md` §3 预言机校验；`6.3_告警规则集.md` CriticalWatermarkLag / OraclePriceSkew |
| **在请求链路中的位置** | TradeRequest → **Sync Sentinel（水位线校验）** → **Triple-Oracle Guard（价格校验）** → 仿真器 |
| **上游依赖** | 任务 1（真实 sync_block 数据）；Phase 2 MtM 估值（三源价格已使用，本任务激活真实版） |
| **下游影响** | 阻塞任务 3（影子模式必须在熔断到位后才能安全接入真实数据） |
| **接口 Owner** | 财务风控组（Zone B）与区块链底层组 |

### ⛔ 本任务禁止事项

```
⛔ 禁止 Sync Sentinel 作为软拦截（必须是硬拦截：lag > 2 则强制返回 StaleDataError，无例外）
⛔ 禁止预言机熔断阈值可被策略代码修改（0.5% 偏差上限为全局配置，仅管理员通过时间锁修改）
⛔ 禁止 MtM NAV 计算在 Chainlink 降级期间使用未经审核的价格组合（须遵循降级预案 A 中的策略）
⛔ 禁止三源预言机中任意一源失效后继续以双源均值估值（必须触发告警，等待 On-Call 决策）
```

---

### 子任务 2.1：激活 Sync Sentinel 水位线硬拦截

**输入：**
- `3.3_幂等状态机FSM.md` §4 水位线校验逻辑
- 任务 1 产出的 `sync_block` 水位线数据
- `watermark_max_lag` 配置项（默认值：2 个区块）

**输出：**
- `pkg/middleware/sync_sentinel.go`：gRPC 拦截器，对比 `chain_head_block - ctx.last_sync_block`
- 拦截逻辑：`lag > watermark_max_lag` 时强制返回 `StaleDataError`（error_code: `STALE_DATA_ERROR`），不进入仿真器
- 副作用写入：`StaleDataError` 同时写 Redis Stream `stream:agent:proposals`（供 AI 反馈飞轮学习）
- 指标：`defi_watermark_lag_blocks` Gauge 实时更新
- 告警：`CriticalWatermarkLag`（lag > 2 且持续 0s，即时触发 P0 CRITICAL）

**不产出（明确排除）：**
- 不包含水位线的软拦截模式（本项目只有硬拦截）

---

### 子任务 2.2：开发 Triple-Oracle Guard 三源预言机

**输入：**
- `3.2_验证层_EVM仿真与三源预言机.md` §3 三源价格校验逻辑
- Chainlink 喂价合约地址（Adapter Registry 中注册）
- Uniswap V3 TWAP 接口（30 分钟时间加权）
- DEX Slot0 现货接口

**输出：**
- `pkg/oracle/triple_oracle.go`：并行查询三源，取中位价
- 偏差校验：任意两源价格偏差 > 0.5%（50 bps）立即返回 `ORACLE_DEVIATION_TOO_HIGH`，熔断交易
- 价格数据唯一合法来源：MtM NAV 计算使用三源中位价，禁止 Slot0 即时价
- 指标：`defi_oracle_deviation_bps{pair}` Gauge；告警：`OraclePriceSkew`（偏差 > 50 bps，P0 即时触发）
- **Chainlink 降级联动**：接入降级预案 A 的逻辑（`defi_chainlink_last_update_age_seconds` 监控 + Level 1/2 处理流程）

---

## 任务 3：影子状态机与对账基础

### 📐 架构定位

| 项目 | 内容 |
|---|---|
| **架构位置** | Zone B，Phase 2 FSM 的扩展，接入真实数据后完整演练影子路径 |
| **对应架构文档** | `3.3_幂等状态机FSM.md` §3 影子模式；`4.1_双分录会计系统.md` §3 标准分录规则 |
| **在请求链路中的位置** | Python Agent（影子模式）→ `ShadowTrade` gRPC → FSM（INIT → SIMULATED，截断 SIGNED）→ 影子分录写入 → MtM NAV 更新 |
| **上游依赖** | 任务 1（真实事件流）、任务 2（熔断到位）、Phase 2 任务 4（双分录账本）、Phase 2 任务 6（MtM 估值） |
| **下游影响** | 影子 PnL 数据为 G7 验收提供依据；阻塞 Phase 4 STAGED 阶段真实发单 |
| **接口 Owner** | 区块链底层组（FSM）；财务风控组（对账） |

### ⛔ 本任务禁止事项

```
⛔ 禁止影子模式 FSM 推进至 SIGNED 状态（截断必须在代码层面硬限制，不能仅靠配置控制）
⛔ 禁止影子 PnL 使用简单累加（必须基于 MtM NAV 序列计算，Phase 2 任务 6 交付的 MtMCalculator）
⛔ 禁止对账告警（diff != 0）由非财务风控组关闭（第一处理人规则，见 Phase 0 §A.2）
```

---

### 子任务 3.1：实现影子记账引擎

**输入：**
- Phase 2 任务 4 产出的 `LedgerEngine`
- Phase 2 任务 6 产出的 `MtMCalculator`
- FSM SIMULATED 状态输出的仿真执行收据

**输出：**
- 影子分录写入：FSM 推进至 SIMULATED 后，触发 `LedgerEngine.WriteEntry()` 写影子分录（标注 `is_shadow=true`）
- 分录类型：`TRADE`（交易本金）、`GAS_FEE`（历史 Gas 估算值）、`PROTOCOL_FEE`（手续费），三类在同一事务中原子写入
- 分录写入后通知 `MtMCalculator` 更新 NAV 快照（event-driven，不轮询）
- 影子 PnL 计算：累计 NAV 变化量，展示于 Admin Web Portal 策略总览页（接入 Phase 2 任务 5 的页面）

---

### 子任务 3.2：实现跨链在途资金（Transit）追踪

**输入：**
- `4.1_双分录会计系统.md` §4 跨链在途处理规范
- Phase 1 Adapter Registry（跨链桥协议白名单）

**输出：**
- 跨链发起时：资金记入 `BRIDGE_IN_TRANSIT` 虚拟账户（影子模式仅模拟，不真实发桥接交易）
- 目标链到账（模拟）时：平账操作（Debit `BRIDGE_IN_TRANSIT`，Credit 目标链策略账户）
- 30 分钟超时后触发 P1 告警，转入人工处理队列（Saga 回滚在 Phase 4 真实跨链时实施）

---

### 子任务 3.3：对账第一责任人明确与验证

**输入：**
- Phase 2 任务 4 产出的 `Reconciler`

**输出：**
- 影子模式下启用全量对账（每小时 + 每次 SIMULATED 后）
- 对账告警触发时自动在 Jira/工单系统创建工单，指派给财务风控组（默认 assignee = 财务组 Tech Lead）
- 财务组关闭工单须填写"账本逻辑验证结论"字段，关闭后工单系统自动通知数据工程组排查底层事件

---

## 任务 4：PAUSED / DEPRECATED 异常路径状态机

### 📐 架构定位

| 项目 | 内容 |
|---|---|
| **架构位置** | Zone B FSM 异常路径，策略风控的最后一道自动防线 |
| **对应架构文档** | `策略生命周期定义说明文档.md` §3（7 条自动触发规则）、§4（DEPRECATED 退役流程） |
| **在请求链路中的位置** | 任一 R1-R7 触发条件满足 → FSM 推进至 **PAUSED** → Admin Portal 展示 → 人工多签确认 → 恢复；或累计触发 ≥ 5 次 → **DEPRECATED** → 资金归还 → 数据归档 |
| **上游依赖** | Phase 2 任务 6（MtM 实时 MDD，R1 规则上游）；Phase 2 任务 4（Reconciler，R4 规则上游）；任务 2（Sync Sentinel，R5 规则上游）；Phase 1 Adapter Registry（R6 规则上游） |
| **下游影响** | Phase 4 LIVE 阶段真实资金的安全阀；DEPRECATED 数据归档是 AI 飞轮负样本的来源 |
| **接口 Owner** | 区块链底层与执行组 |

> **为什么必须在 Phase 3 而非 Phase 4 实现？** LIVE 阶段的真实资金自动风控（MDD 超限、治理哨兵联动）依赖 PAUSED 状态机。若 Phase 4 才实现，STAGED 阶段已有真实小额资金，将缺乏异常路径保护。

### ⛔ 本任务禁止事项

```
⛔ 禁止 PAUSED 状态变更与触发原因记录不在同一数据库事务（必须原子提交）
⛔ 禁止 R4（对账差异）触发的 PAUSED 由非财务风控组提交恢复申请（须财务组先关闭对账工单）
⛔ 禁止 DEPRECATED 流程跳过 exit_all_positions() 直接归档（必须确认所有头寸已退出）
⛔ 禁止强制中断已处于 SIGNED 或 BROADCASTED 状态的任务（PAUSED 触发后等待其自然完成）
```

---

### 子任务 4.1：实现 PAUSED 自动触发逻辑（7 条规则）

**输入：**
- `策略生命周期定义说明文档.md` §3 全部 7 条触发规则
- Phase 2 MtM / Reconciler / Sync Sentinel 各模块数据接口

**输出：**
- `pkg/fsm/pause_trigger.go`：FSM 每次状态推进前轮询检查以下 7 条规则：

| 规则 | 触发条件 | 数据来源 |
|---|---|---|
| R1 | 实时 MDD > 回测基准 MDD × 1.5 | `MtMCalculator.CalcRealtimeMDD()` |
| R2 | 连续 N 笔（默认 5）交易亏损 | `ledger_entries` 统计 |
| R3 | 连续 3 次 `ToxicFlowDetected`（30 分钟内）| 毒性流模块（Phase 4 完整实现，本阶段影子模式下预留接口） |
| R4 | 对账差异 `diff != 0`（`RECONCILIATION_FAILURE`）| `Reconciler` |
| R5 | 水位线滞后 > `pause_lag_threshold`（默认 5 区块）| `Sync Sentinel` |
| R6 | 依赖协议被 Adapter Registry 标记为 `HIGH_RISK` 或 `SUSPENDED` | Registry 状态变更事件 |
| R7 | G7 影子收益偏差 > 15%（仅在 SHADOWED → STAGED 审核期）| G7 验收报告 |

- 触发任一规则：停止接受新 `generate_proposals`，等待已 SIGNED/BROADCASTED 任务自然完成，FSM 置为 PAUSED
- 原子写入：PAUSED 状态 + `pause_reason`（规则编号 + 触发值）同一事务提交
- 告警：P0 CRITICAL 告警，Admin Web Portal 显示暂停原因和关联 trace_id

---

### 子任务 4.2：实现 PAUSED → LIVE 人工确认恢复流程

**输入：**
- Phase 2 Admin Web Portal（任务 5）`Admin-Approval-API`

**输出：**
- Admin Web Portal 暂停详情页：显示触发规则、当前指标值、建议处理步骤
- 管理员提交"恢复申请"须填写：根因分析说明 + 对应修复措施
- R4 触发的 PAUSED：额外要求财务工单状态为"已关闭"才允许提交恢复申请（API 层校验）
- 多签确认（2-of-N 管理员签名）后，FSM 退回 STAGED 或 LIVE（依触发规则来源决定）
- 恢复操作写完整审计日志

---

### 子任务 4.3：实现 DEPRECATED 自动退役触发

**输入：**
- `策略生命周期定义说明文档.md` §4 退役条件

**输出：**
- `pkg/fsm/deprecate_trigger.go`：监控以下三条退役条件（任一满足自动推进 DEPRECATED）：
  1. Adapter Registry 对应 `protocol_id` 状态为 `SUSPENDED` 且超过 7 天未恢复
  2. 策略在 30 天内累计触发 PAUSED ≥ 5 次（`pause_history` 表统计）
  3. 管理员手动退役（Admin Portal 多签确认）

---

### 子任务 4.4：实现退役资金归还与数据归档流程

**输入：**
- Phase 1 Adapter Registry（提供 `exit_all_positions()` 接口签名）
- `4.1_双分录会计系统.md` §5 资金归还分录规则

**输出：**
- 退役执行序列（严格顺序）：
  1. 停止所有新 TradeRequest
  2. 等待已广播交易链上确认（最大等待时间：30 分钟）
  3. 通过 Adapter `exit_all_positions()` 逐步退出全部头寸（影子模式模拟，Phase 4 真实执行）
  4. 写资金归还分录：`Credit 策略账户 → Debit 资金池账户`
  5. 压缩归档至冷存储（S3/MinIO）：`trade_tasks`、`ledger_entries`、`mtm_snapshots`、G5/G6 报告，保留 2 年
  6. 写入 Redis Stream `stream:internal:accounting`（AI 飞轮负样本投递）
- 归档完成后 Admin Portal 策略状态更新为 DEPRECATED，不可恢复

---

## 任务 5：自愈状态机与混沌工程攻坚

### 📐 架构定位

| 项目 | 内容 |
|---|---|
| **架构位置** | Zone B FSM 的容错层，应对进程崩溃、网络中断、数据库超时等异常场景 |
| **对应架构文档** | `3.3_幂等状态机FSM.md` §5 Recovery Manager；`8.1_Go仿真器参考实现.md` §6 异常处理 |
| **在请求链路中的位置** | 进程重启 → **Recovery Manager（扫描孤儿任务）** → 按链上实际状态修复 FSM |
| **上游依赖** | Phase 2 任务 3（FSM + Nonce 锁基础）；任务 1（真实链上数据） |
| **下游影响** | Phase 4 STAGED/LIVE 阶段真实资金的进程安全保障 |
| **接口 Owner** | 区块链底层与执行组（防守方）；架构与 SRE 组（攻击方） |

### ⛔ 本任务禁止事项

```
⛔ 禁止 Recovery Manager 修复孤儿任务时不写审计日志（每次修复须记录原状态、新状态、链上依据）
⛔ 禁止混沌测试中断后不验证 diff == 0（每次中断恢复后必须执行 Reconciler 全量对账）
⛔ 禁止混沌测试对同一 trace_id 触发两次广播（Recovery Manager 必须通过 Nonce 锁保证幂等）
```

---

### 子任务 5.1：实现 Recovery Manager 自愈管理器

**输入：**
- `trade_tasks` 表中 `status IN ('SIGNED', 'BROADCASTED')` 的孤儿任务
- 链上 `eth_getTransactionReceipt(tx_hash)` 查询

**输出：**
- `pkg/recovery/manager.go`：服务启动时阻塞扫描孤儿任务，按链上实际状态决策：
  - 链上已确认（receipt.status == 1）→ FSM 推进至 CONFIRMED，触发对账
  - 链上已回滚（receipt.status == 0）→ FSM 推进至 REVERTED，释放 Nonce 锁
  - 链上不存在（receipt == null）→ 重新广播，启动阶梯 Gas 加速（Phase 4 完整实现，本阶段仅影子模式模拟）
- Nonce 锁原子释放：进程重启后释放所有属于该进程 instance_id 的 `nonce_locks` 行
- 每次修复写审计日志：`trace_id`、`original_status`、`recovered_status`、`chain_evidence`

**不产出（明确排除）：**
- 不包含真实重广播逻辑（Phase 4 实现，本阶段影子模式下只修复 FSM 状态不真实广播）

---

### 子任务 5.2：执行持续混沌工程测试

**输入：**
- STAGING 环境（50+ Agent 并发运行的影子模式）
- `非功能性量化指标说明文档.md` Recovery 目标（15 秒内接管，0 笔重复，0 笔丢失）

**输出：**
- 混沌脚本 `scripts/chaos_test.sh`，在 50+ Agent 发单期间随机注入以下 4 类中断：
  1. `kill -9 go-execution-engine`（测试启动时 Recovery Manager 自愈扫描）
  2. `iptables DROP` 切断 Go ↔ PostgreSQL 连接（测试 WAL 日志保护）
  3. 重启 Redis Cluster 触发主从切换（测试 Stream Consumer Group 重放）
  4. 停止 RPC 节点同步（测试水位线熔断：须在 24 秒内触发 `StaleDataError`）
- 通过标准：1000 次随机中断，每次 Recovery Manager 在 **15 秒内**接管，`conflict_count == 0`，`lost_count == 0`
- 每次中断后自动运行 Reconciler 全量对账，`diff == 0` 才视为该轮通过
- 测试报告：含中断类型分布、最大恢复时间、中位恢复时间

---

## 任务 6：NFR 专项验收

### 📐 架构定位

| 项目 | 内容 |
|---|---|
| **架构位置** | 横切关注点，在真实负载（STAGING + 50 并发）下验证系统级非功能性指标 |
| **对应架构文档** | `非功能性量化指标说明文档.md` 全文；`1.2_容器资源配额.md` 性能预算 |
| **在请求链路中的位置** | 全链路 Prometheus Trace → Grafana 延迟仪表盘 → SLA 统计 → 可复现性验证 |
| **上游依赖** | 所有 Phase 3 任务完成后，在真实负载下采集数据 |
| **下游影响** | NFR 验收通过是 Phase 4 申请真实资金拨测的前置条件 |
| **接口 Owner** | 架构与 SRE 组（主导） |

### ⛔ 本任务禁止事项

```
⛔ 禁止使用 DEV 或沙盒环境的延迟数据代替 STAGING 真实负载数据（NFR 验收必须在 STAGING 下进行）
⛔ 禁止混沌测试导致的主动中断计入 SLA 不可用时间（须在测试报告中显式排除并注明时间段）
⛔ 禁止两档延迟指标混合统计（Grafana 必须展示独立的 lite/full 两条延迟分布曲线）
```

---

### 子任务 6.1：端到端延迟两档验收

**输入：**
- `非功能性量化指标说明文档.md` 两档延迟预算
- Prometheus `defi_fsm_transition_duration_ms{mode="lite|full"}` 数据

**输出：**
- 测试方法：全链路每个阶段用 Trace ID 打时间戳（数据获取 → 策略计算 → EVM 仿真 → 签名请求），滚动 1 小时窗口统计 P50/P99
- 通过标准：

| 档位 | P50 | P99 | 超限告警阈值 |
|---|---|---|---|
| 轻量路径（常规交易）| < 180ms | < 510ms | P99 > 800ms 记为异常 |
| 完整路径（大额/STAGED）| < 250ms | < 680ms | P99 > 800ms 记为异常 |

- Grafana 仪表盘：两档延迟独立曲线，不混合计算；`defi_fsm_transition_duration_ms` 按 `mode` label 分组

---

### 子任务 6.2：核心链路可用性 SLA 验收

**输入：**
- Phase 3 影子模式 72 小时运行期间的可用性数据

**输出：**
- 统计方法：监控 `go-execution-engine` 和 `Data Pipeline` 计划外停机时间（混沌工程主动中断不计入）
- 通过标准：核心执行链路可用性 ≥ 99.9%（等价月度停机 ≤ 43.8 分钟）
- 报告：含计划外停机事件列表（时间、时长、原因、是否触发告警）

---

### 子任务 6.3：历史数据可复现性验收

**输入：**
- Phase 1 `IDataProvider` + `dataset_sha256` 机制

**输出：**
- 测试方法：同一策略、同一时间段、相同随机数种子，执行两次独立回测
- 通过标准：两次回测 `strategy_version_hash` + `dataset_sha256` 相同，所有交易时间戳/金额/Gas 精确一致（差值 = 0）
- 自动化验证脚本：`scripts/verify_reproducibility.sh`

---

### 子任务 6.4：最大并发策略扩展性验收

**输入：**
- `1.2_容器资源配额.md` §3 性能指标（max_connections = 200）

**输出：**
- 测试方法：启动 50 个 Python Agent 容器，每个运行独立策略，同时向 Go-Execution-Engine 发单
- 通过标准：稳定运行不崩溃，P99 满足两档标准，数据库连接池使用率 < 80%（活跃连接 < 160/200）
- 监控指标：CPU 使用率（目标 < 80%）、内存使用量、`defi_db_pool_active_connections` Gauge

---

## 任务依赖关系

```
任务 1（Chain-Adapter + Data-Processor）
  └─ 阻塞 → 任务 2（Sync Sentinel 需要真实 sync_block 才能激活）
  └─ 阻塞 → 任务 3（影子模式需要真实事件流驱动）

任务 2（Sync Sentinel + Triple-Oracle Guard）
  └─ 阻塞 → 任务 3（熔断防线必须先于影子模式就位，防止数据不新鲜时的影子 PnL 失真）

任务 3（影子 FSM + 对账）
  ├─ 阻塞 → 任务 4（PAUSED 触发后影子对账数据须可用）
  └─ 阻塞 → 任务 6（NFR 验收需要影子模式稳定运行 72 小时）

Phase 2 任务 6（MtM 估值，已交付）
  └─ 是任务 4 的上游依赖（R1 规则依赖实时 MDD 数据）

任务 4（PAUSED / DEPRECATED 状态机）
  └─ 阻塞 → Phase 4 真实资金安全（必须在 STAGED 前就位）

任务 5（Recovery Manager + 混沌工程）
  └─ 可与任务 3、4 并行启动
  └─ 阻塞 → Phase 4 生产就绪评审（混沌测试必须通过）

任务 6（NFR 专项验收）
  └─ 依赖任务 1-5 全部完成后，在真实负载下采集数据
  └─ 阻塞 → Phase 4 申请真实资金

可并行执行：任务 1 + 任务 5（Recovery Manager 基础实现可提前）
关键路径：任务 1 → 任务 2 → 任务 3 → 任务 6（G7 验收的 72 小时最长等待）
```

---

## 🏁 Phase 3 阶段验收标准（DoD）

| # | 验收标准 | 验收方法 | 验收责任人 |
|---|---|---|---|
| 1 | **影子模式 G7 通过**：真实数据下 72 小时稳定，影子 PnL 偏差 ≤ 15% | 在 STAGING 环境运行策略 72 小时，Admin Portal 读取影子 PnL 与 G5 回测假设收益对比，偏差值写入 `gate_results`（`gate_id='G7'`） | AI 策略组 + 财务风控组 |
| 2 | **PAUSED 自动触发可演示**：R1-R7 全部通过单元测试覆盖 | R1-R7 各写一条单元测试（覆盖边界值）；现场演示：构造 MDD 超限场景，FSM 在 2 个区块内自动置为 PAUSED，Admin Portal 显示 `pause_reason: R1` | 区块链底层组 Tech Lead |
| 3 | **DEPRECATED 流程可演示**：资金归还 + 数据归档路径完整 | 手动触发一次 DEPRECATED，验证：影子头寸正确归还资金池账本；归档文件出现在 MinIO 冷存储路径；`stream:internal:accounting` 收到退役策略数据投递 | 区块链底层组 + 财务风控组 |
| 4 | **零重复幂等验收（混沌通过）**：1000 次随机中断后 Recovery Manager 15 秒内接管 | 运行 `scripts/chaos_test.sh --rounds 1000`，最终报告 `conflict_count == 0`，`lost_count == 0`，`max_recovery_time < 15s` | 架构与 SRE 组 + 区块链底层组 |
| 5 | **熔断精准度验收**：水位线 2 区块内精准触发 | 停止 RPC 节点同步，确认 24 秒（约 2 个以太坊区块）内 `StaleDataError` 触发，Alertmanager 发出 `CriticalWatermarkLag` P0 告警（测试 PagerDuty/Slack 收到） | 财务风控组 + 架构与 SRE 组 |
| 6 | **NFR 延迟验收**：两档 P99 达标 | Grafana 展示两档独立延迟曲线，轻量 P99 < 510ms，完整 P99 < 680ms（50 并发真实负载下） | 架构与 SRE 组 |
| 7 | **可用性与可复现性验收** | 72 小时影子运行可用性统计 ≥ 99.9%（排除主动混沌中断）；`scripts/verify_reproducibility.sh` 输出 `PASS`，两次回测 `dataset_sha256` 完全一致 | 架构与 SRE 组 |
| 8 | **Chainlink 降级逻辑验证** | 模拟 Chainlink 2 小时不更新（停止 mock Chainlink 喂价更新），确认：`defi_chainlink_last_update_age_seconds > 7200` 触发 P1 告警；系统自动切换为 TWAP + Slot0 双源定价；新建仓未受影响（Level 1 行为） | 财务风控组 + 架构与 SRE 组 |
| 9 | **全链路可观测验收** | 取任意一笔影子模式的 trace_id，在 Grafana Loki 中检索出完整 5 节点日志（Agent 发单 → gRPC → DB → 影子签名 → 影子 Event Indexer 确认） | 架构与 SRE 组 |
