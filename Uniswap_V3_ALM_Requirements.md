# Uniswap V3 自动化流动性管理系统 (ALM) 工业级需求规格文档 (Final-Master)

## 1. 项目概述
本系统旨在通过 Python 构建一个自动化做市机器人，在 Uniswap V3 上执行动态流动性管理。系统针对 L2 环境优化，具备高度的数学严谨性、状态持久化能力、交易原子性及多重安全防御机制。

## 2. 核心技术规格
*   **语言与交互**：Python 3.10+，`web3.py` v6+。
*   **网络环境**：Arbitrum One / Base。
*   **数学标准**：强制使用 `decimal.Decimal`。执行层处理代币精度归一化（10^18）及幂次补偿。
*   **交易机制设计**：
    *   **双路径策略**：根据调仓规模和类型，动态选择“原子路径（Atomic）”或“拆分路径（Split）”。
    *   **Multicall 范围**：限于 `NonfungiblePositionManager` 内部操作（Decrease + Collect + Increase）。

## 3. 功能需求

### 3.1 增强型数据采集 (Data & Oracle)
*   **多源价格核验**：实时对比 `slot0`、`Pool TWAP` 和 `Chainlink`。三者偏差超限（默认 0.5%）则熔断。
*   **池子元数据适配**：自动读取 `feeTier` 对应的 `tickSpacing` 约束。

### 3.2 动态策略接口与冷启动
*   **IStrategy 接口**：支持可插拔的宽度计算逻辑（固定宽度/ATR 动态宽度）。
*   **冷启动流程 (Cold Start)**：
    1.  `APPROVAL_CHECK`：首先检查并执行对 `PositionManager` 和 `SwapRouter` 的授权。
    2.  `PORTFOLIO_CHECK`：若无现有 NFT，检查钱包余额。
    3.  `SWAP_FOR_INIT`：若比例不均，先执行兑换，再进行首笔 `MINT`。

### 3.3 状态机决策逻辑 (FSM Path Selection)
系统需根据情况选择执行路径：
*   **路径 A：原子路径 (Atomic Path)**
    *   **适用场景**：仅提取收益、微调区间或 Swap 金额极小（Price Impact < 0.01%）。
    *   **流转**：`IDLE` -> `PENDING_ATOMIC` (一次性发送 Multicall [Decrease+Collect+Increase])。
    *   **特点**：极高安全性，防价格漂移，但滑点需设严。
*   **路径 B：拆分路径 (Split Path)**
    *   **适用场景**：需要显著配平大额资产。
    *   **流转**：`IDLE` -> `PENDING_WITHDRAW` -> `PENDING_SWAP` -> `REFRESH_PRICE` (反馈修正) -> `PENDING_MINT`。
    *   **特点**：高精确度，防止因 Swap 导致 Mint 失败，但涉及多次交易。

### 3.4 执行层深度优化
*   **反馈修正环 (Feedback Loop)**：在 `Split Path` 中，Swap 确认后必须重新计算 Tick 边界。
*   **QuoterV2 接入**：所有 Swap 损耗预估强制通过 `QuoterV2` 合约。
*   **Gas 缓冲带**：设置 15% - 20% 的 Gas Buffer 以应对 L2 瞬时拥堵。

## 4. 风险控制与安全 (Safety)

### 4.1 避险模式与授权管理
*   **避险模式 (Safe Mode)**：单边极速行情触发清仓为稳定币。**必须人工恢复**。
*   **安全加固**：避险模式下支持一键 `RevokeAll` 清除合约授权。

### 4.2 熔断阈值
*   **单笔交易限额**：`MAX_SWAP_VALUE_USD`。
*   **损耗熔断**：单次调仓总损耗（Gas + Slippage + Fee）超过预估手续费收益的 $X\%$ 时，放弃本次非紧急调仓。

## 5. 仿真验证标准 (Validation)
*   **原子性验证**：在 Anvil 中验证原子路径在高频场景下的成功率。
*   **反馈修正测试**：验证在大额 Swap 引起价格大幅变动后，`REFRESH_PRICE` 状态能否准确修正 Mint 边界。
*   **压力测试**：模拟节点断网、授权失效、余额不足等极端状态下的 FSM 恢复。

## 6. 日志与绩效 (Analytics)
*   **资产归因分析**：分拆 `Fee_Income`、`HODL_Gain/Loss`、`Rebalance_Loss`。
*   **状态追踪**：通过本地 SQLite 数据库记录每个 `TokenID` 的生命周期及 FSM 状态快照。
