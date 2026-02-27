# 开发计划概览（修订版 v1.1）

这是一份"策略先行"架构开发计划。该计划将整个系统拆解为五个阶段，通过明确的接口契约实现团队并行开发，让量化和 AI 团队能在第一周就投入到策略研发中，而基础设施团队则在底层同步构建安全防线。

> **v1.1 修订说明：** 本版本修正了以下六项缺失：
> 1. G1-G4 与 G5-G6 门控顺序错位（G5/G6 需在 G1-G4 之后才具备生产有效性）
> 2. 策略异常路径（PAUSED / DEPRECATED 状态机）缺少开发任务
> 3. Admin Web Portal 无明确归属团队与交付阶段
> 4. Protocol Adapter Registry 未作为独立开发任务
> 5. MtM 实时估值与未领取奖励扫描模块缺失
> 6. 可观测性基础设施未提前部署，非功能性指标（两档仿真、SLA）无专项验收

---

## Phase 0：奠基与契约对齐（并行启动期）

**目标：** 确立系统的"通用语言"与物理边界，消除跨模块开发时的阻塞。**同步部署可观测性基础设施，确保后续阶段有度量手段。**
**参与团队：** 全体架构师、DevOps、DBA、财务风控组

* **任务 1：接口契约固化**
  编译并分发 `common.proto`、`trading.proto` 和 `data_stream.proto`。明确 `BigAmount` 必须使用 `string` 传输 Wei 级别的原始整数，严禁浮点数过境。

* **任务 2：存储底座搭建**
  在 PostgreSQL 中拉起核心 Schema：`trade_tasks`、`nonce_locks`、`ledger_entries`、`accounts`、`asset_balances`。部署 Redis Cluster，创建 `stream:chain:events` 和 `stream:internal:accounting` 两个 Topic。

* **任务 3：三区网络隔离**
  划分 Zone A（策略区）、Zone B（验证区）和 Zone C（签名区）的 VPC 与 Docker 网络；生成内部 Root CA，签发 mTLS 证书体系。

* **任务 4：可观测性基础设施部署** *(新增)*
  部署 Prometheus + Alertmanager + Grafana + Loki 全套可观测性栈。配置各服务 `/metrics` 抓取端点，建立 16 条核心告警规则的初始框架，确保 Phase 1 起各阶段的开发质量有数据支撑。

* **任务 5：制造假通道（Mocking）解除阻塞**
  各团队 Tech Lead 协作，交付 Zone B 假 gRPC Server、Zone C 假签名服务、假数据推送脚本，让 Phase 1 各团队可以立即并行开工。

---

## Phase 1：策略大脑与回测预评估（AI 投研期）

**目标：** 优先让核心策略逻辑跑通，同时完成 G1 静态分析门控与 Adapter Registry 基础框架。**注意：本阶段的 G5/G6 为"预评估"，不具备生产有效性，须在 Phase 2 完成 G1-G4 后重跑才算正式通过。**
**参与团队：** AI/策略组、数据工程组、区块链底层组（Adapter Registry 部分）

* **任务 1：历史数据归档管道**
  开发 `Archiver` 服务，沉淀 Parquet 格式历史数据；实现零 Look-Ahead 的 `IDataProvider` 接口。

* **任务 2：G1 静态分析门控** *(新增)*
  实现策略代码的 AST 静态扫描工具：检测直接 RPC 调用、浮点金额传输、invariants 为空等违规模式。这是所有安全门控的第一关，须在策略进入回测前通过。

* **任务 3：Adapter Registry 基础框架** *(新增)*
  开发协议适配器注册表（Adapter Registry），包含注册、查询、状态标记（ACTIVE / HIGH_RISK / SUSPENDED）接口。这是白名单校验、治理哨兵和时间锁联动的共同基础。

* **任务 4：Python Agent 基类与首个策略**
  封装 `BaseStrategyAgent`；针对 CLAMM 等策略编写 `generate_proposals` 并跑通 G5/G6 **预评估**（数据作为基准存档，不作为上线凭据）。

---

## Phase 2：核心引擎与数学对齐（沙盒校准期）

**目标：** 消除 Python 策略环境与真实 EVM 之间的误差。完成 G1-G4 全部安全门控，并在此基础上正式通过 G5/G6 绩效门控。**同步交付 Admin Web Portal 和 MtM 实时估值模块。**
**参与团队：** 区块链底层开发组、财务风控组、AI 策略组

* **任务 1：Go EVM 仿真器（两档模式）** *(修订)*
  基于 `go-ethereum/core/vm` 实现 StateDB 仿真，须同时支持两档模式：
  - **轻量仿真**（常规交易）：仅验证不变性断言，P99 < 80ms
  - **完整仿真**（大额 / 首次执行）：完整 Anvil 模拟，P99 < 250ms
  完整仿真触发条件：单笔 > 策略资金 15%、策略首次执行、或处于 STAGED 状态。

* **任务 2：G2/G3/G4 安全门控**
  通过 G2（500 组随机输入位级对齐）、G3（不变性断言全量拦截）、G4（1000 轮极端 Fuzzing）。G1-G4 全部通过后，策略方可进入 VALIDATED 状态。

* **任务 3：G5/G6 正式绩效评估** *(修订)*
  在 G1-G4 通过后重跑历史回测（G5）与蒙特卡洛（G6），此次结果具备生产有效性，可作为上线凭据。

* **任务 4：Admin Web Portal** *(新增)*
  由 AI/策略组 Web 方向开发，部署于 Zone A。承担：PENDING_APPROVAL 审批队列展示与放行/拒绝操作、策略状态总览、Kill Switch 入口。Phase 3 影子模式上线前必须可用。

* **任务 5：MtM 实时估值与奖励扫描** *(新增)*
  由财务风控组实现 `MtMCalculator`（每区块更新 NAV）和 `RewardScanner`（每 5-10 分钟扫描未领取奖励），两者均计入 NAV，为实时 MDD 监控提供准确基准。

---

## Phase 3：实时数据与影子模式（虚拟实盘期）

**目标：** 接入真实的实时网络，完成 G7 影子模式的端到端演练。**同步实现策略异常路径（PAUSED / DEPRECATED）和 NFR 专项验收。**
**参与团队：** 后端业务组、财务核算组、架构/SRE 组、区块链底层组

* **任务 1：实时链上事件推送**
  开发 `Chain-Adapter` 接入 RPC，`Data-Processor` 强制注入 `last_sync_block` 水位线标签，写入 Redis Stream。

* **任务 2：水位线与预言机熔断**
  实现 `Sync Sentinel` 硬拦截过期请求；上线三源预言机，偏差超 0.5% 立即熔断。

* **任务 3：PAUSED / DEPRECATED 异常路径** *(新增)*
  实现策略生命周期的完整异常路径，包括：
  - 自动触发 PAUSED 的 7 条规则（实时 MDD 超限、治理哨兵告警、连续亏损等）
  - PAUSED → LIVE 的人工确认恢复流程
  - DEPRECATED 的自动退役条件（适配器失效、熔断超次数上限）及资金归还、数据归档流程

* **任务 4：影子状态机与对账基础**
  实现 FSM 的 `INIT → SIMULATED` 路径，截断 `SIGNED`；上线双分录会计系统，开始记录影子模式下的虚拟 PnL。

* **任务 5：NFR 专项验收** *(新增)*
  由 SRE 组主导，针对非功能性量化指标进行专项压测：
  - 50+ Agent 并发时端到端 P99 延迟 < 500ms（轻量路径）/ < 680ms（完整路径）
  - 核心链路可用性 ≥ 99.9%/月（混沌工程验证）
  - 历史数据可复现性 100%（数据哈希校验）

---

## Phase 4：安全装甲与飞轮迭代（生产就绪期）

**目标：** 补齐全套安全设施，处理防夹与资金管控，激活系统自我学习能力，完成外部安全审计。
**参与团队：** 安全工程组、智能合约组、AI 组、第三方审计机构

* **任务 1：Zone C 物理签名与大额审批**
  部署 Zone C 的 mTLS 白名单代理，对接真实的 MPC 节点；实现 STAGED 阶段（绝对金额 $10,000）与 LIVE 阶段（相对比例 15%）差异化的大额人工审批逻辑；上线 Kill Switch 绿色通道（绕过审批阈值的紧急提款路径）。

* **任务 2：确定性广播与防夹**
  实现 FSM 的 `Recovery Manager`；上线 `MEV Router`，将高价值交易路由至 Flashbots。

* **任务 3：断路器与 AI 反馈飞轮**
  部署治理风险哨兵（联动 Adapter Registry 状态）、24 小时时间锁与全局 Kill Switch；通过审计模块计算软滑点与毒性流，将其转化为 Prompt 注入 Vector DB，驱动策略迭代。

---

## 开发团队建设

（详见 `team_overall.md`）

---

## 跨团队协作的"防扯皮"契约红线

在执行上述计划时，管理层必须严格捍卫以下规则：

1. **门控顺序不可颠倒**：G5/G6 的"预评估"结果不得用于上线申请。只有 G1-G4 全部通过后的正式评估报告，才是策略进入 VALIDATED 状态的凭据。

2. **对账事故的工单流转（Finance vs Data）**：任何对账告警（`diff != 0`），工单第一处理人绝对是**财务风控组**。财务组必须先提供证据证明"双分录借贷逻辑"无误，才能将工单转给**数据工程组**。

3. **API 修改的"一票否决"**：`trading.proto` 和 `data_stream.proto` 是跨栈协作的根本。任何私自新增字段或将 Wei 金额改为 `float` 传输的尝试，架构组拥有一票否决权。

4. **Zone C 的"不信任"原则**：无论 Zone B 测试得多完美，Zone C 必须假设 Zone B 可能被攻陷，强制接收明文 Payload 并独立在内网重新计算 Hash，绝不接受直接传来的 Hash 值进行签名。

5. **Adapter Registry 是白名单的唯一来源**：Zone C Whitelist Proxy 的合约白名单必须从 Adapter Registry 读取，任何绕过 Registry 直接硬编码合约地址的做法视为安全违规。
