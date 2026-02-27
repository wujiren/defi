# Phase 4 详细分解：安全装甲与飞轮迭代（修订版 v1.1）

**⏱️ 预计耗时**：4-5 周（含外部安全审计与真实资金拨测）
**🎯 核心目标**：实现物理级别的硬件签名防线，上线防夹路由，部署链上断路器，完成外部安全审计，并最终闭合从"交易损耗"到"大模型 Prompt"的 AI 自我学习飞轮。

> **v1.1 修订说明：**
> - 任务 1 补充：Kill Switch 绿色通道的开发内容（绕过审批阈值的紧急提款路径，原计划遗漏）
> - 任务 1 补充：STAGED 绝对阈值 $10,000 与 LIVE 相对阈值 15% 的差异化逻辑代码细节
> - 任务 3 补充：Adapter Registry 与时间锁的联动（新注册协议须走 24h 时间锁，原计划缺失）
> - 任务 4 补充：AI 飞轮的 60 秒 SLA 要求（对齐 `6_4_AI反馈飞轮.md` 中的时效定义）

---

## 任务 1：Zone C 物理签名与灰度审批流

**主导团队**：极高安全组（Zone C）
**外部协作**：第三方安全审计机构（如 Trail of Bits / Zellic）

废弃 Phase 0 的假签名服务，将生杀大权真正移交给硬件和管理员。

* **子任务 1.1：上线 `Whitelist-Enforcement-Proxy`**
  * **工作内容**：在 Zone C 部署真实的白名单拦截代理。
  * **不信任原则（硬性约束）**：该代理须接收来自 Zone B 的**未签名明文 Payload**，在 Zone C 内网独立解析 `to`（目标合约地址）和 `data[:4]`（函数选择器），调用 Adapter Registry 的 `IsAllowedAddress()` 和 `IsAllowedSelector()` 校验，**自行计算 Keccak256 Hash** 后再提交签名，绝不盲签 Zone B 传来的 Hash 值。
  * **Adapter Registry 为唯一来源**：合约白名单不允许在 Proxy 内部硬编码，必须通过 `AdapterRegistry.GetAllowedAddresses()` 接口动态读取（联动 Phase 1 任务 3 的 Registry）。

* **子任务 1.2：对接硬件 MPC 签名**
  * **工作内容**：将白名单代理与真实的物理 HSM / 硬件级 MPC 节点对接，完成 M-of-N 分片签名算法集成；中间态签名分片在内存中处理，完成后立即销毁，不落盘。
  * **审计日志**：每次签名操作须向 HSM 不可篡改日志写入：`timestamp`、`trace_id`、`target_address`、`method_selector`、`signer_node_ids`。

* **子任务 1.3：实现 STAGED / LIVE 差异化大额审批引擎**
  * **工作内容**：在 Zone B 的发单拦截器中，实现基于策略生命周期状态的差异化拦截。
  * **STAGED 阶段**（保守保护，绝对金额）：单笔交易价值 > $10,000 USD（以三源预言机中位价折算），自动推进至 `PENDING_APPROVAL`，暂停 FSM，等待管理员多签放行。
  * **LIVE 阶段**（效率导向，相对比例）：满足以下**任一条件**触发审批：
    1. 单笔金额 > 策略当前分配资金 15%
    2. 首次调用 Adapter Registry 中的新合约地址（策略上线以来未曾交互过）
    3. 单笔金额 > 当前不变性边界的 50%（接近风控底线的大额操作）
  * **超时处理**：`PENDING_APPROVAL` 超过 30 分钟未有管理员响应，FSM 自动推进至 `FAILED`，触发 P1 级告警提醒管理员处理队列积压。

* **子任务 1.4：实现 Kill Switch 绿色通道** *(补充)*
  * **工作内容**：全局断路器触发后，允许紧急提款类交易（`IS_KILL_SWITCH=TRUE` 标记）**跳过所有审批阈值**（包括大额多签审批和时间锁）直接进入 Zone C 签名流程；Whitelist-Enforcement-Proxy 在识别到 Kill Switch 标记时，仅做最小化校验（目标合约是否为已知的退出合约，函数选择器是否为 `withdraw` / `removeLiquidity` 等安全函数），不做金额阈值拦截。
  * **安全约束**：`IS_KILL_SWITCH` 标记只能由 Kill Switch 控制器（`global_kill_switch` 服务）写入，不允许策略代码或 Zone A 自行设置。标记使用防篡改签名，Zone C 须校验签名来源。
  * **性能目标**：Kill Switch 触发至所有协议 `Approve` 额度清零 ≤ 60 秒（NFR 要求）。

---

## 任务 2：MEV 防夹路由与确定性广播

**主导团队**：区块链底层与执行组（Zone B，含专职 MEV 研究员）

在黑暗森林中保护交易不被恶意 MEV 机器人（夹子）收割。

* **子任务 2.1：开发独立的 `MEV Router` 容器**
  * **工作内容**：监听 FSM 中处于 `SIGNED` 状态的任务；根据交易价值路由：
    - 高价值交易（> `mev_private_route_threshold` 配置项）→ Flashbots 私有中继
    - 低价值交易 → 公共 Mempool（速度优先）
  * **Flashbots 流程**：调用 `eth_callBundle` 预仿真（失败则不消耗 Gas），成功后通过 Flashbots Relay 发送；最多重试 3 个区块，3 次未被打包后静默失败（无 Gas 损失）并降级至公共路由。

* **子任务 2.2：动态 Gas Tip 竞价策略**
  * **工作内容**：MEV 研究员主导，编写算法根据当前区块 BaseFee 和策略预期利润动态计算 Miner Tip（参考公式：`利润 × 10% + BaseFee × 1.5`，可配置），确保 Bundle 打包成功率。
  * **Gas 加速阶梯**：交易进入公共 Mempool 后，每个区块未确认则按 10% 增量提高 Gas Price，上限为初始估算的 3 倍。超过 3 倍后停止加速，触发告警。

* **子任务 2.3：FSM 幂等性保障**
  * **工作内容**：MEV Router 须保证幂等（同一 `trace_id` 多次调用只广播一次），将 `txHash` 和广播状态回写 FSM；广播成功后更新 `trade_tasks.status = BROADCASTED`，记录广播时间戳和使用的路由类型（`private|public`）。

---

## 任务 3：断路器、时间锁与治理风险哨兵

**主导团队**：财务风控组与安全组

实现系统的紧急制动装置，防范外部协议遭黑客攻击或恶意治理提议。

* **子任务 3.1：部署 24 小时 `Admin Timelock`**
  * **工作内容**：编写并部署链上时间锁合约，接管全局风控参数（白名单、最大滑点阈值、资金比例上限、Flashbots 门槛配置）的修改权限；修改提交后强制等待 24 小时，期间 Guardian 多签账号拥有一票否决权（Veto），Veto 操作本身不受时间锁约束。
  * **⚠️ 时间锁边界（与策略状态转换的区别）**：24 小时时间锁仅适用于**全局风控参数修改**；策略状态转换（如 STAGED → LIVE、管理员批准大额交易）不受时间锁约束，审批通过后立即生效。

* **子任务 3.2：Adapter Registry 与时间锁联动** *(补充)*
  * **工作内容**：Phase 1 建立的 Adapter Registry 在开发环境允许直接注册新协议；生产环境须补接时间锁合约：新协议的 `Register()` 操作提交后，24 小时内 Guardian 未 Veto，才自动生效并写入 Registry。已注册协议的状态变更（如 `ACTIVE → HIGH_RISK`）由治理哨兵自动触发，不受时间锁约束（风险标记须实时生效）。

* **子任务 3.3：部署 `Global Kill Switch`**
  * **工作内容**：编写紧急撤退脚本（预生成 Revoke 交易模板），实现：
    1. 冻结 Zone B FSM（停止接受新的 `ExecuteTrade` 请求）
    2. 并行广播预生成的 `Approve(0)` Revoke 模板（按协议优先级排序，优先撤销高价值协议授权）
    3. 触发各协议的紧急提款（调用 Adapter 的 `emergency_exit()`）
    4. 向全员发送 P0 告警
  * **性能目标**：≤ 60 秒完成所有协议 `Approve` 额度清零（可通过预生成交易模板、批量广播优化）。

* **子任务 3.4：开发 `Governance Sentinel` 治理风险哨兵**
  * **工作内容**：监控 Snapshot.org API（每 5 分钟）、Tally.xyz API（每 5 分钟）、链上 Governor 合约事件（每区块实时）；发现重大参数修改时，调用 Adapter Registry 的 `SetStatus()` 接口：
    - CRITICAL 事件（紧急暂停 / 合约迁移）→ 立即设为 `SUSPENDED`，触发涉及该协议的所有策略自动 PAUSED（R6 规则）
    - HIGH 事件（关键参数 > 20% 变化）→ 设为 `HIGH_RISK`，暂停新建头寸
    - 提案否决 / 到期 → 自动恢复为 `ACTIVE`

---

## 任务 4：AI 审计闭环与进化飞轮

**主导团队**：AI 与量化策略组（Zone A）& 财务风控组（Zone B）

让系统不仅能执行，还能从每一次亏损中学习。

* **子任务 4.1：计算并投递审计结果（60 秒 SLA）** *(补充)*
  * **工作内容**：财务组在交易 `CONFIRMED` 后：
    1. **软滑点计算**：对比 `TradeRequest` 中的 `proposalPrice`（策略发单时的预期价格）与链上收据实际成交价，计算滑点 bps，写入 `ledger_entries`（`entry_type = SLIPPAGE`）
    2. **毒性流检测**：在确认区块 +10 区块后，检查价格是否持续对策略不利方向移动，若持续 > 配置阈值则标记 `ToxicFlowDetected`
    3. **异步投递**：上述结果通过 Redis Stream `stream:internal:accounting` 推送给 Zone A 的 AI-Feedback-Service
  * **时效硬约束（对齐 `6_4_AI反馈飞轮.md`）**：交易 CONFIRMED 后 **60 秒内**必须完成软滑点计算并写入 Redis Stream，超过 60 秒触发 P1 级告警。毒性流分析因需等待 +10 区块（约 120 秒），允许在 CONFIRMED + 180 秒内完成。

* **子任务 4.2：打通 Vector DB 长期记忆**
  * **工作内容**：AI 组接收 Redis Stream 中的 `FeedbackEvent`，将执行上下文（市场状态特征向量：波动率、流动性深度、Gas 价格、水位线滞后）+ 执行结果（软滑点、毒性流标记、FSM 最终状态）向量化，存入 Vector DB（embedding 须携带完整 `trace_id` 和 `strategy_id`）。

* **子任务 4.3：Prompt 动态生成机制**
  * **工作内容**：策略进入下一次迭代周期时，AI Feedback Service 自动以当前市场特征向量检索 Vector DB，召回相似历史场景（Top-K，K=5），将历史失败教训拼装为优化 Prompt，例如："在波动率 > 3%、流动性 < 5M 的场景中（trace_id: xxxxxx），上次 CLAMM 策略遭遇 30% 实际滑点，建议本次采用 TWAP 拆单或放弃该区块发单"。
  * **DEPRECATED 策略利用**：退役策略的全量历史数据（Phase 3 任务 4.4 归档的数据）作为负样本批量写入 Vector DB，增强飞轮的学习多样性。

---

## 🏁 Phase 4 阶段合并与验收标准（DoD）

1. **外部安全审计闭环**：第三方审计团队出具官方报告，确认 Zone C `Whitelist-Enforcement-Proxy` 的明文 Hash 逻辑无绕过漏洞（重点验证 `IS_KILL_SWITCH` 标记的防伪造机制），链上时间锁合约无提权漏洞，Adapter Registry 的 `IsAllowedAddress()` 无注入漏洞。

2. **Kill Switch 演练通过**：在测试网模拟单节点被黑场景，系统能够在 **60 秒内**通过 Kill Switch 绿色通道成功将 10 个协议的 `Approve` 额度清零；Admin Web Portal 展示执行进度实时更新。

3. **防夹能力实证**：MEV Router 成功在实盘灰度阶段构造出 10 笔 Flashbots Bundle，链上浏览器确认交易未经过公共 Mempool，未被三明治机器人夹击。

4. **时间锁与 Registry 联动验收**：提交一个新协议的注册请求，确认在 24 小时等待期内 `IsAllowedAddress()` 对新协议地址返回 `false`；等待期结束后自动返回 `true`；Guardian 在等待期内执行 Veto，确认注册被中止。

5. **AI 飞轮 60 秒 SLA 验收**：执行一笔真实交易并确认，监控 Grafana 指标，确认软滑点数据在 60 秒内写入 Redis Stream `stream:internal:accounting`；Vector DB 在 300 秒内完成向量化存储。

6. **STAGED → LIVE 全流程验收**：走完 STAGED 阶段真实小额灰度审批，包括：大额审批弹窗在 Admin Web Portal 正确触发、批准后 FSM 正确推进、Flashbots 私有路由正确使用。第一笔由 AI 决策、经过硬件 MPC 签名、通过防夹私有路由广播的真实主网交易成功落地。系统进入 LIVE 状态。
