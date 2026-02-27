# Phase 1 详细分解：策略大脑与回测预评估（修订版 v1.1）

**⏱️ 预计耗时**：2-3 周
**🎯 核心目标**：在没有任何真实链上交互的情况下，搭建高并发回测环境，完成无未来函数（Zero Look-Ahead）的策略预评估。**同步交付 G1 静态分析门控与 Adapter Registry 基础框架。**

> **v1.1 修订说明：**
> - 新增任务 2：G1 静态代码分析门控（原计划完全缺失）
> - 新增任务 3：Adapter Registry 基础框架（原计划完全缺失）
> - 原任务 3 调整为任务 4，并明确本阶段 G5/G6 为"预评估"，**不具备生产有效性**
> - G5/G6 的生产有效版本在 Phase 2 完成 G1-G4 全部安全门控后方可正式跑出

---

## 任务 1：历史数据归档与回测引擎搭建

**主导团队**：数据工程组
**协办团队**：架构与 SRE 组（提供 ClickHouse/MinIO 存储环境）

为了让策略团队有真实的"弹药"进行回测，必须将链上历史数据标准化。

* **子任务 1.1：开发 `Archiver` 归档服务**
  * **工作内容**：编写 ETL 脚本，将历史链上 Tick 数据（如 Swap、Mint、Burn）清洗并压缩为 Parquet 列式存储格式。
  * **硬性约束**：按链、协议、日期建立三级分区；建立 ClickHouse 索引，支持按区块范围（谓词下推）极速检索。Gas 成本必须使用历史真实值（历史每区块的 `baseFee + priorityFee`），不得使用当前 Gas 价格估算。

* **子任务 1.2：实现 `IDataProvider` 接口**
  * **工作内容**：开发面向 Python Agent 的本地数据读取器。
  * **硬性约束（最高红线）**：必须实现**物理级别的零 Look-Ahead（无前向偏差）**。接口的 `get_data(target_block)` 方法必须严格阻断任何大于 `target_block` 的数据访问，防止回测收益虚高。每次回测任务须记录 `start_block`、`end_block` 和数据集 SHA-256 哈希，确保结果可复现。

* **子任务 1.3：预留实时管道接口**
  * **工作内容**：数据工程组开始构思 `Chain-Adapter` 的基本框架，预留向 Redis Stream 注入 `last_sync_block` 水位线的伪代码接口，为 Phase 3 接入真实数据做准备。

---

## 任务 2：G1 静态代码分析门控 *(新增)*

**主导团队**：AI 与量化策略组（工具使用）+ 架构与 SRE 组（工具开发）
**依赖**：无（可与任务 1 完全并行）

**为什么 G1 在 Phase 1 就要到位？** G1 是 G1-G4 全部安全门控的第一关，是策略代码进入任何验证流水线的前置条件。如果 G1 工具本阶段不到位，Phase 2 的 G2-G4 开发就缺乏一致的输入格式保障。

* **子任务 2.1：开发策略代码 AST 扫描工具**
  * **工作内容**：基于 Python `ast` 模块，实现对策略代码的静态扫描。
  * **检查项清单（全部为必过）**：
    1. **禁止直接 RPC 调用**：扫描 `web3.eth.*`、`requests.get(rpc_url)` 等直接访问链上节点的调用，策略代码必须通过 SDK DataAPI 获取数据
    2. **禁止浮点金额**：扫描 `float()`、`int(amount)` 等对金额的直接浮点转换，必须使用 `Decimal` → `str` 传输
    3. **invariants 非空校验**：策略类的 `invariants()` 方法返回值不得为空列表
    4. **协议适配器合法性**：策略引用的 `protocol_id` 必须在 Adapter Registry 中已注册
    5. **禁止时间戳作为随机源**：扫描 `datetime.now()` 等在链下引入非确定性的调用（会导致回测结果不可复现）
  * **交付物**：`sdk.validate.static_check(strategy_file)` 接口，返回结构化 JSON 报告（PASS/FAIL/WARN 三态）。

* **子任务 2.2：将 G1 集成进策略提交流程**
  * **工作内容**：在 SDK `validate()` 入口的第一步调用静态扫描工具。扫描未通过的策略直接拒绝，不进入后续 G2-G4 流程。同时将 G1 结果写入 Grafana（触发 `defi_gate_g1_failures_total` 指标更新）。

---

## 任务 3：Adapter Registry 基础框架 *(新增)*

**主导团队**：区块链底层与执行组（Zone B）
**依赖**：任务 1.1（历史数据接口）、Phase 0 的 PostgreSQL Schema

**为什么 Adapter Registry 在 Phase 1 就要建？**
1. G1 的第 4 条检查项（`protocol_id` 合法性）依赖 Registry 存在
2. Phase 2 的 G3 不变性断言中，协议适配器需要通过 Registry 分发 `payload`
3. Phase 4 的 Whitelist Proxy 和治理哨兵都从 Registry 读取合约地址与状态
早建早受益，晚建会成为多个团队的阻塞点。

* **子任务 3.1：定义 Registry 数据模型**
  * **工作内容**：在 PostgreSQL 新建 `protocol_adapters` 表，字段包含：
    - `protocol_id`（如 `UNISWAP_V3_ETH`）
    - `contract_addresses`（JSONB 数组，对应白名单来源）
    - `allowed_method_selectors`（允许调用的函数选择器列表）
    - `status`：`ACTIVE / HIGH_RISK / SUSPENDED`
    - `registered_at`、`updated_at`

* **子任务 3.2：实现 Registry 核心接口**
  * **工作内容**：开发以下接口：
    - `Register(protocol_id, config)`：注册新协议（须走 Phase 4 的时间锁流程，本阶段先实现注册逻辑，时间锁在 Phase 4 接入）
    - `GetStatus(protocol_id) → ACTIVE | HIGH_RISK | SUSPENDED`
    - `SetStatus(protocol_id, status, reason)`：供治理哨兵调用
    - `IsAllowedAddress(to_address) → bool`：供 Zone C Whitelist Proxy 调用
    - `GetAdapter(protocol_id) → AdapterConfig`：供 Go 执行引擎分发 payload

* **子任务 3.3：录入第一批协议**
  * **工作内容**：录入 CLAMM 策略涉及的协议（Uniswap V3/V4 Factory、Pool 合约地址及函数选择器），状态设为 `ACTIVE`。同时编写 Registry 的单元测试，覆盖地址校验和状态变更逻辑。

---

## 任务 4：Python Agent 框架与评估指标库

**主导团队**：AI 与量化策略组（Zone A）

在写具体策略前，必须把基类和"裁判系统"建好。

* **子任务 4.1：封装 `BaseStrategyAgent` 基类**
  * **工作内容**：处理底层事件监听，封装所有与 Zone B 交互的样板代码。
  * **关键功能**：实现影子模式切换（`is_shadow` 参数）；强制的浮点数安全转换（`Decimal` → `string` 传输 Wei）；将策略生成的提案打包为 Protobuf 格式的 `TradeRequest`；在 `create_context()` 中生成 UUID v4 格式的 `trace_id` 并注入 `last_sync_block`。

* **子任务 4.2：开发 25 项评估指标计算库**
  * **工作内容**：依据 `评估指标体系说明文档` 将 5 大类、25 个核心指标代码化。
  * **硬性约束**：所有的收益计算必须折算为 U 本位（USDC 6 位小数整数）；Gas 成本（按历史真实值）和协议手续费必须计入净收益扣减；Sharpe Ratio 无风险利率固定为 0（DeFi 场景）。

---

## 任务 5：首批策略研发与 G5/G6 预评估

**主导团队**：AI 与量化策略组（Zone A）

> ⚠️ **重要说明：本任务产出的 G5/G6 报告为"预评估"性质，不具备生产有效性，不可作为策略进入 VALIDATED 状态的凭据。** 正式的 G5/G6 评估须在 Phase 2 完成 G1-G4 全部安全门控之后重新运行，才具备生产有效性。本阶段的预评估目的是尽早发现策略逻辑问题，缩短整体研发周期。

* **子任务 5.1：开发集中流动性做市（CLAMM）策略**
  * **工作内容**：基于 Uniswap V3/V4，编写动态调整流动性区间的 `generate_proposals` 逻辑。
  * **风控注入**：在策略内部显式声明物理底线，如 `invariants() -> [MaxDrawdown(pct=0.01), MaxSlippage(pct=0.005)]`。

* **子任务 5.2：跑通 G5/G6 预评估**
  * **工作内容**：将上述策略接入回测引擎，执行 >90 天历史重放（G5 预评估），并运行 5000 轮蒙特卡洛随机路径模拟（G6 预评估）。
  * **报告标注**：评估报告 JSON 中必须包含 `"evaluation_type": "PRE_EVAL_NOT_PRODUCTION_VALID"`，明确标识其不具备生产有效性。

---

## 🏁 Phase 1 阶段合并与验收标准（DoD）

1. **G1 门控工具就绪**：对一段包含直接 RPC 调用的策略代码执行静态扫描，必须返回 `FAIL` 并指出具体行号和违规原因。

2. **Adapter Registry 基础可用**：可以通过 Registry API 查询 Uniswap V3 的合约地址和方法选择器，`IsAllowedAddress()` 接口对非注册地址返回 `false`。

3. **回测防作弊验证**：代码审查确认 `IDataProvider` 接口无法通过任何手段提取 `current_block + 1` 的数据；每次回测任务生成唯一数据集哈希。

4. **基类序列化验证**：`BaseStrategyAgent` 能够成功将 CLAMM 策略的复杂参数打包进 Protobuf 的 `payload` 字节流中，且金额严格为 String，`trace_id` 自动生成且格式为 UUID v4。

5. **预评估结果存档（非生产凭据）**：至少有一个策略的预评估报告显示系统底线 Sharpe Ratio ≥ 1.0、最大回撤 MDD ≤ 20%、蒙特卡洛正收益占比 ≥ 60%。报告带有 `PRE_EVAL_NOT_PRODUCTION_VALID` 标注，归档存储备用。

6. **可观测性连通**：策略预评估运行时，Grafana 日志面板能检索到对应 `trace_id` 的结构化日志条目。
