# Sprint 1 Plan: Math Heart (镜像算法实现)

## 1. 任务目标
实现 Uniswap V3 的核心数学逻辑，确保链下计算结果与 EVM 链上计算完全一致。

## 2. 核心模块开发
### 2.1 `src/math/tick_math.py` (Solidity 1:1 镜像)
- **目标**：复刻 `TickMath.sol` 的所有核心逻辑。
- **关键函数**：
    - `getSqrtRatioAtTick(tick: int) -> int`: 根据 Tick 计算 `sqrtPriceX96`。
    - `getTickAtSqrtRatio(sqrtPriceX96: int) -> int`: 根据 `sqrtPriceX96` 计算 Tick。
- **实现要求**：
    - 使用 `0xfffcb933bd6ad8...` 等二进制逼近幻数。
    - 严禁使用 Python 原生 `**` 或 `math.pow`，必须使用位运算和整数算术。
    - 确保 `MIN_TICK` 和 `MAX_TICK` 与协议一致。

### 2.2 `src/math/v3_math.py` (流动性与代币转换)
- **目标**：实现流动性 (Liquidity) 与代币金额 (Amount) 的相互转换。
- **关键函数**：
    - `get_liquidity_for_amount0/1`
    - `get_amounts_for_liquidity`
- **精度要求**：全局使用 `decimal.Decimal` 处理高精度浮点数，但在与 TickMath 交互时转换为整数。

## 3. 验证策略
- **单元测试**：
    - 在 `tests/math/test_tick_math.py` 中硬编码 Uniswap V3 官方文档/合约中的已知 Tick-Price 对。
    - 随机抽取 1000 个 Tick，对比其计算出的 `sqrtPriceX96`。
- **跨语言对比**：验证计算结果是否与链上 `TickMath` 库输出逐位一致。

## 4. 进度追踪
- [x] 初始化 `src/math/__init__.py`
- [x] 实现 `tick_math.py`
- [x] 实现 `v3_math.py`
- [x] 编写并验证单元测试
