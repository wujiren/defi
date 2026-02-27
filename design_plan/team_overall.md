# DeFi 基础设施极致并行开发计划（修订版 v1.1）

这是一个**"接口先行，契约驱动"（API-First & Contract-Driven）**的敏捷开发思路。为了实现极致并行化，我们采用**"定义契约 → 制造假数据/假接口（Mocking）→ 多线并行开发 → 合并联调（Merge）"**的循环。

> **v1.1 修订说明：**
> - 各阶段补充 G1 静态分析门控、Adapter Registry、Admin Web Portal、MtM 估值、PAUSED/DEPRECATED 状态机、NFR 专项验收的团队任务与里程碑
> - 第一部分团队建制中，AI 策略组补充 Admin Web Portal 职责，区块链底层组补充 PAUSED/DEPRECATED 责任，财务风控组补充 MtM 估值职责
> - 第三部分"防扯皮"契约红线新增两条规则

---

## 第一部分：团队建制与模块责任矩阵（优化版）

整个研发团队分为 **6 个内部跨职能小队（Squads）** 和 **1 个外部协作方**。

### 1. 系统架构与 SRE 组（Architecture, SRE & Chaos）

*系统的"基建狂魔"与"破坏者"，保障基础设施与抗脆弱性。*

* **人员画像**：资深架构师、云原生网络专家、SRE / 混沌工程师。
* **核心职责**：
  - 统筹全局 Protobuf 契约（`common.proto` 等）与数据库 Schema
  - 搭建 Zone A/B/C 三区网络隔离、VPC 与 Docker 安全组
  - **Phase 0 新增**：部署 Prometheus + Alertmanager + Grafana + Loki 可观测性全栈；分发 `6_2_Prometheus指标定义.md` 中的埋点规范，确保各组按统一命名空间（`defi_`）埋点
  - **混沌工程**：在影子模式和灰度阶段，专职"拔网线"、强制 Kill 掉 Go 进程、注入脏数据，验证 FSM 状态机的 15 秒自愈能力与"零重复执行"指标
  - **NFR 专项验收**（Phase 3）：主导端到端延迟（两档）、可用性 SLA、数据可复现性、50 并发扩展性的专项压测

---

### 2. AI 与量化策略组（Zone A: Intelligence Squad）

*系统的"大脑"，负责赚钱逻辑和认知进化。*

* **人员画像**：量化研究员、AI 算法工程师、Python 后端开发（含 Web 方向 1 名）。
* **核心职责**：
  - 维护 `BaseStrategyAgent`，处理所有策略的样板代码
  - 开发 CLAMM 动态做市等高频策略，跑通 G5/G6 历史回测与蒙特卡洛评估
  - **G1 静态分析工具使用（Phase 1）**：负责在策略提交前执行 G1 扫描，确保所有策略代码通过静态检查（工具由架构组开发，使用方是 AI 策略组）
  - **Admin Web Portal（Phase 2，新增）**：由 Web 方向工程师负责开发 Zone A 的管理后台，包含审批队列管理、策略状态总览、Kill Switch 入口；Phase 3 影子模式前交付
  - 打通 Vector DB，将历史毒性流和滑点数据转化为大模型优化 Prompt，驱动 AI 反馈飞轮

---

### 3. 区块链底层与执行组（Zone B: Execution Squad）

*系统的"脊梁"，解决并发、执行确定性与黑暗森林博弈。*

* **人员画像**：资深 Go 语言工程师、EVM 底层专家、专职 MEV 研究员。
* **核心职责**：
  - 开发 StateDB 内存仿真器，实现 Python 与 Solidity 的位级精度对齐（G2 门控）；须同时实现**轻量仿真档**（P99 < 80ms）和**完整仿真档**（P99 < 250ms）
  - 开发核心的**幂等状态机（FSM）**与 Nonce 行级锁
  - **PAUSED / DEPRECATED 状态机（Phase 3，新增）**：实现策略异常路径的完整 FSM 逻辑，包含 7 条自动 PAUSED 触发规则、人工恢复审批流、DEPRECATED 退役 + 资金归还 + 数据归档流程
  - **Adapter Registry（Phase 1，新增）**：开发协议适配器注册表，包含注册、状态标记、白名单查询接口，是 Zone C Whitelist Proxy 和治理哨兵的共同基础
  - 防夹与路由：MEV 研究员专职负责 `MEV Router` 策略调优，监控 Mempool 毒性流特征，动态制定 Flashbots Bundle 竞价策略

---

### 4. 数据工程组（Data Pipeline Squad）

*系统的"眼耳神经"，负责数据摄取与历史归档。*

* **人员画像**：数据开发工程师、Redis/ClickHouse 专家。
* **核心职责**：
  - 开发 `Chain-Adapter` 与 `Data-Processor`，为事件注入 `last_sync_block` 水位线
  - 开发 `Archiver` 归档服务与 `IDataProvider` 接口，支撑零 Look-Ahead 的历史回测；每次回测须生成数据集 SHA-256 哈希（支撑 NFR 可复现性验收）
  - **数据连续性保障（对账协同）**：确保 WebSocket 流不漏推、链重组事件被正确感知；对账告警时为第二处理人（财务风控组首当其冲，确认账本无误后转给数据组排查底层事件缺失）

---

### 5. 财务风控与治理组（Finance & Risk Squad）

*系统的"大管家"，确保账实相符，风险可控。*

* **人员画像**：金融账本开发专家、DeFi 风控研究员。
* **核心职责**：
  - 维护 `ledger_entries`，实现双分录会计系统（Double-Entry Ledger）
  - 实现三源预言机校验、跨链资金（Saga 模式）回滚及水位线硬熔断（`Sync Sentinel`）
  - **MtM 实时估值与奖励扫描（Phase 2，新增）**：实现 `MtMCalculator`（每区块更新 NAV）和 `RewardScanner`（每 5-10 分钟扫描未领取奖励），两者共同构成实时 MDD 监控的数据基础，是 PAUSED 自动触发的上游依赖
  - **AI 飞轮审计数据投递**：负责在交易 CONFIRMED 后 **60 秒内**完成软滑点计算并写入 Redis Stream，确保 AI 反馈飞轮的时效 SLA
  - **对账第一责任人**：触发 `diff != 0` 告警时，财务风控组须首先证明双分录借贷逻辑无误，才能将工单转给数据工程组

---

### 6. 极高安全组（Zone C: Security Squad）& 外部审计

*系统的"保险箱"，掌握资金生杀大权。*

* **人员画像**：系统底层安全工程师、密码学专家 + 第三方独立审计机构（External Auditors）。
* **核心职责**：
  - 部署 Zone C 的 mTLS 白名单代理与 MPC 节点；实现接收明文 Payload、独立计算 Keccak256 Hash 的不信任原则
  - 部署 24 小时管理员时间锁合约（作用域：全局风控参数修改；不含策略状态转换）与 Kill Switch
  - **Kill Switch 绿色通道（Phase 4，新增）**：实现 `IS_KILL_SWITCH` 标记的防伪造校验逻辑（签名来源验证），确保紧急提款路径绕过审批阈值的安全性
  - **Adapter Registry 时间锁联动（Phase 4，新增）**：将生产环境新协议注册操作接入 24h 时间锁流程
  - **独立审计**：在实盘资金注入前，委托第三方审计机构对 Zone C Whitelist Proxy 源码、时间锁合约、Kill Switch 绿色通道防伪造逻辑进行安全审计

---

## 第二部分："策略先行"五阶段开发计划（Timeline）

### Phase 0：奠基与契约对齐（并行启动期）

**目标：确立系统的"通用语言"与物理边界，同步建立可观测性基础设施。**

* **架构组/SRE**：分发全套 Protobuf 契约（铁律：`string` 传输 Wei 金额）；拉起 PostgreSQL 核心表与 Redis Cluster；**部署 Prometheus + Grafana + Loki**，分发指标埋点规范。
* **各组 Tech Lead**：协作交付 Mocking 假通道（Zone B 假 gRPC Server、Zone C 假签名服务、假数据推送脚本）。
* **里程碑**：各团队拿到本地可运行的 Mock 环境；Grafana 仪表盘可访问，假通道的请求日志（含 `trace_id`）可检索。

### Phase 1：策略大脑与回测预评估（AI 投研期）

**目标：AI 团队直接下场，跑通历史数据回测；同步交付 G1 门控工具和 Adapter Registry。**

* **数据工程组**：开发 `Archiver` 写入 Parquet，实现 `IDataProvider` 接口（数据集哈希记录）。
* **架构组（协作）**：开发 G1 静态代码分析工具（`sdk.validate.static_check()`）。
* **区块链底层组**：开发 Adapter Registry（注册、查询、状态标记接口），录入 CLAMM 协议第一批白名单。
* **AI 策略组**：基于 `BaseStrategyAgent` 编写第一批策略；通过 G1 扫描后，跑通 G5/G6 **预评估**（报告标注 `PRE_EVAL_NOT_PRODUCTION_VALID`）。
* **里程碑**：G1 工具可识别违规策略代码；Adapter Registry API 可用；至少一个策略的预评估 Sharpe Ratio ≥ 1.0（存档备用，非生产凭据）。

### Phase 2：核心引擎与数学对齐（沙盒校准期）

**目标：消除 Python 与真实 EVM 间的误差，完成 G1-G4 全部安全门控，正式通过 G5/G6，交付 Admin Web Portal 和 MtM 估值。**

* **区块链底层组**：开发 Go StateDB 仿真器（两档：轻量 P99<80ms / 完整 P99<250ms）；实现 FSM（`INIT → SIMULATED`），Mock 签名。
* **财务风控组**：实现 G3 不变性断言（G3 门控）；实现双分录账本；**实现 MtM 实时 NAV 更新和未领取奖励扫描**。
* **AI 策略组（Web 方向）**：**交付 Admin Web Portal**（审批队列、策略总览、Kill Switch 入口、2FA 访问控制）。
* **G1-G4 全部通过后**：AI 策略组使用 SDK 标准数据源重新运行 G5/G6 **正式评估**，生成 `"evaluation_type": "OFFICIAL"` 报告。
* **里程碑**：位级精度对齐（G2）；1000 轮极端 Fuzzing（G4）通过；Admin Web Portal 可登录并操作审批队列；MtM NAV 每区块更新；G5/G6 官方报告产出（具备上线凭据）。

### Phase 3：虚拟实盘与混沌测试（影子模式期）

**目标：接入实时数据，引入破坏性测试，跑通虚拟对账，实现策略异常路径，完成 NFR 专项验收。**

* **数据工程组**：`Chain-Adapter` 接入真实 RPC，注入水位线推入 Redis。
* **财务风控组**：上线双分录账本，核算影子 PnL；激活 `Sync Sentinel` 与三源预言机熔断；MtM 与 PAUSED 自动触发联动。
* **区块链底层组**：**实现 PAUSED / DEPRECATED 状态机完整逻辑**（7 条自动触发规则、人工恢复流、退役资金归还与数据归档）。
* **架构组/SRE（混沌工程）**：持续对 FSM 容器进行 Kill、断网测试，验证 Recovery Manager 15 秒自愈能力。
* **架构组/SRE（NFR 验收）**：执行两档延迟压测、可用性 SLA 统计、数据可复现性验证、50 并发扩展性测试。
* **里程碑**：策略在真实市场数据下稳定运行 ≥ 72 小时，影子 PnL 与回测偏差 ≤ 15%（G7）；PAUSED 自动触发可演示；NFR 两档延迟、SLA、可复现性全部通过。

### Phase 4：安全装甲与飞轮迭代（生产就绪期）

**目标：真金白银上线，防夹路由就绪，AI 形成自学习闭环，外部审计签核。**

* **极高安全组**：上线 Zone C 白名单代理与 MPC 签名；实现 Kill Switch 绿色通道防伪造逻辑；Adapter Registry 接入时间锁；引入**第三方审计团队**签核。
* **区块链底层组**：MEV 研究员主导上线 `MEV Router`，调优动态 Gas 竞价策略。
* **AI 策略组 & 财务风控组**：财务组 60 秒内投递软滑点/毒性流审计数据；AI 组向量化后注入 Vector DB，DEPRECATED 策略数据作为负样本批量写入，驱动策略进化。
* **里程碑**：外部安全审计报告出具；10 笔 Flashbots Bundle 实盘验证防夹效果；Kill Switch 60 秒清零演练通过；第一笔真实主网交易成功落地，系统正式进入 LIVE 阶段。

---

## 第三部分：跨团队协作的"防扯皮"契约红线

在执行上述计划时，管理层必须严格捍卫以下规则：

1. **门控顺序不可颠倒**
   G5/G6 的"预评估"结果（Phase 1 产出）**不得**用于上线申请。只有 G1-G4 全部通过后的**正式评估报告**（Phase 2 产出，含 `"evaluation_type": "OFFICIAL"` 标注），才是策略进入 VALIDATED 状态的唯一凭据。

2. **对账事故的工单流转（Finance vs Data）**
   任何对账告警（`diff != 0`），工单第一处理人绝对是**财务风控组**。财务组必须先提供证据证明"双分录借贷逻辑"无误，才能将工单转给**数据工程组**去抓包排查 RPC 漏推。

3. **API 修改的"一票否决"**
   `trading.proto` 和 `data_stream.proto` 是跨栈协作的根本。任何模块如果试图在 Proto 中私自新增字段、或将 Wei 金额改为 `float` 传输，**架构组拥有一票否决权**。

4. **Zone C 的"不信任"原则**
   无论 Go-Execution-Engine（Zone B）测试得多么完美，安全组（Zone C）必须假设 Zone B 是可能被攻陷的。Zone C 须强制接收**明文 Payload**，独立在内网重新计算 Hash，**绝不接受**直接传来的 Hash 值进行签名。

5. **Adapter Registry 是白名单的唯一来源** *(新增)*
   Zone C Whitelist Proxy 的合约白名单必须从 Adapter Registry 动态读取，**任何绕过 Registry 直接在 Proxy 内部硬编码合约地址的做法视为安全违规**，架构组和安全组均有权阻止此类代码合并。生产环境的 Registry 修改须走 24h 时间锁。

6. **Admin Web Portal 不持有签名权限** *(新增)*
   Admin Web Portal（Zone A）的任何审批操作，均须通过 Zone B 的 `Admin-Approval-API` 中转，Portal 不直接连接 Zone C，不持有任何私钥或签名凭证。任何试图在 Portal 中直接发起链上操作的设计方案，安全组拥有一票否决权。
