# Phase 1：策略大脑与回测预评估（修订版 v2.0）

**⏱️ 预计耗时**：2–3 周
**🎯 核心目标**：在没有任何真实链上交互的情况下，搭建高并发回测环境，完成零 Look-Ahead 的策略预评估。同步交付 G1 静态分析门控与 Adapter Registry 基础框架。

> **v2.0 修订说明（相较 v1.1）**
> - 新增：本阶段在整体架构中的位置图
> - 新增：各任务架构定位说明（📐）
> - 新增：各任务禁止事项（⛔）
> - 新增：各子任务输入/输出规范
> - 新增：任务依赖关系图
> - 新增：DoD 验收方法与责任人
> - 明确：本阶段 G5/G6 为"预评估"，不具备生产有效性，正式评估在 Phase 2 重跑

> **全局参考**：接口所有权表、争议仲裁路径、环境隔离策略、密钥证书生命周期、灾难恢复预案均定义在 Phase 0 §A–§D，本阶段不重复，遇到问题优先查阅。

---

## 📍 本阶段在整体架构中的位置

```
本阶段目标：历史数据就绪，G1门控可用，Adapter Registry建立，策略预评估跑通

[Zone A]                      [Zone B]                        [Zone C]
──────────────────────        ──────────────────────────      ──────────────────
★ BaseStrategyAgent基类       ★ Archiver归档服务              假签名(仍在用)
★ CLAMM首批策略               ★ IDataProvider接口
★ G5/G6预评估(非生产凭据)      ★ G1静态分析工具
★ 25项评估指标库               ★ Adapter Registry(注册/查询)
                               PostgreSQL(已有)
                               Redis Stream(已有)
                               Prometheus/Grafana(已有)

★ = 本 Phase 新增    括号 = 上阶段已有    — = 尚未存在
链上交互：无（全部使用历史归档数据 + Mock假通道）
```

**本阶段不涉及**：任何真实 EVM 仿真、任何链上交互、任何真实签名、任何实时 RPC 数据接入。

---

## ⛔ 本阶段全局禁止事项

```
回测防作弊类：
  ⛔ 禁止 IDataProvider 返回 target_block + N（任意 N > 0）的任何数据（物理级零 Look-Ahead）
  ⛔ 禁止回测中使用当前 Gas 价格估算历史成本（必须使用历史 baseFee + priorityFee 真实记录）
  ⛔ 禁止策略代码在回测中调用 datetime.now() 等非确定性来源（影响结果可复现性）
  ⛔ 禁止将带有 PRE_EVAL_NOT_PRODUCTION_VALID 标注的报告用于策略上线申请

金融精度类（继承自 Phase 0）：
  ⛔ 禁止用 float/double 表示金额（必须 Decimal → string 传输 Wei）
  ⛔ 禁止策略代码直接调用 RPC（必须通过 SDK DataAPI）

Adapter Registry 类：
  ⛔ 禁止策略代码引用未在 Registry 中注册的 protocol_id（G1 第 4 条检查项）
  ⛔ 禁止在 Registry 内部硬编码合约地址（须通过注册接口写入，为 Phase 4 时间锁联动预留钩子）
```

---

## 任务 1：历史数据归档与回测引擎搭建

### 📐 架构定位

| 项目 | 内容 |
|---|---|
| **架构位置** | Zone B 数据层 → Archiver 归档服务 + IDataProvider 接口 |
| **对应架构文档** | `2.4_历史数据归档.md` §4 数据格式与分区；`7.1_回测引擎与数据接口.md` §2–§3 |
| **在请求链路中的位置** | 链上历史原始数据 → Archiver → Parquet 归档 → IDataProvider → Python Agent 回测 |
| **上游依赖** | Phase 0 的 PostgreSQL、MinIO/S3 存储环境 |
| **下游影响** | 阻塞任务 5（Python Agent 需要数据源跑 G5/G6 预评估） |
| **接口 Owner** | 数据工程组 |

### ⛔ 本任务禁止事项

```
⛔ 禁止 IDataProvider 使用任何形式的 "peek ahead"（包括缓存预读超出 target_block）
⛔ 禁止归档时丢弃历史 Gas 记录（baseFee + priorityFee 必须完整保留，用于成本回测）
⛔ 禁止分区策略使用时间戳（必须使用 chain/protocol/date 三级，确保区块范围可精确检索）
```

---

### 子任务 1.1：开发 Archiver 归档服务

**输入：**
- `2.4_历史数据归档.md` §4 数据格式与分区规范
- 历史链上 Tick 数据（Swap / Mint / Burn Event Log）
- Phase 0 搭建的 MinIO/S3 存储环境

**输出：**
- Parquet 格式历史数据，按 `chain_id/protocol_id/date` 三级分区存储
- ClickHouse 索引（支持按区块范围谓词下推查询，目标 P99 < 500ms）
- 每个归档批次生成 `dataset_hash`（SHA-256），写入 PostgreSQL `archive_manifests` 表
- Gas 成本字段：`base_fee_per_gas`、`priority_fee_per_gas`（来自历史区块头，不得估算）

**不产出（明确排除）：**
- 不包含实时数据流接入（由 Phase 3 Chain-Adapter 负责）
- 不包含 L3/L4 聚合层（由 Data-Processor 在 Phase 3 负责）

---

### 子任务 1.2：实现 IDataProvider 接口

**输入：**
- `7.1_回测引擎与数据接口.md` §2 接口规范
- 子任务 1.1 产出的 Parquet 归档数据

**输出：**
- `IDataProvider` Python 接口实现，核心方法：
  - `get_data(target_block: int) -> BlockData`：严格阻断任何 `> target_block` 的数据访问
  - `get_gas_cost(block_number: int) -> GasInfo`：返回历史真实 baseFee + priorityFee
- 每次回测任务自动记录 `start_block`、`end_block`、`dataset_sha256`，写入 PostgreSQL `backtest_runs` 表
- 防越界测试：`test_no_lookahead.py`，覆盖直接访问、缓存绕过、切片越界三种场景

**不产出（明确排除）：**
- 不包含实时数据推送（`stream:chain:events` 由 Phase 3 负责）

---

### 子任务 1.3：预留实时管道接口

**输入：**
- `2.2_Redis_Stream实时管道.md` §2 消息结构（仅作参考，本阶段不实现）

**输出：**
- `ChainAdapterInterface` 抽象基类（Python），定义 `subscribe_events(consumer_group)` 方法签名
- 接口文档注释：说明 Phase 3 Chain-Adapter 须实现此接口，保持 Python Agent 代码不改动

---

## 任务 2：G1 静态代码分析门控

### 📐 架构定位

| 项目 | 内容 |
|---|---|
| **架构位置** | SDK 验证层第一关，策略代码进入任何验证流水线的前置条件 |
| **对应架构文档** | `验证门控标准说明文档.md` §3 G1 关卡；`8.2_Python_Agent基类参考实现.md` §3 |
| **在请求链路中的位置** | 策略代码提交 → `sdk.validate()` 入口 → **[本模块]** → G2-G4（如 G1 失败则终止） |
| **上游依赖** | 任务 3（Adapter Registry，G1 第 4 条检查项需查询 Registry） |
| **下游影响** | 阻塞任务 5（策略须先通过 G1 才能跑 G5/G6 预评估） |
| **接口 Owner** | 架构与 SRE 组（工具开发）；AI 与量化策略组（工具使用） |

### ⛔ 本任务禁止事项

```
⛔ 禁止 G1 检查结果被策略开发者绕过（G1 失败的策略不得进入 G2-G4 流水线，须在 SDK 入口强制拦截）
⛔ 禁止 G1 报告仅输出 PASS/FAIL，必须输出结构化 JSON（含违规行号、违规类型、修复建议）
⛔ 禁止 G1 工具依赖运行时执行策略代码（必须纯静态 AST 分析，不得执行任何代码）
```

---

### 子任务 2.1：开发策略代码 AST 扫描工具

**输入：**
- `验证门控标准说明文档.md` §3 G1 五项检查规范
- Python `ast` 模块文档
- 任务 3 产出的 Adapter Registry API（用于第 4 项检查）

**输出：**
- `sdk/validators/static_checker.py`：基于 Python `ast` 模块的扫描工具
- 五项检查逻辑（全部为必过）：
  1. `check_no_direct_rpc`：扫描 `web3.eth.*`、`requests.get(rpc_url)` 等直接 RPC 调用
  2. `check_no_float_amount`：扫描 `float()`、`int(amount)` 金额浮点转换（须使用 `Decimal → str`）
  3. `check_invariants_nonempty`：策略类 `invariants()` 返回值不得为空列表
  4. `check_protocol_registered`：策略引用的 `protocol_id` 必须在 Adapter Registry 中已注册
  5. `check_no_datetime_source`：扫描 `datetime.now()`、`time.time()` 等非确定性来源
- 返回结构化 JSON 报告：`{status: PASS|FAIL|WARN, violations: [{rule, line, column, message, fix_hint}]}`
- Prometheus 指标写入：`defi_gate_g1_failures_total{rule_id}` 计数

**不产出（明确排除）：**
- 不执行任何策略代码（纯静态分析）
- 不包含 G2-G4 的任何逻辑

---

### 子任务 2.2：将 G1 集成进策略提交流程

**输入：**
- 子任务 2.1 产出的扫描工具
- `8.2_Python_Agent基类参考实现.md` §3 SDK validate() 入口

**输出：**
- `sdk.validate(strategy_file)` 入口的第一步强制调用 `static_checker.check()`
- G1 失败时立即抛出 `G1ValidationError`，阻断后续 G2-G4 流程（不得配置跳过）
- G1 结果写入 PostgreSQL `gate_results` 表（`gate_id='G1'`、`strategy_id`、`status`、`violations_json`）

---

## 任务 3：Adapter Registry 基础框架

### 📐 架构定位

| 项目 | 内容 |
|---|---|
| **架构位置** | Zone B 共享基础服务，白名单校验、治理哨兵、时间锁的共同数据源 |
| **对应架构文档** | `5.1_MPC签名与白名单拦截.md` §2 白名单来源；`3.3_幂等状态机FSM.md` §4 协议适配器 |
| **在请求链路中的位置** | G1 检查 → [Registry 查询 protocol_id] → Zone C Whitelist Proxy → [Registry 查询合约地址] |
| **上游依赖** | Phase 0 的 PostgreSQL Schema |
| **下游影响** | 阻塞 G1 第 4 条检查、Phase 4 Zone C Whitelist Proxy、Phase 4 治理哨兵 |
| **接口 Owner** | 区块链底层与执行组 |

### ⛔ 本任务禁止事项

```
⛔ 禁止在 Registry 内部硬编码任何合约地址（所有地址须通过 Register() 接口写入）
⛔ 禁止直接修改数据库 protocol_adapters 表（须通过 Registry 接口操作，为审计日志留痕）
⛔ 禁止生产环境跳过时间锁直接注册新协议（本阶段开发环境允许直接注册，Phase 4 接入时间锁）
```

---

### 子任务 3.1：定义 Registry 数据模型

**输入：**
- `5.1_MPC签名与白名单拦截.md` §2 白名单数据结构
- Phase 0 子任务 2.3 产出的 PostgreSQL 实例

**输出：**
- PostgreSQL `protocol_adapters` 表 DDL：
  ```sql
  CREATE TABLE protocol_adapters (
    protocol_id        VARCHAR(64) PRIMARY KEY,           -- 如 UNISWAP_V3_ETH
    chain_id           INTEGER NOT NULL,
    contract_addresses JSONB NOT NULL,                    -- 白名单合约地址数组
    allowed_selectors  JSONB NOT NULL,                    -- 允许的函数选择器列表
    status             VARCHAR(16) NOT NULL               -- ACTIVE / HIGH_RISK / SUSPENDED
                       CHECK (status IN ('ACTIVE','HIGH_RISK','SUSPENDED')),
    registered_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at         TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_by         VARCHAR(64),                       -- 操作来源（治理哨兵/管理员）
    update_reason      TEXT
  );
  ```
- 索引：`(chain_id, status)`、`(contract_addresses) GIN`（支持地址快速查找）

---

### 子任务 3.2：实现 Registry 核心接口

**输入：**
- 子任务 3.1 产出的数据模型
- `5.1_MPC签名与白名单拦截.md` §2 接口规范

**输出：**
- Go gRPC 服务 `AdapterRegistryService`，实现以下接口：
  - `Register(protocol_id, config) → error`：注册新协议（开发环境直接生效，Phase 4 接时间锁）
  - `GetStatus(protocol_id) → {ACTIVE|HIGH_RISK|SUSPENDED}`
  - `SetStatus(protocol_id, status, reason)`：供治理哨兵调用，状态变更立即生效
  - `IsAllowedAddress(to_address, chain_id) → bool`：供 Zone C Whitelist Proxy 调用
  - `GetAdapter(protocol_id) → AdapterConfig`：供 Go 执行引擎分发 payload
- 每次 SetStatus 写审计日志：`registry_audit_log` 表（`protocol_id`、`old_status`、`new_status`、`reason`、`changed_by`、`changed_at`）

**不产出（明确排除）：**
- 不包含时间锁逻辑（Phase 4 接入）
- 不包含治理哨兵本体（Phase 4 开发）

---

### 子任务 3.3：录入第一批协议

**输入：**
- Uniswap V3/V4 合约地址文档（测试网 + 主网，按 chain_id 区分，对应 §B 环境隔离规则）
- CLAMM 策略依赖的函数选择器清单

**输出：**
- 初始化 SQL 脚本，录入 UNISWAP_V3_ETH、UNISWAP_V4_ETH（状态：ACTIVE）
- 测试网地址（Sepolia chain_id=11155111）与主网地址（chain_id=1）分条记录，不混用
- Registry 单元测试覆盖：地址精确匹配、非注册地址返回 false、状态变更后查询一致性

---

## 任务 4：Python Agent 框架与评估指标库

### 📐 架构定位

| 项目 | 内容 |
|---|---|
| **架构位置** | Zone A 策略层骨架，所有具体策略的基类 |
| **对应架构文档** | `8.2_Python_Agent基类参考实现.md` §2–§3；`评估指标体系说明文档.md` §3–§7 |
| **在请求链路中的位置** | Python Agent → `BaseStrategyAgent.generate_proposals()` → `TradeRequest` Protobuf → Zone B gRPC |
| **上游依赖** | Phase 0 的 Proto 桩代码；任务 2 的 G1 工具（集成在 SDK validate 中） |
| **下游影响** | 阻塞任务 5（首批策略基于基类开发） |
| **接口 Owner** | AI 与量化策略组 |

### ⛔ 本任务禁止事项

```
⛔ 禁止基类内部使用 float/double 处理金额（必须 Decimal → str → BigAmount.value）
⛔ 禁止 trace_id 由策略逻辑自定义（必须在 BaseStrategyAgent.create_context() 中统一生成 UUID v4）
⛔ 禁止策略代码绕过基类直接构造 TradeRequest（须通过基类的 submit_proposal() 方法）
⛔ 禁止 Sharpe Ratio 计算使用非零无风险利率（DeFi 场景固定为 0）
```

---

### 子任务 4.1：封装 BaseStrategyAgent 基类

**输入：**
- `8.2_Python_Agent基类参考实现.md` §2 基类规范
- Phase 0 产出的 `trading_pb2.py` 桩代码
- `6.1_全链路Trace_ID规范.md` §3 Trace ID 生成规范

**输出：**
- `sdk/agents/base_strategy_agent.py`，核心方法：
  - `create_context() → Context`：生成 UUID v4 `trace_id`，注入 `last_sync_block`（来自 IDataProvider 当前位置）
  - `submit_proposal(proposal) → TradeRequest`：将策略提案转为 Protobuf，金额强制 `Decimal → str`
  - `run_shadow(is_shadow: bool)`：影子模式开关，`True` 时调用 `ShadowTrade` RPC（不广播）
  - `invariants() → List[InvariantRule]`：子类必须重写，基类返回空时抛出异常（与 G1 第 3 条联动）
- 100% 单元测试覆盖上述核心方法

---

### 子任务 4.2：开发 25 项评估指标计算库

**输入：**
- `评估指标体系说明文档.md` §3–§7 全部 25 个指标定义与计算口径

**输出：**
- `sdk/metrics/evaluator.py`，实现 5 大类指标：
  - **收益类**（5项）：净收益、Sharpe（无风险利率=0）、Sortino、Calmar、Profit Factor（≥1.3）
  - **风险类**（5项）：MDD、下行波动率、VaR(95%)、月度亏损频率、最大连续亏损
  - **成本类**（5项）：Gas 占比（≤30%）、无常损失估算、MEV 损失估算、协议手续费、滑点成本
  - **效率类**（5项）：资金利用率（≥40%）、交易成功率、换手率、执行偏差率、仓位集中度
  - **健康类**（5项）：对账差异（零容忍）、数据新鲜度、熔断触发率、Nonce 冲突率、孤儿任务率
- 所有收益计算折算为 USDC（6 位小数整数，禁止浮点）
- Gas 成本使用 `IDataProvider.get_gas_cost(block)` 历史真实值

---

## 任务 5：首批策略研发与 G5/G6 预评估

### 📐 架构定位

| 项目 | 内容 |
|---|---|
| **架构位置** | Zone A 策略层，第一个具体策略实现 |
| **对应架构文档** | `验证门控标准说明文档.md` §4 G5/G6 标准；`评估指标体系说明文档.md` §8 双层阈值 |
| **在请求链路中的位置** | IDataProvider → BaseStrategyAgent → G5 历史回测 → G6 蒙特卡洛 → 预评估报告（存档） |
| **上游依赖** | 任务 1（数据）、任务 2（G1 工具）、任务 3（Registry）、任务 4（基类+指标库） |
| **下游影响** | 预评估结果存档，Phase 2 完成 G1-G4 后重跑正式评估 |
| **接口 Owner** | AI 与量化策略组 |

> ⚠️ **重要说明：本任务产出的 G5/G6 报告为"预评估"，不具备生产有效性，不可作为策略进入 VALIDATED 状态的凭据。** 正式评估须在 Phase 2 完成 G1-G4 全部安全门控后重新运行。

### ⛔ 本任务禁止事项

```
⛔ 禁止预评估报告省略 PRE_EVAL_NOT_PRODUCTION_VALID 标注（必须在 JSON 顶层字段明确标注）
⛔ 禁止将预评估 Sharpe ≥ 1.0 的结论用于向管理层声称"策略已通过验证"
⛔ 禁止策略代码跳过 G1 扫描直接进入回测（sdk.validate() 入口强制执行，无法绕过）
```

---

### 子任务 5.1：开发 CLAMM 集中流动性做市策略

**输入：**
- 任务 4 产出的 `BaseStrategyAgent` 基类
- 任务 3 注册的 Uniswap V3/V4 Adapter 配置
- Uniswap V3/V4 TickMath、流动性计算逻辑文档

**输出：**
- `strategies/clamm_market_maker.py`：实现 `generate_proposals()` 动态调整流动性区间逻辑
- 策略内显式声明不变性约束：
  ```python
  def invariants(self):
      return [
          MaxDrawdown(pct=Decimal("0.01")),      # 单次执行 U 本位跌幅 ≤ 1%
          MaxSlippage(pct=Decimal("0.005")),     # 实际滑点 ≤ 0.5%
          MaxGasMultiple(multiple=Decimal("3")), # 实际 Gas ≤ 估算 × 3
      ]
  ```
- 通过 G1 扫描（由 sdk.validate() 强制检查）

---

### 子任务 5.2：跑通 G5/G6 预评估

**输入：**
- 子任务 5.1 产出的 CLAMM 策略
- 任务 1 产出的 IDataProvider（≥90 天历史数据）
- 任务 4 产出的 25 项评估指标库

**输出：**
- G5 历史回测报告（≥90 天，事务级回放）：含 Sharpe、MDD、Calmar、Gas 占比、资金利用率等
- G6 蒙特卡洛报告（5000 轮，随机数种子固定可复现）：含正收益占比、中位数年化、最差 5% 分位
- 报告 JSON 必须包含：
  ```json
  {
    "evaluation_type": "PRE_EVAL_NOT_PRODUCTION_VALID",
    "strategy_version_hash": "<git sha>",
    "dataset_sha256": "<IDataProvider 记录的数据集哈希>",
    "random_seed": 42,
    "g5_result": {...},
    "g6_result": {...}
  }
  ```
- 报告归档至 PostgreSQL `gate_results`（`gate_id='G5_PRE'` / `'G6_PRE'`）

**不产出（明确排除）：**
- 不产出 `evaluation_type: OFFICIAL` 的报告（须等待 Phase 2 G1-G4 全部通过后）

---

## 任务依赖关系

```
任务 1（历史数据归档 + IDataProvider）
  └─ 阻塞 → 任务 5（策略预评估需要数据源）

任务 2（G1 静态分析工具）
  ├─ 依赖 → 任务 3（G1 第 4 条需查询 Registry）
  └─ 阻塞 → 任务 5（策略须先通过 G1）

任务 3（Adapter Registry）
  ├─ 阻塞 → 任务 2.1（G1 第 4 条依赖 Registry 存在）
  └─ 阻塞 → Phase 2 任务 3（G3 不变性断言中协议适配器通过 Registry 分发）
  └─ 阻塞 → Phase 4 任务 1（Zone C Whitelist Proxy 从 Registry 读合约白名单）

任务 4（BaseStrategyAgent + 指标库）
  └─ 阻塞 → 任务 5（首批策略基于基类开发）

任务 5（首批策略 + G5/G6 预评估）
  └─ 依赖 → 任务 1 + 任务 2 + 任务 3 + 任务 4（全部前置）
  └─ 产出存档 → Phase 2 任务 7（正式评估对比基准）

可并行执行：任务 1 + 任务 3 + 任务 4（三任务可同时启动）
串行依赖：任务 2 依赖任务 3（部分），任务 5 依赖全部前置任务
关键路径：任务 3 → 任务 2 → 任务 5（最长串行链）
```

---

## 🏁 Phase 1 阶段验收标准（DoD）

| # | 验收标准 | 验收方法 | 验收责任人 |
|---|---|---|---|
| 1 | **G1 门控拦截有效**：含直接 RPC 调用的策略代码必须返回 FAIL 并指出行号 | 运行 `sdk.validate("tests/fixtures/bad_strategy_rpc.py")`，检查返回 JSON 的 `status == "FAIL"` 且 `violations[0].rule == "check_no_direct_rpc"` | 架构与 SRE 组 |
| 2 | **Adapter Registry 基础可用**：Uniswap V3 合约地址可查，非注册地址返回 false | `curl GET /registry/adapters/UNISWAP_V3_ETH` 返回合约地址列表；`IsAllowedAddress("0xdeadbeef")` 返回 `false` | 区块链底层组 |
| 3 | **回测防作弊验证**：IDataProvider 物理阻断 Look-Ahead | 代码审查确认 `get_data(target_block + 1)` 抛出 `LookAheadError`；运行 `pytest tests/test_no_lookahead.py`，0 项失败 | 数据工程组 + 架构组交叉审查 |
| 4 | **基类序列化验证**：金额为 String，trace_id 为 UUID v4 | 运行 `pytest tests/test_base_agent_serialization.py`：检查 `TradeRequest.amount` 字段为 string 类型，`trace_id` 通过 UUID v4 正则校验 | AI 策略组 Tech Lead |
| 5 | **预评估结果存档**：至少一个策略达到系统底线且带 PRE_EVAL 标注 | 查询 `gate_results` 表，确认存在 `gate_id='G5_PRE'` 记录，`result.sharpe >= 1.0`、`result.mdd <= 0.20`，`result.evaluation_type == "PRE_EVAL_NOT_PRODUCTION_VALID"` | AI 策略组 + 财务风控组交叉确认 |
| 6 | **可观测性连通**：预评估运行时日志可检索 | 在 Grafana Loki 中以任一预评估 `trace_id` 检索，返回 ≥ 3 条结构化日志（含 `component`、`sync_block` 字段） | 架构与 SRE 组 |
| 7 | **数据集哈希可复现**：同参数两次回测哈希一致 | 运行同一策略两次 G5 预评估（相同 start_block、end_block、random_seed），检查两次 `dataset_sha256` 完全一致 | 数据工程组 |
