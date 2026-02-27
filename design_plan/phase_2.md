# Phase 2 详细分解：核心引擎与数学对齐（修订版 v1.1）

**⏱️ 预计耗时**：3-4 周（硬核攻坚期）
**🎯 核心目标**：消除跨语言浮点数与智能合约整数运算的误差，跑通核心状态机（FSM）的前半段，完成 G1-G4 全部安全门控，并在此基础上正式通过 G5/G6 绩效门控。**同步交付 Admin Web Portal、两档仿真模式和 MtM 实时估值模块。**

> **v1.1 修订说明：**
> - 任务 1 修订：Go EVM 仿真器须同时实现"轻量仿真"与"完整仿真"两档模式（对齐 NFR 文档中的两档延迟预算）
> - 新增任务 5：Admin Web Portal（原计划无任何团队认领此模块）
> - 新增任务 6：MtM 实时估值与未领取奖励扫描（原计划完全缺失）
> - 新增任务 7：G5/G6 正式绩效评估（G1-G4 完成后才具备生产有效性，与 Phase 1 的预评估形成闭环）

---

## 任务 1：开发 Go EVM 仿真器（两档模式）*(修订)*

**主导团队**：区块链底层与执行组（Zone B）
**协办团队**：AI 与量化策略组（提供 Python 端测试用例）

这是消灭"回测陷阱"的最关键战役，必须做到位级对齐。**仿真器须同时支持两档模式，以兼顾风控完整性与执行速度。**

* **子任务 1.1：构建 StateDB 内存仿真内核**
  * **工作内容**：基于 `go-ethereum/core/vm` 包，编写轻量级的 EVM 仿真运行环境；实现 MockStateDB 的懒加载、Snapshot/Revert 隔离机制，支持 50+ Agent 并发仿真互不干扰。
  * **硬性约束**：StateDB 热缓存命中率目标 > 90%；所有金额计算使用 `math/big.Int`，禁止浮点数。

* **子任务 1.2：实现轻量仿真档（常规交易路径）**
  * **工作内容**：轻量仿真只执行不变性断言（MaxDrawdown / MaxSlippage / MaxGasMultiple），不走完整 Anvil 链上状态模拟。
  * **性能目标**：P50 < 30ms，P99 < 80ms（对应 NFR 文档轻量仿真预算）；超过 200ms 触发降级日志记录。
  * **适用范围**：所有常规交易（非首次执行、非大额、非 STAGED 状态）。

* **子任务 1.3：实现完整仿真档（大额 / 首次 / STAGED 交易路径）**
  * **工作内容**：完整仿真走全套 Anvil 主网分叉模拟，包含完整 EVM 状态变更、Gas 精确计算、链上返回值解析。
  * **触发条件（任一满足即走完整仿真）**：
    1. 单笔金额 > 策略当前分配资金 15%
    2. 该策略在当前区块首次执行
    3. 策略处于 STAGED 状态（全部交易强制走完整仿真）
  * **性能目标**：P50 < 100ms，P99 < 250ms（对应 NFR 文档完整仿真预算）；超过 600ms 取消本次决策。
  * **指标埋点**：`defi_fsm_transition_duration_ms{transition="INIT_to_SIMULATED", mode="full|lite"}` 须分档统计，不混合计算。

* **子任务 1.4：实现仿真档位路由逻辑**
  * **工作内容**：在 `ExecuteTrade` gRPC handler 中，根据请求参数自动判断应走哪一档仿真，记录档位选择原因到结构化日志中。

---

## 任务 2：G2 精度校验——位级对齐

**主导团队**：区块链底层与执行组（Zone B）
**依据**：`验证门控标准说明文档` G2 关卡

* **子任务 2.1：打通真实 gRPC 代理**
  * **工作内容**：废弃 Phase 0 的假接口，Go 端接收到 `TradeRequest` 后，将其解析为真实的 `SimulationRequest` 并输入 EVM 仿真器。

* **子任务 2.2：攻克 G2 精度校验**
  * **工作内容**：构造 500 组随机输入，将 Python 策略输出的 CLAMM 流动性计算结果与 Go 仿真器调用真实合约 `TickMath` 的结果进行对比。
  * **交付标准**：整数除法舍入（floor 除法）、TickMath 计算、uint256 边界值（0, 2^256-1）、费率复利计算等场景均须达到 **0 误差（位级对齐）**；流动性计算允许 ≤ 1 wei 误差。

---

## 任务 3：构建 G3 不变性断言与 G4 极端 Fuzzing

**主导团队**：财务风控组（断言逻辑）+ 架构与 SRE 组（Fuzzing 参数注入）

* **子任务 3.1：实现幂等状态机（FSM）与 Nonce 锁**
  * **工作内容**：基于 PostgreSQL `FOR UPDATE` 悲观锁实现 Nonce 行级锁，打通 `INIT → SIMULATED` 状态流转，强制遵循"先写 WAL 日志，后改状态"的哲学。
  * **硬性约束**：50+ Agent 并发时绝不出现 Nonce 冲突或跳号。

* **子任务 3.2：实现 G3 不变性断言（Invariant Assertions）**
  * **工作内容**：在 EVM 仿真后，捕获 StateDB 变更，强制校验：
    - `MaxDrawdown`：单次执行 U 本位跌幅 ≤ 1%
    - `MaxSlippage`：实际滑点 ≤ 声明容忍度
    - `MaxGasMultiple`：实际 Gas ≤ 估算 Gas × 3
    - `PositionLimitPct`：单策略持仓 ≤ 资金池上限比例
  * **硬性约束**：100% 场景下满足，即使 999/1000 场景通过仍判定失败，策略退回 DRAFT。

* **子任务 3.3：搭建 G4 极端 Fuzzing 测试**
  * **工作内容**：使用本地 Anvil 分叉，注入极端参数（Gas 飙升 1000 倍、流动性枯竭至 0.1%、预言机 ±90% 价格跳动、网络超时 0-30s、Nonce 冲突、链重组 1-3 区块），每类 100-200 轮，总计 1000 轮。
  * **交付标准**：1000 轮无破线（允许策略正确触发熔断拒绝发单，不允许产生超限亏损）。

---

## 任务 4：双分录会计核心引擎

**主导团队**：财务风控组（Zone B）

* **子任务 4.1：实现记账分录引擎**
  * **工作内容**：基于 `SIMULATED` 状态输出的模拟执行收据，编写将交易金额转化为 Debit/Credit 分录的逻辑；写入 `ledger_entries` 时须强校验 `sum(debit) == sum(credit)`，否则触发 SQL 事务回滚。
  * **分录类型覆盖**：`TRADE`（交易本金）、`GAS_FEE`（执行损耗，单独记录不混入交易 PnL）、`PROTOCOL_FEE`（协议手续费）须在同一数据库事务中原子写入。

* **子任务 4.2：实现账实核查（Reconciliation）**
  * **工作内容**：实现 `Reconciler`，对比内部账本余额与链上实际余额（须指定 block_number，禁止使用 `latest`）。
  * **硬性约束**：差值超过 1 wei 立即触发 `RECONCILIATION_FAILURE` 告警，自动暂停策略，零容忍。

---

## 任务 5：Admin Web Portal *(新增)*

**主导团队**：AI 与量化策略组（Web 方向开发）
**部署位置**：Zone A（`intelligence_net`，对外暴露 HTTPS 443）
**依赖**：Phase 0 的 mTLS 证书体系、任务 3 的 FSM 状态机

**为什么在 Phase 2 交付？** Phase 3 影子模式上线后，管理员需要通过 Portal 处理 `PENDING_APPROVAL` 队列，并在 G7 验收时查看影子 PnL 报告。Portal 必须在 Phase 3 开始前可用。

* **子任务 5.1：实现审批队列管理页面**
  * **工作内容**：展示处于 `PENDING_APPROVAL` 状态的交易队列，含策略 ID、触发原因（`staged_high_value` / `live_high_ratio` / `new_contract` / `invariant_boundary`）、估算价值；提供"批准"和"拒绝"操作按钮，操作结果通过 Zone B 的 `Admin-Approval-API` 传递至 Zone C。
  * **安全约束**：Web Portal 本身不持有任何签名权限，所有审批指令须通过 Zone B 中转，Portal 不直接连接 Zone C。

* **子任务 5.2：实现策略状态总览页面**
  * **工作内容**：展示所有活跃策略的生命周期状态（DRAFT / VALIDATED / EVALUATED / SHADOWED / STAGED / LIVE / PAUSED）、实时 PnL（接入 MtM 数据）、最近告警。

* **子任务 5.3：实现 Kill Switch 入口**
  * **工作内容**：设置高权限管理员专属入口（二次确认弹窗），点击后向 Zone B 发送全局断路器触发指令；界面展示执行进度（已撤销授权数 / 总数）。

* **子任务 5.4：访问控制**
  * **工作内容**：实现基于 IP 白名单的访问限制，管理员登录须二因素认证（2FA）；所有操作记录完整审计日志（操作者身份、时间戳、关联 trace_id）。

---

## 任务 6：MtM 实时估值与未领取奖励扫描 *(新增)*

**主导团队**：财务风控组（Zone B）
**依赖**：任务 4 的双分录账本、Phase 1 的 Adapter Registry（提供协议接口）

**为什么在 Phase 2 而非 Phase 3 交付？** 实时 MDD 监控是 PAUSED 自动触发的判断依据（`实时 MDD > 回测 MDD × 1.5`），而 PAUSED 状态机在 Phase 3 交付。若 MtM 模块不先于 PAUSED 状态机到位，Phase 3 的自动熔断就是空壳。

* **子任务 6.1：实现 `MtMCalculator` 每区块 NAV 更新**
  * **工作内容**：实现 `CalcNAV(strategy_id, at_block)` 方法：
    1. 从 `asset_balances` 读取已实现资产余额
    2. 从 `unclaimed_rewards` 读取未领取奖励（计入 NAV，防止风控误杀）
    3. 所有资产通过三源预言机中位价折算为 USDC（禁止使用 DEX Slot0 即时价）
    4. 减去未偿债务（借贷协议负债）
  * **更新频率**：每区块（约 12 秒）更新一次，写入 `mtm_snapshots` 表并更新 `defi_strategy_pnl_usdc` Prometheus Gauge。

* **子任务 6.2：实现 `RewardScanner` 未领取奖励定时扫描**
  * **工作内容**：定时任务（每 5-10 分钟），通过 Adapter 调用各协议合约查询待领取奖励（不产生链上交易），结果 Upsert 至 `unclaimed_rewards` 表。
  * **注意**：未领取奖励不进双分录（因为未发生实际 Transfer），但必须计入 NAV。

* **子任务 6.3：实现实时 MDD 监控**
  * **工作内容**：基于 MtM NAV 序列动态计算 `实时 MDD = (历史最高 NAV - 当前 NAV) / 历史最高 NAV`，写入 `defi_strategy_max_drawdown_pct` Prometheus Gauge，与 Phase 3 的 PAUSED 自动触发逻辑联动。

---

## 任务 7：G5/G6 正式绩效评估 *(新增)*

**主导团队**：AI 与量化策略组（Zone A）
**前置条件**：G1-G4 全部通过（须等待任务 2 和任务 3 的验收完成）

> ⚠️ 这是区别于 Phase 1 "预评估"的**正式评估**，具备生产有效性，可作为策略进入 VALIDATED 状态的凭据。

* **工作内容**：在 G1-G4 全部通过后，使用 SDK 标准数据源（非自行清洗的非标数据）重新运行 G5 历史回测和 G6 蒙特卡洛评估。
* **G5 通过标准（系统底线）**：回测时间跨度 ≥ 90 天（覆盖牛熊震荡），Sharpe Ratio ≥ 1.0，MDD ≤ 20%，Calmar Ratio ≥ 0.5，Gas 占比 ≤ 30%，资金利用率 ≥ 40%，盈亏比 ≥ 1.3。
* **G6 通过标准**：5000 轮模拟，正收益占比 ≥ 60%，中位数年化收益 ≥ 0%，最差 5% 分位回撤 ≤ 40%。
* **报告要求**：生成包含 `strategy_version_hash`、数据集哈希、随机数种子、各指标实际值/阈值对比的结构化 JSON 报告，报告中 `"evaluation_type": "OFFICIAL"` 标注，锁定内容不可篡改。

---

## 🏁 Phase 2 阶段合并与验收标准（DoD）

1. **两档仿真区分验收（新增）**：轻量仿真的 P99 延迟 ≤ 80ms（实测），完整仿真的 P99 延迟 ≤ 250ms（实测）；Grafana 中 `defi_fsm_transition_duration_ms` 指标可按 `mode="full|lite"` 分别查询。

2. **数学对齐验收（G2 Passed）**：AI 策略组提供 100 笔最复杂的历史 CLAMM 提案，在 Go EVM 仿真器中跑出的预估 Token 余额变化与真实链上历史变化**误差必须严格等于 0 wei**（流动性计算 ≤ 1 wei）。

3. **并发压测验收（Nonce 安全）**：50 个 Agent 在 1 秒内同时发出 100 笔交易，FSM 稳定排队仿真，数据库中 `nonce_locks` 必须连续无重复。

4. **极限抗压验收（G4 Passed）**：注入流动性枯竭且 Gas 暴涨 1000 倍的恶意参数，系统通过 G3 不变性断言正确拦截，FSM 状态正确回退，不产生超额亏损模拟记录。

5. **Admin Web Portal 可用**：管理员可通过 Portal 看到 `PENDING_APPROVAL` 队列，并成功执行一次批准和一次拒绝操作；Kill Switch 入口可渲染，点击后二次确认弹窗正常弹出。

6. **MtM 估值激活（新增）**：Grafana 中 `defi_strategy_pnl_usdc` 指标每区块更新，`unclaimed_rewards` 表在 10 分钟内至少被定时扫描一次，`实时 MDD` Gauge 数据可读。

7. **G5/G6 正式评估报告**：至少一个策略产出 `"evaluation_type": "OFFICIAL"` 的结构化报告，Sharpe Ratio ≥ 1.0，MDD ≤ 20%，蒙特卡洛正收益占比 ≥ 60%。此报告可作为进入 VALIDATED 状态的凭据。

8. **日志与审计就绪**：所有拒绝、拦截、仿真过程完整携带 `trace_id` 写入结构化日志；Grafana Loki 能通过 `trace_id` 检索单笔交易的全链路日志。
