# Phase 2：核心引擎与数学对齐（修订版 v2.0）

**⏱️ 预计耗时**：3–4 周（硬核攻坚期）
**🎯 核心目标**：消除跨语言浮点数与智能合约整数运算的误差，跑通核心状态机（FSM）前半段，完成 G1-G4 全部安全门控，并在此基础上正式通过 G5/G6 绩效门控。同步交付 Admin Web Portal、两档仿真模式和 MtM 实时估值模块。

> **v2.0 修订说明（相较 v1.1）**
> - 新增：本阶段在整体架构中的位置图
> - 新增：各任务架构定位说明（📐）
> - 新增：各任务禁止事项（⛔）
> - 新增：各子任务输入/输出规范
> - 新增：任务依赖关系图
> - 新增：DoD 验收方法与责任人
> - 新增：外部依赖降级预案（Chainlink 长时间不更新场景）
> - 新增：证书自动化轮换脚本交付（对应 Phase 0 §C 约定）

> **全局参考**：接口所有权表、争议仲裁路径、环境隔离策略、密钥证书生命周期、灾难恢复预案均定义在 Phase 0 §A–§D，本阶段不重复。

---

## 📍 本阶段在整体架构中的位置

```
本阶段目标：EVM 仿真器上线（两档），G1-G4 全部安全门控通过，G5/G6 正式评估，Admin Portal 可用，MtM 估值激活

[Zone A]                        [Zone B]                              [Zone C]
──────────────────────          ──────────────────────────────────    ──────────────────
Python Agent(已有)              ★ Go EVM 仿真器(两档: 轻量/完整)      假签名(仍在用)
★ G5/G6 正式评估(本阶段)         ★ FSM: INIT → SIMULATED
★ Admin Web Portal(本阶段)      ★ G2 精度校验(位级对齐)
                                ★ G3 不变性断言
                                ★ G4 极端 Fuzzing (1000轮)
                                ★ 双分录账本(记账引擎)
                                ★ MtM 估值(每区块 NAV)
                                ★ 奖励扫描器(RewardScanner)
                                ★ mTLS 证书自动轮换脚本
                                Adapter Registry(已有)
                                PostgreSQL / Redis(已有)

★ = 本 Phase 新增    括号 = 上阶段已有    — = 尚未存在（Phase 3+ 接入）
链上交互：无（仍使用历史数据 + Mock 签名，真实 RPC 在 Phase 3 接入）
```

**本阶段不涉及**：真实链上广播、实时 RPC 数据流、真实 MPC 签名。FSM 在本阶段仅跑通 `INIT → SIMULATED`，`SIGNED` 之后的路径在 Phase 4 完成。

---

## ⛔ 本阶段全局禁止事项

```
精度类：
  ⛔ 禁止仿真器内部使用 float/double（全程 math/big.Int，Go 侧严格执行）
  ⛔ 禁止 G2 校验接受"接近相等"（流动性计算允许 ≤ 1 wei，其余场景必须 == 0 误差）

FSM 类：
  ⛔ 禁止 FSM 状态变更不经过数据库事务（状态变更与分录写入必须原子提交）
  ⛔ 禁止跳过 Nonce 锁直接写 trade_tasks（必须先获取行级锁）

账本类：
  ⛔ 禁止分录写入时 SUM(debit) ≠ SUM(credit)（触发 SQL 事务回滚，不允许部分提交）
  ⛔ 禁止 MtM 估值使用 DEX Slot0 即时价（必须三源预言机中位价）

Admin Portal 类：
  ⛔ 禁止 Admin Web Portal 直接连接 Zone C（所有审批指令须经 Zone B Admin-Approval-API 中转）
  ⛔ 禁止 Portal 持有任何私钥或签名凭证

仿真档位类：
  ⛔ 禁止轻量仿真档混入完整仿真档的 Prometheus 指标（必须按 mode="full|lite" 分开统计）
```

---

## 任务 1：开发 Go EVM 仿真器（两档模式）

### 📐 架构定位

| 项目 | 内容 |
|---|---|
| **架构位置** | Zone B → Go-Execution-Engine 内部核心模块，消灭"回测陷阱"的最关键战役 |
| **对应架构文档** | `3.2_验证层_EVM仿真与三源预言机.md` §2–§3；`8.1_Go仿真器参考实现.md` §2–§4 |
| **在请求链路中的位置** | Sync Sentinel 水位线通过 → **[本模块: 仿真器]** → G3 不变性断言 → FSM 推进至 SIMULATED |
| **上游依赖** | Phase 0 DB Schema（trade_tasks）；Phase 1 Adapter Registry（获取协议 ABI） |
| **下游影响** | 阻塞任务 2（G2 精度校验依赖仿真器运行）；阻塞任务 3（G3/G4 依赖仿真器） |
| **接口 Owner** | 区块链底层与执行组 |

### ⛔ 本任务禁止事项

```
⛔ 禁止仿真器使用 float/double 进行任何金额计算（必须 math/big.Int）
⛔ 禁止轻量档和完整档共用 StateDB 实例（每次仿真须独立 Snapshot，defer Revert 保证隔离）
⛔ 禁止完整仿真超过 600ms 不取消（超时须返回 SimulationTimeout 错误，保护 FSM 不阻塞）
⛔ 禁止两档 Prometheus 指标混合计算（defi_fsm_transition_duration_ms 必须含 mode label）
```

---

### 子任务 1.1：构建 StateDB 内存仿真内核

**输入：**
- `8.1_Go仿真器参考实现.md` §2 StateDB 实现规范
- `go-ethereum/core/vm` 包

**输出：**
- `pkg/simulator/state_db.go`：MockStateDB 实现，含懒加载 + Snapshot/Revert 隔离
- 并发安全：每个 goroutine 持有独立 StateDB 副本（无全局锁），支持 50+ Agent 并发
- StateDB 热缓存预热脚本（`protocol_warmup.yaml`）：服务启动时预加载高频合约 Storage Slot
- 热缓存命中率指标：`defi_statedb_cache_hit_ratio`（目标 > 90%）

**不产出（明确排除）：**
- 不包含完整 Anvil 链上状态模拟（由子任务 1.3 负责）

---

### 子任务 1.2：实现轻量仿真档（常规交易路径）

**输入：**
- 子任务 1.1 产出的 StateDB 内核
- `TradeRequest` Protobuf（含 invariants 列表）
- `非功能性量化指标说明文档.md` 轻量档延迟预算

**输出：**
- `pkg/simulator/lite_simulator.go`：仅执行不变性断言，不走完整 Anvil 模拟
- 输出 `SimulationResult`：`{success, gas_estimated, amount_out, duration_ms, mode: "lite"}`
- 性能目标：P50 < 30ms，P99 < 80ms；超过 200ms 写降级日志（`warn: lite_sim_slow`）
- Prometheus：`defi_fsm_transition_duration_ms{transition="INIT_to_SIMULATED", mode="lite"}`

**适用范围**：非首次执行 + 单笔 ≤ 15% 分配资金 + 非 STAGED 状态

---

### 子任务 1.3：实现完整仿真档（大额/首次/STAGED 路径）

**输入：**
- 子任务 1.1 产出的 StateDB 内核
- 本地 Anvil 实例（on-demand 启动）
- 触发条件配置

**输出：**
- `pkg/simulator/full_simulator.go`：完整 Anvil 主网分叉仿真，含完整 EVM 状态变更、精确 Gas
- 触发条件判断（任一满足走完整档）：
  1. 单笔金额 > 策略当前分配资金 15%
  2. 该策略在当前区块首次执行
  3. 策略处于 STAGED 状态（全部交易强制完整档）
- 性能目标：P50 < 100ms，P99 < 250ms；超过 600ms 取消仿真，返回 `SimulationTimeout`
- Prometheus：`defi_fsm_transition_duration_ms{transition="INIT_to_SIMULATED", mode="full"}`

---

### 子任务 1.4：实现仿真档位路由逻辑

**输入：**
- 子任务 1.2、1.3 产出的两档仿真器
- `TradeRequest` 上下文（strategy_id、当前分配资金、lifecycle_state）

**输出：**
- `pkg/simulator/router.go`：根据请求参数自动判断档位，写结构化日志记录选择原因
  ```json
  {"level":"info","msg":"sim_route","mode":"full","reason":"staged_state","trace_id":"...","strategy_id":"..."}
  ```
- gRPC 拦截器集成：`ExecuteTrade` handler 调用 router，路由结果透传 FSM

---

## 任务 2：G2 精度校验——位级对齐

### 📐 架构定位

| 项目 | 内容 |
|---|---|
| **架构位置** | Zone B 验证层，EVM 仿真器与真实链结果的精度比对关卡 |
| **对应架构文档** | `验证门控标准说明文档.md` §3 G2 关卡规范 |
| **在请求链路中的位置** | EVM 仿真器输出 → **[G2 精度比对]** → 确认 Python 侧与 Go 侧计算误差 = 0 |
| **上游依赖** | 任务 1（EVM 仿真器可运行） |
| **下游影响** | G2 通过是 G3/G4 的前置条件（精度不对齐则断言无意义） |
| **接口 Owner** | 区块链底层与执行组（Go 侧）；AI 策略组（Python 测试用例提供方） |

### ⛔ 本任务禁止事项

```
⛔ 禁止接受流动性计算以外的任何场景出现 > 0 wei 的误差（非流动性计算必须精确 == 0）
⛔ 禁止使用生产主网数据进行 G2 测试（须使用测试网或本地 Anvil fork，对应 §B 环境隔离）
```

---

### 子任务 2.1：打通真实 gRPC 代理

**输入：**
- Phase 0 假 gRPC Server（待废弃）
- 任务 1 产出的 EVM 仿真器

**输出：**
- 废弃 Phase 0 假接口，Go gRPC handler 接收 `TradeRequest` 后输入真实仿真器
- 集成测试：Python 发送真实 CLAMM TradeRequest，Go 返回真实 `SimulationResult`

---

### 子任务 2.2：攻克 G2 精度校验

**输入：**
- AI 策略组提供的 500 组 CLAMM 随机测试用例（含边界值：uint256 max、最小 tick、极端价格比）
- 真实链上历史交易结果（作为 Ground Truth）

**输出：**
- `tests/g2_precision_test.go`：500 组对比测试
- 通过标准：
  - TickMath 计算：误差 == 0 wei
  - 整数除法舍入（floor）：误差 == 0 wei
  - uint256 边界值（0, 2^256-1）：误差 == 0 wei
  - 费率复利计算：误差 == 0 wei
  - 流动性计算（唯一例外）：误差 ≤ 1 wei
- G2 通过后写入 `gate_results`（`gate_id='G2'`）

---

## 任务 3：G3 不变性断言与 G4 极端 Fuzzing

### 📐 架构定位

| 项目 | 内容 |
|---|---|
| **架构位置** | Zone B 验证层 G3/G4 关卡，同时建立幂等 FSM 骨架 |
| **对应架构文档** | `验证门控标准说明文档.md` §3 G3/G4；`3.3_幂等状态机FSM.md` §2–§3 |
| **在请求链路中的位置** | 仿真输出 → **[G3 断言]** → FSM 状态推进；G4 Fuzzing 在 Anvil 注入极端参数验证 G3 覆盖率 |
| **上游依赖** | 任务 1（仿真器）、任务 2（G2 通过） |
| **下游影响** | 阻塞任务 4（双分录账本依赖 FSM SIMULATED 状态）；阻塞任务 7（G3/G4 是 G1-G4 的最后一环） |
| **接口 Owner** | 财务风控组（断言逻辑）；架构与 SRE 组（Fuzzing 参数注入） |

### ⛔ 本任务禁止事项

```
⛔ 禁止 G3 断言出现 999/1000 通过就判定为"基本满足"（必须 100% 通过，任何违反即退回 DRAFT）
⛔ 禁止 Nonce 锁使用乐观锁（必须 PostgreSQL FOR UPDATE 悲观锁，防止并发冲突）
⛔ 禁止 FSM 状态跳跃（INIT 只能推进至 SIMULATED，不可跳过中间状态）
```

---

### 子任务 3.1：实现幂等状态机（FSM）与 Nonce 锁

**输入：**
- `3.3_幂等状态机FSM.md` §2 状态定义与流转规则
- Phase 0 `trade_tasks`、`nonce_locks` 表 DDL

**输出：**
- `pkg/fsm/state_machine.go`：`INIT → SIMULATED` 状态流转，基于 PostgreSQL `FOR UPDATE` 悲观锁
- Nonce 行级锁逻辑：`AcquireNonce(address, chain_id) → nonce`，超时自动释放（TTL = 30s）
- 写入原则：先写 WAL 日志，后改状态（PostgreSQL 事务保证）
- 并发测试：`tests/test_nonce_concurrent.go`，50 goroutine 同时发单验证无重复无跳号

**不产出（明确排除）：**
- 不包含 SIGNED → BROADCASTED → CONFIRMED 路径（Phase 4 补全）
- 不包含 PAUSED/DEPRECATED 路径（Phase 3 任务 4 负责）

---

### 子任务 3.2：实现 G3 不变性断言

**输入：**
- 任务 1 仿真器产出的 StateDB 变更快照
- 策略声明的 `invariants()` 列表（来自 Phase 1 BaseStrategyAgent）

**输出：**
- `pkg/validators/invariant_checker.go`：捕获 StateDB 变更，强制校验四项约束：
  - `MaxDrawdown`：单次执行 U 本位跌幅 ≤ 1%
  - `MaxSlippage`：实际滑点 ≤ 策略声明容忍度
  - `MaxGasMultiple`：实际 Gas ≤ 估算 Gas × 3
  - `PositionLimitPct`：单策略持仓 ≤ 资金池上限
- 100% 场景满足要求，任何违反立即返回 `INVARIANT_BROKEN`，FSM 退回 DRAFT
- 告警：触发 `defi_invariant_violations_total{strategy_id, rule}` 指标

---

### 子任务 3.3：搭建 G4 极端 Fuzzing 测试

**输入：**
- 本地 Anvil 分叉环境
- 极端参数矩阵（来自 `验证门控标准说明文档.md` §3 G4 参数表）

**输出：**
- `tests/g4_fuzzing_test.go`：注入以下极端场景，每类 100–200 轮，总计 1000 轮：
  - Gas 飙升 1000 倍
  - 流动性枯竭至 0.1%
  - 预言机 ±90% 价格跳动
  - 网络超时 0–30s（随机）
  - Nonce 冲突（并发重复 Nonce）
  - 链重组 1–3 个区块
- 通过标准：1000 轮无破线（允许策略触发熔断拒绝发单，不允许产生超限亏损模拟记录）
- G4 通过写入 `gate_results`（`gate_id='G4'`）

---

## 任务 4：双分录会计核心引擎

### 📐 架构定位

| 项目 | 内容 |
|---|---|
| **架构位置** | Zone B 财务层，所有 PnL 核算的数学基础 |
| **对应架构文档** | `4.1_双分录会计系统.md` §3 标准分录规则；`3.5_数据库Schema汇总.md` §3–§5 |
| **在请求链路中的位置** | FSM 推进至 SIMULATED → **[记账引擎]** → ledger_entries 写入 → Reconciler 核查 |
| **上游依赖** | 任务 3（FSM SIMULATED 状态触发记账）；Phase 0 账本表 DDL |
| **下游影响** | 阻塞任务 6（MtM 估值依赖账本数据）；阻塞 Phase 3 影子对账 |
| **接口 Owner** | 财务风控组 |

### ⛔ 本任务禁止事项

```
⛔ 禁止 ledger_entries 写入时不校验借贷平衡（必须在同一事务中校验 SUM(debit) == SUM(credit)）
⛔ 禁止对账时使用 eth_call latest 标签（必须指定 block_number，防止重放不一致）
⛔ 禁止 GAS_FEE 分录合并入交易 PnL（Gas 成本须独立记录，不混入交易本金）
```

---

### 子任务 4.1：实现记账分录引擎

**输入：**
- `4.1_双分录会计系统.md` §3 标准分录规则（Swap / Add Liquidity / Claim Reward 三种基础类型）
- 任务 3 FSM 产出的仿真执行收据

**输出：**
- `pkg/accounting/ledger_engine.go`：将仿真收据转化为 Debit/Credit 分录
- 原子写入：`TRADE`、`GAS_FEE`、`PROTOCOL_FEE` 三类分录在同一数据库事务中提交
- 借贷平衡校验：写入前检查 `SUM(debit_amount) == SUM(credit_amount)`，不平衡则回滚
- 支持分录类型：`TRADE / GAS_FEE / PROTOCOL_FEE / REWARD / SLIPPAGE / MEV_LOSS`

---

### 子任务 4.2：实现账实核查（Reconciliation）

**输入：**
- `ledger_entries` 内部账本余额
- 链上 `eth_call`（指定 block_number，查询实际余额）

**输出：**
- `pkg/accounting/reconciler.go`：对比内部账本余额与链上实际余额
- 触发时机：每次 CONFIRMED 状态后触发 + 每小时全量扫描
- 硬性约束：差值 > 1 wei 立即触发 `RECONCILIATION_FAILURE` 告警，自动暂停策略（FSM → PAUSED）
- 指标：`defi_reconciliation_discrepancy_wei{strategy_id}` Gauge

---

## 任务 5：Admin Web Portal

### 📐 架构定位

| 项目 | 内容 |
|---|---|
| **架构位置** | Zone A（intelligence_net），对外暴露 HTTPS 443，不直接连接 Zone C |
| **对应架构文档** | `1.1_三区网络拓扑.md` §2 Zone A 定义；`5.3_全局断路器与时间锁.md` §1.3 Kill Switch 入口 |
| **在请求链路中的位置** | 管理员浏览器 → Admin Portal（Zone A）→ Admin-Approval-API（Zone B）→ Zone C 签名 |
| **上游依赖** | Phase 0 mTLS 证书体系；任务 3 FSM（提供 PENDING_APPROVAL 状态） |
| **下游影响** | Phase 3 影子模式上线前必须可用（管理员需要通过 Portal 处理审批队列） |
| **接口 Owner** | AI 与量化策略组（Web 方向开发）；区块链底层组（Admin-Approval-API） |

### ⛔ 本任务禁止事项

```
⛔ 禁止 Admin Web Portal 直接连接 Zone C（所有审批指令须经 Zone B Admin-Approval-API 中转）
⛔ 禁止 Portal 持有任何私钥、MPC 分片或签名凭证
⛔ 禁止 Portal 接口不经 2FA 直接暴露审批操作（Kill Switch 入口须二次确认弹窗 + 2FA）
⛔ 禁止审批操作不记录审计日志（操作者身份、时间戳、关联 trace_id 必须完整记录）
```

---

### 子任务 5.1：实现审批队列管理页面

**输入：**
- Zone B Admin-Approval-API 接口（GET /approvals/pending, POST /approvals/{id}/approve|reject）
- `策略生命周期定义说明文档.md` §3.5/§3.6 PENDING_APPROVAL 触发条件

**输出：**
- 列表页：展示 PENDING_APPROVAL 状态的交易队列，含字段：`strategy_id`、`trigger_reason`（`staged_high_value / live_high_ratio / new_contract / invariant_boundary`）、`estimated_value_usd`、`waiting_since`
- 批准/拒绝按钮：操作通过 Zone B API 中转，Portal 不直接发链上指令
- 超时提醒：等待 > 25 分钟高亮显示（接近 30 分钟自动 FAILED 阈值）
- 审计日志写入：`operator_id`、`action`、`trace_id`、`timestamp`

---

### 子任务 5.2：实现策略状态总览页面

**输入：**
- Zone B 状态查询 API（GET /strategies/{id}/status）
- 任务 6 产出的 MtM NAV 数据（`defi_strategy_pnl_usdc` Prometheus Gauge）

**输出：**
- 展示所有活跃策略：生命周期状态（DRAFT/VALIDATED/EVALUATED/SHADOWED/STAGED/LIVE/PAUSED）、实时 PnL（接入 MtM 数据）、最近 3 条告警、当前 MDD

---

### 子任务 5.3：实现 Kill Switch 入口

**输入：**
- Zone B Kill Switch API（POST /kill-switch/trigger）
- `5.3_全局断路器与时间锁.md` §1.3 Kill Switch 执行序列

**输出：**
- 高权限管理员专属入口（须满足 IP 白名单 + 2FA 验证才能访问）
- 点击后弹出二次确认弹窗：显示"此操作将触发全局 Kill Switch，影响所有 LIVE/STAGED 策略，确认？"
- 执行后展示进度：已撤销授权数 / 总协议数（实时轮询 API 更新）

---

### 子任务 5.4：访问控制

**输入：**
- §B 环境隔离策略（Portal 仅部署于各环境独立实例）

**输出：**
- IP 白名单限制（仅公司 VPN 出口 IP 可访问 HTTPS 443）
- 管理员登录须 2FA（TOTP，兼容 Google Authenticator）
- 操作完整审计日志，保留 1 年（合规要求）

---

## 任务 6：MtM 实时估值与未领取奖励扫描

### 📐 架构定位

| 项目 | 内容 |
|---|---|
| **架构位置** | Zone B 财务层，实时 MDD 监控的数据基础 |
| **对应架构文档** | `4.2_资产估值与审计.md` §2–§4；`3.2_验证层_EVM仿真与三源预言机.md` §3 |
| **在请求链路中的位置** | 每个新区块 → **[MtMCalculator]** → NAV 快照 → 实时 MDD 计算 → PAUSED 自动触发判断 |
| **上游依赖** | 任务 4（双分录账本提供资产数据）；Phase 1 Adapter Registry（协议接口） |
| **下游影响** | 阻塞 Phase 3 任务 4（PAUSED 自动触发 R1 规则依赖实时 MDD） |
| **接口 Owner** | 财务风控组 |

> **为什么在 Phase 2 而非 Phase 3 交付？** 实时 MDD 监控是 PAUSED 自动触发的判断依据（`实时 MDD > 回测 MDD × 1.5`），而 PAUSED 状态机在 Phase 3 交付。若 MtM 模块不先于 PAUSED 到位，Phase 3 的自动熔断就是空壳。

### ⛔ 本任务禁止事项

```
⛔ 禁止 MtM 估值使用 DEX Slot0 即时价（防止 MEV 操控价格影响风控）
⛔ 禁止未领取奖励写入 ledger_entries（未实际 Transfer，只记录 unclaimed_rewards 表，但必须计入 NAV）
⛔ 禁止 NAV 计算不含 BRIDGE_IN_TRANSIT 账户余额（在途资金须计入，防止误触发 MDD 告警）
```

---

### 子任务 6.1：实现 MtMCalculator 每区块 NAV 更新

**输入：**
- `4.2_资产估值与审计.md` §2 NAV 计算公式
- `ledger_entries` / `asset_balances` / `unclaimed_rewards` 表数据
- 三源预言机（本阶段使用历史价格模拟，Phase 3 接入真实预言机）

**输出：**
- `pkg/mtm/calculator.go`：`CalcNAV(strategy_id, at_block) → NAV_USDC`
  1. 从 `asset_balances` 读取已实现资产余额
  2. 从 `unclaimed_rewards` 读取未领取奖励（计入 NAV）
  3. `BRIDGE_IN_TRANSIT` 账户余额计入 NAV（防误触发）
  4. 所有资产通过三源中位价折算为 USDC（6 位精度整数）
  5. 减去未偿债务
- 写入 `mtm_snapshots` 表（`strategy_id`、`at_block`、`nav_usdc`、`timestamp`）
- Prometheus：`defi_strategy_pnl_usdc{strategy_id, pnl_type="total|realized|unrealized"}` 每区块更新

---

### 子任务 6.2：实现 RewardScanner 未领取奖励定时扫描

**输入：**
- Phase 1 Adapter Registry（获取各协议的奖励查询接口）
- `unclaimed_rewards` 表

**输出：**
- `pkg/mtm/reward_scanner.go`：定时任务（每 5–10 分钟），通过 Adapter 调用各协议查询待领取奖励
- Upsert 至 `unclaimed_rewards`（`strategy_id`、`protocol_id`、`token_address`、`raw_amount`、`last_scan_block`）
- 不产生链上交易（只读查询）

---

### 子任务 6.3：实现实时 MDD 监控

**输入：**
- 子任务 6.1 产出的 `mtm_snapshots` 数据
- Phase 2 G5 正式评估报告中的回测基准 MDD

**输出：**
- `pkg/mtm/drawdown_monitor.go`：`CalcRealtimeMDD(strategy_id) → realtime_mdd_pct`
  - `实时 MDD = (历史最高 NAV - 当前 NAV) / 历史最高 NAV`
- Prometheus：`defi_strategy_max_drawdown_pct{strategy_id}` Gauge
- 告警阈值配置：`realtime_mdd > backtest_mdd × 1.5` → 触发 `StrategyDrawdownAlert`（P0 CRITICAL）

**不产出（明确排除）：**
- PAUSED 自动触发逻辑在 Phase 3 任务 4 中实现（本模块只提供数据，不触发 FSM 变更）

---

## 任务 7：G5/G6 正式绩效评估

### 📐 架构定位

| 项目 | 内容 |
|---|---|
| **架构位置** | Zone A 策略层，Phase 1 预评估的正式版本 |
| **对应架构文档** | `验证门控标准说明文档.md` §4 G5/G6 通过标准 |
| **在请求链路中的位置** | G1-G4 全部通过 → **[重跑 G5/G6]** → OFFICIAL 评估报告 → 策略进入 VALIDATED 状态凭据 |
| **上游依赖** | 任务 2（G2 通过）+ 任务 3（G3/G4 通过）= G1-G4 全部通过后才可执行 |
| **下游影响** | 产出 OFFICIAL 报告是策略进入 VALIDATED 状态的唯一凭据 |
| **接口 Owner** | AI 与量化策略组 |

> ⚠️ **前置条件**：必须等待 G1-G4 全部通过（任务 2 + 任务 3 的 DoD 均验收通过）才可启动本任务。

### ⛔ 本任务禁止事项

```
⛔ 禁止在 G1-G4 尚未全部通过时提前运行本任务（中间状态的正式报告无效）
⛔ 禁止使用非 SDK 标准数据源（不得使用自行清洗的非标数据，须使用 Phase 1 Archiver 产出）
⛔ 禁止 OFFICIAL 报告不锁定内容（须含 strategy_version_hash + 数据集哈希，归档后不可修改）
```

---

### 子任务 7.1：执行 G5/G6 正式评估

**输入：**
- G1-G4 全部通过证明（gate_results 表中四项均有 PASS 记录）
- Phase 1 Archiver 产出的 Parquet 数据（SDK 标准数据源）
- Phase 1 预评估报告（作为对比基准）

**输出：**
- G5 正式回测报告（≥90 天，满足系统底线）：Sharpe ≥ 1.0、MDD ≤ 20%、Calmar ≥ 0.5、Gas 占比 ≤ 30%、资金利用率 ≥ 40%、Profit Factor ≥ 1.3
- G6 正式蒙特卡洛报告（5000 轮）：正收益占比 ≥ 60%、中位数年化 ≥ 0%、最差 5% 分位回撤 ≤ 40%
- 报告 JSON 必须包含（内容锁定，不可事后修改）：
  ```json
  {
    "evaluation_type": "OFFICIAL",
    "strategy_version_hash": "<git sha>",
    "dataset_sha256": "<Archiver 记录的数据集哈希>",
    "random_seed": 42,
    "g1_gate_passed": true,
    "g2_gate_passed": true,
    "g3_gate_passed": true,
    "g4_gate_passed": true,
    "g5_result": { ... },
    "g6_result": { ... }
  }
  ```
- 写入 `gate_results`（`gate_id='G5_OFFICIAL'` / `'G6_OFFICIAL'`），策略状态可推进至 VALIDATED

---

## 额外任务：mTLS 证书自动化轮换脚本

### 📐 架构定位

| 项目 | 内容 |
|---|---|
| **交付原因** | Phase 0 §C 约定：证书有效期 90 天，Phase 2 Week 1 必须交付轮换脚本（到期前 30 天） |
| **接口 Owner** | 架构与 SRE 组（Zone B 证书）；安全组（Zone C 证书） |

**输出：**
- 自动轮换脚本（`scripts/cert_rotate.sh`）：
  1. `defi_cert_expiry_days` 告警在到期前 14 天触发
  2. 到期前 7 天自动向 Intermediate CA 申请新证书
  3. 新旧证书并行有效 24 小时（双证书模式，不中断 mTLS 握手）
  4. 旧证书到期后自动吊销，写审计日志通知安全组确认
- 在 Prometheus 中注册 `defi_cert_expiry_days{zone="B|C"}` 指标

---

## 任务依赖关系

```
任务 1（Go EVM 仿真器两档）
  ├─ 阻塞 → 任务 2（G2 精度校验依赖仿真器运行）
  └─ 阻塞 → 任务 3（G3 断言捕获仿真器输出）

任务 2（G2 精度校验）
  └─ 阻塞 → 任务 7（G2 是 G1-G4 的组成部分）

任务 3（FSM + G3 断言 + G4 Fuzzing）
  ├─ 阻塞 → 任务 4（FSM SIMULATED 触发记账）
  ├─ 阻塞 → 任务 5（Admin Portal 审批队列依赖 FSM PENDING_APPROVAL 状态）
  └─ 阻塞 → 任务 7（G3/G4 是 G1-G4 的组成部分）

任务 4（双分录账本）
  └─ 阻塞 → 任务 6（MtM 估值读取 ledger_entries 数据）
  └─ 阻塞 → Phase 3 影子对账

任务 4 + 任务 6（双分录 + MtM 估值）
  └─ 共同阻塞 → Phase 3 任务 4（PAUSED 自动触发 R1/R2/R4 规则）

任务 2 + 任务 3（G1-G4 全部通过）
  └─ 解锁 → 任务 7（G5/G6 正式评估）

可并行执行：任务 1 + 任务 5（Admin Portal 前端）+ 额外任务（证书轮换）可同时启动
关键路径：任务 1 → 任务 2 → 任务 3 → 任务 7（最长串行链）
```

---

## 🏁 Phase 2 阶段验收标准（DoD）

| # | 验收标准 | 验收方法 | 验收责任人 |
|---|---|---|---|
| 1 | **两档仿真区分验收**：轻量 P99 ≤ 80ms，完整 P99 ≤ 250ms | 运行 `scripts/load_test_simulator.sh --mode lite --concurrency 50`，Grafana 中 `defi_fsm_transition_duration_ms{mode="lite"}` P99 ≤ 80ms；`{mode="full"}` P99 ≤ 250ms；两图分开展示 | 区块链底层组 Tech Lead |
| 2 | **G2 位级对齐（G2 Passed）**：500 组测试 0 误差 | 运行 `go test ./tests/g2_precision_test.go -v`，所有用例 `PASS`；查询 `gate_results` 表 `gate_id='G2', status='PASS'` | 区块链底层组 + AI 策略组交叉确认 |
| 3 | **并发压测（Nonce 安全）**：50 Agent 1 秒 100 笔，无重复 | 运行 `scripts/load_test_nonce.sh`，输出报告 `conflict_count == 0`；数据库 `nonce_locks` 无重复 nonce 行 | 区块链底层组 Tech Lead + 财务风控组交叉确认 |
| 4 | **G4 极限抗压（G4 Passed）**：1000 轮 Fuzzing 无破线 | 运行 `go test ./tests/g4_fuzzing_test.go`，所有轮次无超限亏损记录；查询 `gate_results` `gate_id='G4', status='PASS'` | 架构与 SRE 组 |
| 5 | **Admin Web Portal 可用**：审批操作可完整执行 | 浏览器打开 Portal URL，能看到 PENDING_APPROVAL 队列（构造测试数据）；执行一次批准 + 一次拒绝，Zone B API 返回 200；Kill Switch 入口渲染，弹窗正常触发 | AI 策略组 Web 方向 + 架构组 |
| 6 | **MtM 估值激活**：每区块 NAV 更新，奖励扫描运行 | Grafana 中 `defi_strategy_pnl_usdc` 每区块（12 秒）更新一次；查询 `unclaimed_rewards` 表，确认 10 分钟内有新的 `last_scan_block` 更新；`defi_strategy_max_drawdown_pct` Gauge 数据可读 | 财务风控组 |
| 7 | **G5/G6 正式评估报告（OFFICIAL）** | 查询 `gate_results` 表，存在 `gate_id='G5_OFFICIAL', status='PASS'` 记录；报告 JSON 含 `"evaluation_type": "OFFICIAL"`，Sharpe ≥ 1.0，MDD ≤ 20%，蒙特卡洛正收益占比 ≥ 60% | AI 策略组 + 财务风控组交叉确认 |
| 8 | **证书轮换脚本就绪** | 运行 `scripts/cert_rotate.sh --dry-run`，输出预计轮换时间表；Prometheus 中 `defi_cert_expiry_days` 指标可查（两个 Zone 的证书剩余天数） | 架构与 SRE 组 + 安全组 |
| 9 | **全链路日志可追溯** | 取一笔从 G1 扫描到 G4 Fuzzing 的 trace_id，在 Grafana Loki 中能检索全链路完整日志 | 架构与 SRE 组 |
