# Uniswap V3 ALM 系统架构设计规格书 (v1.2)

## 1. 逻辑架构分层 (Layered Architecture)

系统采用分层解耦架构，强制数学逻辑与执行层隔离，并引入 L2 专项优化。

### 1.1 目录结构
```text
/defi-alm-bot
├── src/
│   ├── core/               # 核心引擎：幂等性 FSM、主循环、风控总闸
│   ├── data/               # 数据层：PriceMaster (带区块高度校验), PoolObserver
│   ├── strategy/           # 策略层：IStrategy 接口 (ATR/固定宽度)
│   ├── execution/          # 执行层：路径决策、TransactionHandler (状态核验回滚)
│   ├── math/               # 数学层：高精度 V3 公式, TickMath (Solidity 镜像算法)
│   ├── storage/            # 持久化层：aiosqlite 单例、状态快照、交易存证
│   └── utils/              # 工具类：L2 Gas (含 L1 Calldata Fee) 计算器
├── tests/                  # 验证层：Anvil Forking 测试、Math 单元测试
└── main.py                 # 启动入口 (asyncio)
```

---

## 2. 核心模块演进设计

### 2.1 状态机与幂等执行 (Core/FSM & Execution)
*   **幂等性执行 (Idempotent Execution)**：
    *   在进入 `PENDING_MINT` 或 `PENDING_SWAP` 前，系统必须记录预期的 `Nonce`。
    *   **崩溃恢复逻辑**：重启后，系统首先查询链上最新 `NFT` 列表或 `Transaction Receipt`。若预期操作已在链上完成，则状态机自动跳转至 `SUCCESS`，禁止重复发送交易。
*   **状态持久化**：事务性写入 SQLite，确保“状态变更”与“本地存证”同步。

### 2.2 数据预言机 (Data/Oracle) - 时空一致性
*   **区块高度校验 (BlockHeight Sync)**：
    *   `get_verified_price()` 获取的 `slot0`、`TWAP` 和 `Chainlink` 数据必须附带 `block_number`。
    *   若数据源之间高度偏差超过 N 个块（根据 L2 出块速度动态设定），系统进入 `REFRESH_PRICE` 等待观察期，不触发熔断但暂缓决策。
*   **双模驱动**：WSS 负责低延迟触发，HTTP 负责高可靠校准。

### 2.3 执行层 (Execution) - L2 成本模型
*   **L2 专项优化**：
    *   `utils/l2_gas_calculator.py` 必须实现 **L1 Data Fee** 估算。
    *   对于 Arbitrum，使用 `NodeInterface` 接口获取 `l1EstimatePrecision` 或通过模拟交易提取 `l1Fee` 字段。
    *   损耗评估公式：`Total_Cost = L2_Execution_Fee + L1_Calldata_Fee + Expected_Slippage`。

### 2.4 数学层 (Math/V3Math) - Solidity 1:1 镜像
*   **TickMath 镜像**：
    *   `src/math/tick_math.py` 严禁使用 Python 原生幂运算。
    *   **实现要求**：完全复刻 `TickMath.sol` 中的二进制逼近算法（使用相同幻数 `0xfffcb933bd6ad8...`），确保链下计算的 `sqrtPriceX96` 与 EVM 结果逐位对齐。
*   **精度控制**：全局 `Decimal` 精度保持在 60 位以上。

---

## 3. 持久化层 (Storage) - 审计与存证
*   **架构**：`aiosqlite` 单例。
*   **新增表设计**：
    *   `tx_audit`: 记录每个 `nonce` 对应的 `action` 和 `tx_hash`，用于幂等性核验。
    *   `state_log`: 记录 FSM 每次跳转的上下文，用于崩溃后的现场恢复。

---

## 4. 开发路标 (Roadmap v1.2)

### Sprint 1: 数学心脏与镜像算法 (Math & Precision)
*   完成 `tick_math.py`（镜像 Solidity 算法）。
*   完成 `v3_math.py`（基于镜像算法的流动性计算）。
*   **验收**：与 Uniswap V3 合约生成的 `sqrtPriceX96` 对比，误差为 0。

### Sprint 2: L2 数据基座 (Data & L2 Utils)
*   实现带高度校验的 `PriceMaster`。
*   完成支持 L1 Data Fee 预估的 Gas 计算器。

### Sprint 3: 幂等性状态机 (FSM & Storage)
*   实现具备崩溃恢复能力的 FSM 核心。
*   在 Anvil 中模拟“交易发送后断网”，验证重启后的自动状态同步。
