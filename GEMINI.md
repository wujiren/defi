# Gemini CLI Context: Uniswap V3 ALM System

This project is an industrial-grade **Uniswap V3 Automated Liquidity Management (ALM)** system built with Python. It is designed specifically for Layer 2 networks (Arbitrum, Base) and emphasizes mathematical rigor, idempotent execution, and robust risk management.

## 🛠 Project Overview
- **Language**: Python 3.10+
- **Key Stack**: `web3.py` v6+, `decimal.Decimal`, `aiosqlite`, `asyncio`.
- **Target Networks**: Arbitrum One, Base.
- **Core Goal**: Maximize fee revenue and minimize impermanent loss through dynamic position adjustment based on market volatility.

## 🏗 Architecture
The system follows a layered, decoupled architecture:
- `src/core/`: Idempotent Finite State Machine (FSM) and main event loop.
- `src/math/`: High-precision Uniswap V3 math mirroring Solidity (`TickMath.sol`).
- `src/data/`: Multi-source Oracle (Slot0, TWAP, Chainlink) with BlockHeight synchronization.
- `src/execution/`: Transaction management with dual-path logic (Atomic via Multicall vs Split).
- `src/strategy/`: Pluggable strategy interfaces (e.g., ATR-based dynamic bands).
- `src/storage/`: Async SQLite persistence for state recovery and transaction auditing.
- `src/utils/`: L2-specific gas estimation (including L1 Data Fee).

## 🚀 Key Commands
- **Initialize**: `pip install -r requirements.txt` (TODO: Create requirements.txt)
- **Run Bot**: `python main.py`
- **Testing**: `pytest` (Focus on Anvil forking and Math unit tests)
- **Local Simulation**: `anvil --fork-url <RPC_URL>`

## 📏 Engineering Standards & Conventions
- **Mathematical Precision**: 
    - Mandatory use of `decimal.Decimal` with precision >= 60.
    - `src/math/tick_math.py` must be a **1:1 mirror** of `TickMath.sol` (using binary approximation and magic numbers) to ensure bit-perfect alignment with EVM.
- **Idempotency**: All execution states must be verifiable. Before re-sending a transaction (e.g., after a crash), the system must verify the on-chain state (NFT ownership, Nonce).
- **L2 Optimization**: Gas calculations must account for the **L1 Calldata Fee**.
- **Oracle Safety**: 
    - Verify `slot0` against `TWAP` and `Chainlink`.
    - Data must be BlockHeight-synced; reject or delay if sources are desynced.
- **FSM Persistence**: Every state transition must be persisted to SQLite before execution.

## 🔄 Execution Workflow
For every task or Roadmap item, the following cycle is mandatory:
1. **Detailed Planning**: Before any execution, formulate a detailed plan and document it in a dedicated plan file (e.g., `PLAN.md` or a sprint-specific doc).
2. **Self-Review**: Upon completing a development task, perform a rigorous self-review against requirements and architecture standards.
3. **Test Strategy**: Following self-review, define a detailed testing plan (scenarios, edge cases, expected outcomes).
4. **Testing & Debugging**: Implement test files (unit/integration) and debug until all tests pass in the target environment (e.g., Anvil).
5. **Progress Update & Summary**: Update the progress in the execution plan, summarize the completion status, and wait for human review.
6. **Approval Gate**: Proceed to the next step or adjust the current implementation based on human feedback until final approval is granted.

## 📋 Development Roadmap
Each sprint below MUST strictly follow the **Execution Workflow** defined above.

1. **Sprint 1**: Math Heart (`tick_math.py`, `v3_math.py`) & Unit Tests.
2. **Sprint 2**: Data Layer & L2 Gas Utils (Oracle, WSS/HTTP).
3. **Sprint 3**: Idempotent FSM & Storage (aiosqlite).
4. **Sprint 4**: Strategy Implementation & Split Path Execution.
