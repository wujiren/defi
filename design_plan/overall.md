# 开发计划概览

这是一份“策略先行”架构开发计划。该计划将整个系统拆解为五个阶段，通过明确的接口契约实现团队并行开发，让您的量化和 AI 团队能在第一周就投入到策略研发中，而基础设施团队则在底层同步构建安全防线。

## Phase 0：奠基与契约对齐（并行启动期）

**目标：** 确立系统的“通用语言”与物理边界，消除跨模块开发时的阻塞。
**参与团队：** 全体架构师、DevOps、DBA

* **任务 1：接口契约固化**
* 编译并分发 `common.proto`、`trading.proto` 和 `data_stream.proto`。明确 `BigAmount` 必须使用 `string` 传输 Wei 级别的原始整数，严禁浮点数过境。


* **任务 2：存储底座搭建**
* 在 PostgreSQL 中拉起核心 Schema：`trade_tasks`、`nonce_locks`、`ledger_entries`。
* 部署 Redis Cluster，创建 `stream:chain:events` Topic。


* **任务 3：三区网络隔离**
* 划分 Zone A（策略区）、Zone B（验证区）和 Zone C（签名区）的 VPC 与 Docker 网络。



## Phase 1：策略大脑与回测闭环（AI 投研期）

**目标：** 优先让核心策略逻辑跑通。在 AI Agent 平台架构下，聚焦于将历史数据喂给策略模型，完成无未来函数（Zero Look-Ahead）的评估。
**参与团队：** AI/策略组、数据工程组

* **任务 1：历史数据归档管道**
* 开发 `Archiver` 服务，将链上 Tick 数据（如 Swap、Mint、Burn）沉淀为 Parquet 格式，并存入 ClickHouse 索引。


* **任务 2：回测数据接口**
* 实现 `IDataProvider` 接口，确保策略在任意时间点只能获取当时及之前的截面数据。


* **任务 3：Python Agent 基类与首个策略**
* 封装 `BaseStrategyAgent`，处理好事件监听与精度转换。
* 利用该基类，优先针对 Concentrated Liquidity Automated Market Maker (CLAMM) 等复杂做市逻辑或收益聚合策略，编写第一版 `generate_proposals` 代码并跑通 G5（回测）与 G6（蒙特卡洛）关卡。



## Phase 2：核心引擎与数学对齐（沙盒校准期）

**目标：** 消除 Python 策略环境与真实 EVM 之间的误差。此阶段只做链下仿真，用 Mock 替代真实的签名和广播。
**参与团队：** 区块链底层开发组

* **任务 1：Go EVM 仿真器**
* 基于 `go-ethereum/core/vm` 实现轻量级 StateDB 仿真。
* 运行 G2 验证，确保 Python 端输出的 CLAMM 策略执行参数在 Go 端模拟计算时达到位级对齐（零误差）。


* **任务 2：gRPC 代理与不变性断言**
* 打通 Zone A 到 Zone B 的 `ExecuteTrade` RPC 通信。
* 实现最大滑点、最大回撤等风控物理底线的断言拦截。


* **任务 3：G4 极端模糊测试**
* 注入 Gas 飙升、流动性枯竭等极端参数，进行 1000 轮压力测试。



## Phase 3：实时数据与影子模式（虚拟实盘期）

**目标：** 接入真实的实时网络，但在最后一步物理截断，完成 G7 影子模式的端到端演练。
**参与团队：** 后端业务组、财务核算组

* **任务 1：实时链上事件推送**
* 开发 `Chain-Adapter` 接入 RPC，`Data-Processor` 强制注入 `last_sync_block` 水位线标签，写入 Redis Stream。


* **任务 2：水位线与预言机熔断**
* 实现 `Sync Sentinel` 硬拦截过期请求。
* 上线三源预言机（Triple-Oracle Guard），偏差超 0.5% 立即熔断。


* **任务 3：影子状态机与对账基础**
* 实现 FSM 的 `INIT -> SIMULATED` 路径，截断 `SIGNED`。
* 上线双分录会计系统（Double-Entry Ledger），开始记录影子模式下的虚拟 PnL。



## Phase 4：安全装甲与飞轮迭代（生产就绪期）

**目标：** 补齐全套安全设施，处理防夹与资金管控，激活系统自我学习能力。
**参与团队：** 安全工程组、智能合约组、AI组

* **任务 1：Zone C 物理签名与大额审批**
* 部署 Zone C 的 mTLS 白名单代理，对接真实的 MPC 节点。
* 实现在 STAGED 阶段与 LIVE 阶段差异化的大额人工审批逻辑。


* **任务 2：确定性广播与防夹**
* 实现 FSM 的 `Recovery Manager`（自愈管理器），处理进程崩溃后的状态恢复与 Nonce 锁释放。
* 上线 `MEV Router`，将高价值交易路由至 Flashbots 防御三明治攻击。


* **任务 3：断路器与 AI 反馈飞轮**
* 部署治理风险哨兵、24 小时时间锁与全局 Kill Switch。
* 通过审计模块计算软滑点与毒性流，将其转化为 Prompt 注入 Vector DB，驱动策略迭代。

# 开发团队建设

## 第一部分：团队建制与模块责任矩阵（优化版）

整个研发团队分为 **6 个内部跨职能小队 (Squads)** 和 **1 个外部协作方**。

### 1. 系统架构与 SRE 组 (Architecture, SRE & Chaos)

*系统的“基建狂魔”与“破坏者”，保障基础设施与抗脆弱性。*

* **人员画像**：资深架构师、云原生网络专家、**SRE / 混沌工程师**。
* **核心职责**：
* 统筹全局 `Protobuf` 契约（`common.proto` 等）与数据库 Schema。
* 搭建 Zone A/B/C 三区网络隔离、VPC 与 Docker 安全组。
* **混沌工程（新增）**：在影子模式和灰度阶段，专职负责“拔网线”、强制 Kill 掉 Go 进程、注入脏数据，验证 FSM 状态机的 15 秒自愈能力与“零重复执行”指标。



### 2. AI 与量化策略组 (Zone A: Intelligence Squad)

*系统的“大脑”，负责赚钱逻辑和认知进化。*

* **人员画像**：量化研究员、AI 算法工程师、Python 后端开发。
* **核心职责**：
* 维护 `BaseStrategyAgent`，处理所有策略的样板代码。
* 开发 CLAMM 动态做市等高频策略，跑通 G5/G6 历史回测与蒙特卡洛评估。
* 打通 Vector DB，将历史毒性流和滑点数据转化为大模型优化 Prompt，驱动 AI 反馈飞轮。



### 3. 区块链底层与执行组 (Zone B: Execution Squad)

*系统的“脊梁”，解决并发、执行确定性与黑暗森林博弈。*

* **人员画像**：资深 Go 语言工程师、EVM 底层专家、**专职 MEV 研究员**。
* **核心职责**：
* 开发 StateDB 内存仿真器，实现 Python 与 Solidity 的位级精度对齐（G2 门控）。
* 开发核心的**幂等状态机 (FSM)** 与 Nonce 行级锁。
* **防夹与路由（新增）**：MEV 研究员专职负责 `MEV Router` 的策略调优，监控 Mempool 毒性流特征，动态制定 Flashbots Bundle 竞价策略与 Gas 阶梯加速方案。



### 4. 数据工程组 (Data Pipeline Squad)

*系统的“眼耳神经”，负责数据摄取与历史归档。*

* **人员画像**：数据开发工程师、Redis/ClickHouse 专家。
* **核心职责**：
* 开发 `Chain-Adapter` 与 `Data-Processor`，为事件注入 `last_sync_block` 水位线。
* 开发 `Archiver` 归档服务与 `IDataProvider` 接口，支撑零 Look-Ahead 的历史回测。
* **数据连续性保障（对账协同）**：作为对账的底层支撑，负责确保 WebSocket 流不漏推、链重组事件被正确感知。



### 5. 财务风控与治理组 (Finance & Risk Squad)

*系统的“大管家”，确保账实相符，风险可控。*

* **人员画像**：金融账本开发专家、DeFi 风控研究员。
* **核心职责**：
* 维护 `ledger_entries`，实现双分录会计系统（Double-Entry Ledger）。
* 实现三源预言机校验、跨链资金（Saga 模式）回滚及水位线硬熔断 (`Sync Sentinel`)。
* **对账第一责任人（对账协同）**：编写核查逻辑。触发 `diff != 0` 告警时，财务组首当其冲排查账本逻辑；若确认逻辑无误，交由数据组排查底层事件丢失情况。



### 6. 极高安全组 (Zone C: Security Squad) & 外部审计

*系统的“保险箱”，掌握资金生杀大权。*

* **人员画像**：系统底层安全工程师、密码学专家 + **第三方独立审计机构 (External Auditors)**。
* **核心职责**：
* 部署 Zone C 的 mTLS 白名单代理与 MPC 节点。
* 部署 24 小时管理员时间锁合约与 Kill Switch。
* **独立审计（新增）**：在实盘资金注入前，交由第三方审计机构（如 Trail of Bits 等）对 Zone C 白名单代理源码、智能合约进行安全审计。



---

## 第二部分：“策略先行”五阶段开发计划（Timeline）

### Phase 0：奠基与契约对齐（并行启动期）

**目标：确立系统的“通用语言”与物理边界，消除阻塞。**

* **架构组/SRE**：分发全套 Protobuf 契约（确立 `string` 传输 Wei 金额的铁律）；拉起 PostgreSQL 核心表与 Redis Cluster。
* **里程碑**：各团队拿到本地可运行的 Mock 环境。

### Phase 1：策略大脑与回测闭环（AI 投研期）

**目标：AI 团队直接下场，跑通历史数据回测（零真实链上交互）。**

* **数据工程组**：开发 `Archiver` 将数据写入 Parquet，实现 `IDataProvider` 接口供回测调用。
* **AI 策略组**：基于 `BaseStrategyAgent` 编写第一批高频做市或资管策略，跑通 G5（回测）与 G6（蒙特卡洛）门控。
* **里程碑**：产生第一份达到 Sharpe Ratio ≥ 1.0 标准的结构化策略报告。

### Phase 2：核心引擎与数学对齐（沙盒校准期）

**目标：消除 Python 与真实 EVM 间的误差，跑通核心状态机流转。**

* **区块链底层组**：开发 Go StateDB 仿真器；实现 FSM (`INIT -> SIMULATED`)，使用 Mock 模拟签名。
* **财务风控组**：实现最大滑点、回撤等“不变性断言”（G3 门控）。
* **里程碑**：策略的输出在 Go 仿真器中实现位级对齐（G2）；通过 1000 轮极端 Fuzzing 测试（G4）。

### Phase 3：虚拟实盘与混沌测试（影子模式期）

**目标：接入实时数据，引入破坏性测试，跑通虚拟对账。**

* **数据工程组**：`Chain-Adapter` 接入真实 RPC，注入水位线推入 Redis。
* **财务风控组**：上线双分录账本，核算影子 PnL；激活 `Sync Sentinel` 水位线熔断与三源预言机校验。
* **架构组/SRE（混沌工程）**：在影子运行期间（G7门控），持续对 FSM 容器进行 Kill、断网测试，验证自愈管理器的能力。
* **里程碑**：策略在真实市场数据下稳定运行 ≥ 72 小时，假设收益与回测偏差 ≤ 15%。

### Phase 4：安全装甲与飞轮迭代（生产就绪期）

**目标：真金白银上线，防夹路由就绪，AI 形成自学习闭环。**

* **极高安全组**：上线 Zone C 白名单代理与 MPC 签名，引入**第三方审计团队**进行安全签核。
* **区块链底层组**：MEV 研究员主导上线 `MEV Router`，开始向 Flashbots 发送真实交易，调优 Gas 策略。
* **AI 策略组 & 财务风控组**：财务组生成软滑点与毒性流审计数据，AI 组将其转化为 Prompt 注入 Vector DB，驱动策略进入下一代迭代。
* **里程碑**：走完 STAGED（预上线审批）与 LIVE（生产自动签名），系统完成全形态进化。

---

## 第三部分：跨团队协作的“防扯皮”契约红线

在执行上述计划时，管理层必须严格捍卫以下规则：

1. **对账事故的工单流转（Finance vs Data）**
任何对账告警 (`diff != 0`)，工单第一处理人绝对是**财务风控组**。财务组必须先提供证据证明“双分录借贷逻辑”无误，才能将工单转给**数据工程组**去抓包排查 RPC 漏推。
2. **API 修改的“一票否决”**
`trading.proto` 和 `data_stream.proto` 是跨栈协作的根本。任何模块（即使是为了修复 Bug）如果试图在 Proto 中私自新增字段、或者想将 Wei 金额改为 `float` 传输，架构组拥有一票否决权。
3. **Zone C 的“不信任”原则**
无论 Go-Execution-Engine (Zone B) 测试得多么完美，安全组 (Zone C) 必须假设 Zone B 是**可能被攻陷的**。Zone C 必须强制接收**明文 Payload** 并独立在内网重新计算 Hash，绝不接受直接传过来的 Hash 值进行签名。