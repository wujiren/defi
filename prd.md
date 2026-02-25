# 🚀 DeFi 自动化交易基础设施平台 - 项目需求文档 (PRD)



## 一、 项目愿景与目标

打造一个对“人类与AI”均友好的DeFi策略研发与运行基建，将策略生命周期（开发->验证->回测->实盘监控）的时间从**几周缩短至几天甚至几小时**，并确保资金和逻辑的绝对安全。



*   **目标1（极简开发）：** 提供高度抽象的SDK与DSL（领域特定语言），剥离链上交互底层逻辑，使开发者和AI只需关注“交易逻辑”本身。

*   **目标2（硬核验证）：** 构建基于沙盒、本地分叉（Local Fork）和静态检查的多重验证墙，确保策略在上线前排除99%的致命漏洞。

*   **目标3（全维度评估）：** 融合链上（On-chain状态）、链下（CEX数据、宏观数据），支持历史回测、蒙特卡洛模拟和前向测试（Paper Trading）。



---



## 二、 核心功能模块划分 (Functional Requirements)



为了实现你的目标，系统需要划分为以下四大核心模块：



### 模块1：策略开发引擎 (Strategy Development Engine)

*目标：减少基础代码，方便人类与AI Agent编写。*

*   **DeFi 动作标准库 (Action Primitives)：** 封装主流协议的交互。例如：`swap(tokenA, tokenB, amount)`、`addLiquidity(pool, range, amounts)`、`borrow(asset, collateral)`。无需手写合约ABI调用。

*   **AI Agent 接口层 (LLM-Friendly API)：** 提供格式化的JSON/YAML策略配置模板，或者Python DSL，使得大语言模型能准确输出策略代码而不会产生幻觉。

*   **链上数据抽象层 (Data Abstraction)：** 提供统一的数据获取接口，如获取池子流动性、获取预言机价格、获取账户Token余额，屏蔽底层的RPC调用和解码过程。

*   **事件驱动与定时驱动框架：** 支持策略基于“特定链上事件（如某巨鲸转移资金）”触发，或基于“时间（每区块、每分钟）”触发。



### 模块2：策略安全验证系统 (Validation & Security System)

*目标：快速、系统地验证策略是否安全、稳定、可信赖。*

*   **静态代码与逻辑审查 (Static Analysis)：** 检查策略是否包含死循环、硬编码的地址错误、未处理的异常返回值等。

*   **交易预模拟 (Pre-Trade Simulation)：** 每一笔交易在广播前，必须在本地Fork节点（如Anvil/Hardhat）或通过 `eth_call` / Tenderly API 进行模拟。

    *   *拦截条件：* 模拟失败、Gas消耗异常、滑点超过阈值、授权（Approve）额度过大。

*   **状态不变性检查 (Invariant Checking)：** 允许开发者为策略设定“安全底线”。例如：“无论发生什么，每次交易后我的总资产U本位价值不得下降超过2%”，系统在模拟阶段一旦发现破坏了不变性，立刻阻断。

*   **权限与资产隔离 (Sandboxing)：** 策略运行在权限沙盒中，限制其只能操作分配给该策略的特定资金，防止单一策略Bug导致整个金库清零。



### 模块3：多维综合评估引擎 (Multi-dimensional Evaluation Engine)

*目标：多数据源、多情景下的策略表现评估。*

*   **高保真历史回测 (High-Fidelity Backtesting)：**

    *   连接Archive Node（归档节点），获取历史Tick级数据。

    *   精准的Gas费模型、滑点模型和DEX交易手续费模型。

*   **蒙特卡洛模拟 (Monte Carlo Simulation)：**

    *   针对借贷/LP策略，生成未来成千上万种可能的价格路径（如使用几何布朗运动GBM），测试策略在极端行情下的**无常损失（IL）**和**清算概率**。

*   **压力测试 (Stress Testing)：**

    *   模拟极端DeFi场景：如USDC脱锚、预言机价格短时间闪崩/暴拉（Flash Crash）、网络极度拥堵（Gas飙升100倍）。

*   **影子测试 / 模拟盘 (Shadow Routing / Paper Trading)：**

    *   策略连接实时主网数据源运行，但交易指令只发送到本地Fork节点（随主网实时推进更新），不产生真实Gas成本，评估实时表现。



### 模块4：稳定执行与路由模块 (Execution & Routing)

*为了让策略在真实环境稳定运行的基础设施。*

*   **智能RPC路由：** 自动检测节点延迟和健康度，发生故障时无缝切换备用RPC。

*   **MEV 防护与执行优化：** 集成 Flashbots / 私有内存池（Private Mempool），防止策略交易被三明治攻击（Sandwich Attack）。

*   **动态 Gas 管理：** 自动处理交易卡主（Stuck TX）的情况，根据网络情况自动加速（Bump Fee）或取消交易。



---



## 三、 常见状况与应对机制 (Edge Cases & Fallbacks)



在DeFi中，“稳定可信赖”不仅取决于策略逻辑，更取决于对异常情况的处理。你的基础设施必须能自动应对以下状况：



| 常见状况 (Edge Cases) | 系统应具备的应对机制 (Handling Mechanisms) |

| :--- | :--- |

| **RPC 节点宕机/限流/不同步** | 多RPC节点轮询负载均衡；节点高度健康检查（剔除落后于主网的节点）。 |

| **主网极度拥堵 (Gas 费飙升)** | 设定最大Gas接受阈值；若预期收益无法覆盖Gas成本，系统自动取消本次策略执行。 |

| **流动性枯竭/滑点巨大** | 交易前获取链上实时流动性进行深度测算；强制所有Swap操作带上严格的 `min_amount_out` 滑点保护。 |

| **预言机失效或被操纵** | 引入多源价格校验（如 Chainlink + Uniswap V3 TWAP + Binance CEX 价格），价格偏差超过 x% 时触发**熔断机制（Circuit Breaker）**，暂停交易。 |

| **交易一直未打包 (Pending)** | Nonce 管理器；超时自动使用更高Gas的相同Nonce发一笔金额为0的转账覆盖，或直接提高Gas重发（Speed up）。 |

| **底层协议被黑 / 遭遇闪电贷攻击** | 监听核心协议（如Aave, Uniswap）的异常事件（如TVL骤降），触发“一键撤资/平仓（Emergency Unwind）”逻辑。 |

| **代币发生Rebase或黑名单机制** | (如USDT/USDC黑名单、转账扣税代币)：SDK底层强制校验 `balanceOf` 前后变化，以真实到账金额为准，而非单纯依赖 `amount` 参数。 |



---



## 四、 技术栈建议 (Technology Stack Suggestions)



*   **智能合约/链上模拟层：** **Foundry (Rust/Solidity)**。Foundry的 `anvil` 极度适合做Local Forking，`forge-std` 适合写测试。

*   **策略执行与SDK层：** **Python** 或 **Rust**。Python适合AI Agent生成代码（有大量现成库如 `web3.py`）；Rust适合要求极低延迟的高频策略（如 `alloy` 或 `ethers-rs`）。

*   **数据存储：** **ClickHouse** 或 **TimescaleDB**（处理海量链上历史Tick数据、K线数据），**Redis**（处理缓存和实时状态）。

*   **基础设施服务：** **Tenderly**（用于交易前的云端模拟和监控），**The Graph / Envio**（用于历史事件和状态索引）。



---



## 五、 项目实施路径图 (Roadmap)



建议分阶段迭代你的基础设施，不要一开始就追求大而全：



*   **阶段 1: 最小可行性基建 (MVP) - 核心在“跑通”**

    *   搭建Python/TypeScript基础SDK，封装Swap、Approve、Transfer等基础动作。

    *   接入单一数据源，实现一个基于历史K线的简单回测框架。

    *   实现AI Agent的Prompt模版，让AI能输出符合SDK规范的策略代码。

*   **阶段 2: 注入“安全”基因 - 核心在“防死”**

    *   引入本地Fork环境（Anvil），所有策略交易必须经过本地模拟。

    *   实现Gas估算模块和滑点保护强制校验。

    *   搭建Paper Trading（模拟盘）环境。

*   **阶段 3: 复杂环境模拟 - 核心在“评估”**

    *   引入蒙特卡洛模拟库。

    *   建立多预言机、多流动性池的状态快照系统，支持复杂DeFi协议（如杠杆挖矿、期权）的回测。

    *   实现RPC动态路由和交易卡住自动处理机制。

*   **阶段 4: 生产级与全自动化 - 核心在“无人值守”**

    *   接入Flashbots防MEV。

    *   完成一键紧急平仓（Panic Button）系统。

    *   策略全生命周期AI自动化（AI负责发现机会 -> 编写策略 -> 平台自动验证 -> 回测通过 -> AI决定上线 -> 平台实盘监控）。
