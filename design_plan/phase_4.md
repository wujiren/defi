# Phase 4：安全装甲与飞轮迭代（修订版 v2.0）

**⏱️ 预计耗时**：4–5 周（含外部安全审计与真实资金拨测）
**🎯 核心目标**：实现物理级别的硬件签名防线，上线防夹路由，部署链上断路器，完成外部安全审计，并最终闭合从"交易损耗"到"大模型 Prompt"的 AI 自我学习飞轮。

> **v2.0 修订说明（相较 v1.1）**
> - 新增：本阶段在整体架构中的位置图
> - 新增：各任务架构定位说明（📐）
> - 新增：各任务禁止事项（⛔）
> - 新增：各子任务输入/输出规范
> - 新增：任务依赖关系图
> - 新增：DoD 验收方法与责任人
> - 新增：外部依赖降级预案（Flashbots 长期不可用、Snapshot/Tally 持续异常）
> - 补充：Kill Switch 绿色通道的防伪造校验细节
> - 补充：STAGED 绝对阈值与 LIVE 相对阈值差异化逻辑代码级描述
> - 补充：AI 飞轮 60 秒 SLA 要求

> **全局参考**：接口所有权表、争议仲裁路径、环境隔离策略、密钥证书生命周期（MPC 密钥仪式须在本阶段 Week 1 完成）、灾难恢复预案均定义在 Phase 0 §A–§D，本阶段不重复。

---

## 📍 本阶段在整体架构中的位置

```
本阶段目标：废弃 Zone C 假签名，上线真实 MPC；激活 MEV 防夹路由；
           部署时间锁 + Kill Switch + 治理哨兵；闭合 AI 飞轮；第三方审计签核

[Zone A]                      [Zone B]                              [Zone C]
──────────────────────        ──────────────────────────────────    ──────────────────────────
Python Agent(已有)            ★ MEV Router(独立容器)                ★ Whitelist-Enforcement-Proxy
Admin Portal(已有)            ★ 差异化大额审批引擎                   ★ 真实 MPC 节点(M-of-N 签名)
★ AI 反馈飞轮(Vector DB)       ★ Governance Sentinel(治理哨兵)       ★ Kill Switch 绿色通道
★ DEPRECATED 策略负样本投递     ★ 24h Admin Timelock
                              ★ Global Kill Switch 控制器
                              ★ FSM 完整路径(SIGNED→CONFIRMED)
                              ★ Flashbots Bundle 广播
                              ★ Recovery Manager 完整版(真实重广播)
                              EVM 仿真器/PAUSED 状态机(已有)
                              Adapter Registry(已有)

★ = 本 Phase 新增    括号 = 上阶段已有
环境：PROD 主网环境，真实资金；Zone C 须在独立物理机部署
```

**本阶段不涉及**：基础设施架构调整（Phase 0–3 已完成）、新协议的大规模接入（须走时间锁，按需分批）。

---

## ⛔ 本阶段全局禁止事项

```
签名安全类：
  ⛔ 禁止 Zone C 接收来自 Zone B 的 Hash 值进行签名（必须接收明文 Payload，自行计算 Keccak256）
  ⛔ 禁止合约白名单在 Whitelist Proxy 内部硬编码（必须从 Adapter Registry 动态读取）
  ⛔ 禁止 IS_KILL_SWITCH 标记由策略代码或 Zone A 自行设置（只能由 Kill Switch 控制器签发）
  ⛔ 禁止 MPC 中间态签名分片落盘（内存中处理，完成后立即销毁）

时间锁类：
  ⛔ 禁止将策略状态转换（STAGED → LIVE）混淆为时间锁作用范围（时间锁仅管全局风控参数）
  ⛔ 禁止生产环境新协议注册跳过时间锁（开发环境允许直接注册，生产环境必须走 24h 等待期）
  ⛔ 禁止 Guardian 的 Veto 操作被时间锁延迟（Veto 本身不受时间锁约束，须即时生效）

MEV 类：
  ⛔ 禁止 MEV Router 修改 FSM 状态（Router 只负责广播，状态变更由 FSM 自己处理）
  ⛔ 禁止高价值交易在 Flashbots 降级后不记录降级日志（降级原因须写结构化日志供事后审计）

AI 飞轮类：
  ⛔ 禁止软滑点计算超过 CONFIRMED + 60 秒才写入 Redis Stream（超时触发 P1 告警）
  ⛔ 禁止 Vector DB 存储时省略 trace_id 字段（须携带完整 trace_id，确保可追溯）
```

---

## 外部依赖降级预案

> **为什么在 Phase 4 专节处理？** Phase 4 首次涉及真实资金广播，Flashbots 不可用和治理哨兵数据源长期异常是两个直接影响资产安全的外部风险。

### 降级预案 C：Flashbots Relay 长期不可用

**风险描述**：MEV Router 的高价值交易通过 Flashbots 私有中继路由，以防止三明治攻击。若 Flashbots 长期宕机或接口变更，高价值交易将不得不降级至公共 Mempool，面临 MEV 夹击风险。短期降级（< 3 个区块未被打包）已有处理逻辑，但长期不可用时策略参数需要联动调整。

**降级策略**：

```
触发条件：Flashbots Bundle 连续 3 个区块未被打包，自动降级至公共 Mempool（已有逻辑）

长期不可用应对（降级持续 > 30 分钟）：
  Level 1（30 分钟–2 小时）：
    → 触发 P1 告警，通知 MEV 研究员评估
    → 自动将 mev_private_route_threshold 提高 50%
      （例如：原阈值 $5,000 提升至 $7,500，减少走公共 Mempool 的高价值交易数量）
    → 单笔交易规模上限降至当前阈值的 50%（拆单保护，减少单笔在公共 Mempool 的暴露）

  Level 2（> 2 小时）：
    → 触发 P0 告警
    → 暂停新建仓操作（现有头寸维护继续，但不开新仓）
    → MEV 研究员启动备用私有中继（如 bloXroute）的紧急对接流程
    → 若备用中继 1 小时内未就绪，建议管理员评估是否暂停全部 LIVE 策略

  恢复验证：Flashbots 恢复后，发送测试 Bundle 确认可被打包，
            记录到 flashbots_health_log 表，告警自动 resolve
```

**Prometheus 监控**：`defi_mev_private_route_success_rate`（成功率 < 50% 触发 P1），`defi_flashbots_bundle_latency_ms` Histogram。

---

### 降级预案 D：Snapshot / Tally API 持续异常（生产版）

> 本预案是 Phase 3 降级预案 B 的生产强化版，Phase 3 中已完成基础逻辑和压力测试，Phase 4 在此基础上增加以下内容：

```
生产环境增强措施：
  → 在 Go 侧本地缓存最近 48 小时的已知提案数据（SQLite 或 PostgreSQL），
    API 失败时以缓存数据作为治理状态基准（防止频繁误报）
  → 为 Snapshot.org 和 Tally.xyz 分别配置独立的 HTTP 客户端，
    含 retry 逻辑（3 次重试，指数退避）、超时设置（10s）和 User-Agent 轮换
  → 对于 API 限流（429）：自动降低轮询频率至每 15 分钟，写告警日志但不触发协议状态变更
  → 对于 API 响应结构变更（解析失败）：
      立即触发 P0 告警（需要 MEV/安全工程师介入修复解析逻辑）
      在修复期间依赖链上 Governor 合约事件作为唯一数据源

审计要求：
  → 所有基于 API 数据的 Adapter Registry 状态变更，须在审计日志中记录数据来源
    （"source": "snapshot_api" | "tally_api" | "onchain_governor" | "cache" | "manual"）
  → 每周生成治理哨兵健康报告，统计各数据源的可用率和状态变更次数
```

---

## 任务 1：Zone C 物理签名与差异化审批流

### 📐 架构定位

| 项目 | 内容 |
|---|---|
| **架构位置** | Zone C（fortress_net），系统物理级别的最后签名防线 |
| **对应架构文档** | `5.1_MPC签名与白名单拦截.md` §2–§3；`策略生命周期定义说明文档.md` §3.5/§3.6 大额审批阈值 |
| **在请求链路中的位置** | Zone B 发单拦截器 → **[差异化审批逻辑]** → Zone C `Whitelist-Enforcement-Proxy` → **[MPC 签名]** → txHash 返回 FSM |
| **上游依赖** | Phase 0 §C MPC 密钥仪式（须在本任务启动前完成）；Phase 1 Adapter Registry；Phase 3 PAUSED 状态机（大额触发 PENDING_APPROVAL）|
| **下游影响** | 废弃 Phase 0 假签名服务；解锁 Phase 4 全部真实广播能力 |
| **接口 Owner** | 极高安全组（Zone C）；区块链底层组（Zone B 审批拦截器） |

### ⛔ 本任务禁止事项

```
⛔ 禁止 Whitelist Proxy 接收 Zone B 传来的 Hash 值进行签名（必须明文 Payload，自行计算 Keccak256）
⛔ 禁止合约白名单在 Proxy 内部硬编码（必须通过 AdapterRegistry.GetAllowedAddresses() 动态读取）
⛔ 禁止 MPC 中间态签名分片写磁盘（内存处理，完成后立即销毁）
⛔ 禁止 IS_KILL_SWITCH 标记的防伪造签名由区块链底层组自行生成（密钥由安全组独占）
⛔ 禁止跳过 Key Ceremony 直接上线 MPC（Phase 0 §C.3 规定的仪式流程必须完整执行并留存录像）
```

---

### 子任务 1.1：上线 Whitelist-Enforcement-Proxy

**输入：**
- `5.1_MPC签名与白名单拦截.md` §2 Proxy 校验逻辑
- Phase 1 Adapter Registry `IsAllowedAddress()` / `IsAllowedSelector()` 接口
- Phase 0 §C 产出的 Zone C mTLS 服务端证书

**输出：**
- `services/whitelist_proxy/main.go`：Zone C 部署，mTLS 监听
- 收到 Zone B 明文 Payload 后，Zone C 内网独立执行：
  1. 解析 `to`（目标合约地址）和 `data[:4]`（函数选择器）
  2. 调用 `AdapterRegistry.IsAllowedAddress(to, chain_id)` → 不在白名单则拒绝，返回 `WHITELIST_VIOLATION`
  3. 调用 `AdapterRegistry.IsAllowedSelector(selector)` → 不允许的函数则拒绝
  4. 自行计算 `Keccak256(Payload)` → 提交 MPC 签名
- 每次拒绝写审计日志：`trace_id`、`rejected_address`、`reject_reason`
- 指标：`defi_security_whitelist_violations_total{reason}` Counter

**不产出（明确排除）：**
- 不包含 MPC 签名本体（由子任务 1.2 负责）
- 不持有任何策略代码或 Zone A 的通信凭证

---

### 子任务 1.2：对接硬件 MPC 签名

**输入：**
- Phase 0 §C.3 Key Ceremony 完成证明（签署方案确认书 + 录像存证）
- M-of-N 分片签名方案（如 3-of-5）
- N 台独立物理机已就绪

**输出：**
- MPC 节点部署：N 台物理机各运行签名分片节点，网络仅与 Whitelist Proxy 通信
- 集成测试：Whitelist Proxy 发起签名请求 → M 个节点参与 → 合成完整签名 → 返回 txHash
- HSM 审计日志：每次签名写不可篡改日志（`timestamp`、`trace_id`、`target_address`、`method_selector`、`signer_node_ids`）
- 中间态分片验证：确认签名完成后中间材料不可在磁盘找到（`find /mpc-node -name "*.shard" | wc -l` == 0）

---

### 子任务 1.3：实现 STAGED / LIVE 差异化大额审批引擎

**输入：**
- `策略生命周期定义说明文档.md` §3.5（STAGED 绝对阈值）、§3.6（LIVE 相对阈值）
- Phase 2 任务 5 Admin Web Portal `Admin-Approval-API`

**输出：**
- `pkg/approvals/threshold_engine.go`：在 Zone B 发单拦截器中，基于策略生命周期状态差异化拦截：

**STAGED 阶段**（保守保护，绝对金额）：
```go
// 单笔交易价值 > $10,000 USD（三源预言机中位价折算）→ PENDING_APPROVAL
if lifecycle == STAGED && estimatedValueUSD.Gt(big.NewInt(10_000)) {
    return triggerApproval("staged_high_value", estimatedValueUSD)
}
```

**LIVE 阶段**（效率导向，相对比例 + 合约新鲜度 + 不变性边界）：
```go
// 三个条件任一满足
if lifecycle == LIVE {
    if amount.Gt(allocatedCapital.Mul(big.NewFloat(0.15))) {
        return triggerApproval("live_high_ratio", amount)                 // 条件 1：> 15% 分配资金
    }
    if !registry.HasInteracted(strategy_id, to_address) {
        return triggerApproval("new_contract", to_address)                // 条件 2：首次调用新合约
    }
    if amount.Gt(invariantBoundary.Mul(big.NewFloat(0.50))) {
        return triggerApproval("invariant_boundary", invariantBoundary)   // 条件 3：>50% 不变性边界
    }
}
```

- PENDING_APPROVAL 超过 30 分钟未有响应：FSM 自动推进至 FAILED，触发 P1 告警 `ApprovalQueueBacklog`
- 触发原因写结构化字段：`trigger_reason ∈ {staged_high_value | live_high_ratio | new_contract | invariant_boundary}`

---

### 子任务 1.4：实现 Kill Switch 绿色通道

**输入：**
- `5.3_全局断路器与时间锁.md` §1.3 Kill Switch 执行序列
- 安全组独占的 Kill Switch 控制器签名密钥

**输出：**
- Kill Switch 控制器（仅安全组部署）：`IS_KILL_SWITCH=TRUE` 标记 + 防伪造签名（使用安全组专用 Ed25519 密钥签署 Payload）
- Whitelist Proxy 增强逻辑：识别到 `IS_KILL_SWITCH=TRUE` 标记时：
  1. 验证防伪造签名（使用对应公钥，公钥硬编码在 Zone C，不可远程更新）
  2. 验证通过后：**跳过所有金额阈值校验和大额审批逻辑**（但仍校验目标合约是否为已知退出合约）
  3. 函数选择器仅允许 `withdraw`、`removeLiquidity`、`exit` 等已知安全退出函数
  4. 签名后立即广播，不进入 PENDING_APPROVAL 队列
- 性能目标：Kill Switch 触发至所有协议 Approve 额度清零 ≤ 60 秒
- 安全约束：策略代码和 Zone A 任何组件**不可访问** Kill Switch 控制器的签名密钥，密钥仅存在于安全组控制的系统中

---

## 任务 2：MEV 防夹路由与确定性广播

### 📐 架构定位

| 项目 | 内容 |
|---|---|
| **架构位置** | Zone B 独立容器（`mev-router`），与 Go-Execution-Engine 物理分离 |
| **对应架构文档** | `3.4_MEV路由与防夹执行.md` §2–§3；`1.1_三区网络拓扑.md` DDR-2（MEV Router 独立容器设计决策） |
| **在请求链路中的位置** | FSM 推进至 SIGNED → **MEV Router** → Flashbots 私有中继（高价值）/ 公共 Mempool（低价值）→ tx_hash 回写 FSM |
| **上游依赖** | 任务 1（Zone C 真实签名，提供真实 signed_tx）；Phase 3 Recovery Manager（重广播场景） |
| **下游影响** | 阻塞真实主网广播；Phase 3 Recovery Manager 完整版依赖本模块真实广播能力 |
| **接口 Owner** | 区块链底层与执行组（含专职 MEV 研究员） |

> **为什么 MEV Router 是独立容器？** 见 DDR-2：Flashbots 连接需要访问公共互联网，Go-Execution-Engine 应限制直接公网访问以缩小攻击面，将 MEV Router 独立后可单独配置出站网络规则。

### ⛔ 本任务禁止事项

```
⛔ 禁止 MEV Router 修改 FSM 状态（Router 只负责广播，只返回 txHash 和 error，状态变更由 FSM 处理）
⛔ 禁止同一 trace_id 被 MEV Router 广播两次（必须在广播前检查 trade_tasks 的广播状态）
⛔ 禁止高价值交易在 Flashbots 降级后不记录降级原因（必须写结构化日志 + 更新路由类型字段）
⛔ 禁止 Gas 加速阶梯超过初始估算 3 倍（超过上限后停止加速，触发告警而不是无限加价）
```

---

### 子任务 2.1：开发独立 MEV Router 容器

**输入：**
- `3.4_MEV路由与防夹执行.md` §2 路由逻辑
- `mev_private_route_threshold` 配置项
- Flashbots Relay API 端点

**输出：**
- `services/mev_router/main.go`：独立 Go 服务，监听来自 FSM 的 SIGNED 任务
- 路由逻辑：
  - 高价值（> `mev_private_route_threshold`）→ Flashbots：先调 `eth_callBundle` 预仿真，失败不消耗 Gas，成功后通过 Relay 发送；最多等待 3 个区块，3 次未被打包后降级至公共路由
  - 低价值 → 公共 Mempool（速度优先）
- 广播成功后回写 FSM：`trade_tasks.status = BROADCASTED`，记录 `broadcast_route` 字段（`private | public | flashbots_degraded`）
- 幂等保障：广播前检查 `trade_tasks.broadcast_at IS NOT NULL`，已广播的不重复发送
- **降级预案 C 联动**：接入 Flashbots 健康监控逻辑，触发 Level 1/2 行为

---

### 子任务 2.2：动态 Gas Tip 竞价策略

**输入：**
- MEV 研究员设计的 Tip 计算公式
- 当前区块 BaseFee（来自 Chain-Adapter）

**输出：**
- `pkg/mev/gas_bidder.go`：动态计算 Miner Tip
  - 参考公式：`tip = profit_estimate × 0.10 + base_fee × 1.5`（参数可配置）
  - Flashbots 场景：Bundle 竞价按利润比例分配
- Gas 加速阶梯：进入公共 Mempool 后每个区块未确认则增加 10% Gas Price，上限为初始估算 × 3
- 超过上限后停止加速，触发 P1 告警（`defi_gas_acceleration_capped`）

---

### 子任务 2.3：Phase 3 Recovery Manager 完整版升级

**输入：**
- Phase 3 任务 5.1 产出的 `Recovery Manager`（影子模式版）
- 本任务产出的真实广播能力

**输出：**
- 升级 `pkg/recovery/manager.go`：链上不存在（receipt == null）时，启动真实重广播流程
  1. 检查 Nonce 是否仍有效（未被其他交易占用）
  2. 使用阶梯 Gas 加速策略重新广播
  3. 广播前写审计日志记录重广播原因（`reason: missing_receipt | gas_too_low | nonce_stale`）
- 与 MEV Router 集成：重广播也走路由逻辑（高价值走 Flashbots Bundle，低价值走公共路由）

---

## 任务 3：断路器、时间锁与治理风险哨兵

### 📐 架构定位

| 项目 | 内容 |
|---|---|
| **架构位置** | Zone B（时间锁控制器）+ Zone A（Admin Portal 触发）+ 链上（时间锁合约）|
| **对应架构文档** | `5.3_全局断路器与时间锁.md` 全文；`5.2_治理风险哨兵.md` 全文 |
| **在请求链路中的位置** | 外部治理提案 → **Governance Sentinel** → Adapter Registry 状态变更 → PAUSED 触发（R6）；管理员 → **Admin Timelock** → 24h 等待 → 参数生效 |
| **上游依赖** | Phase 1 Adapter Registry（治理哨兵调用 `SetStatus()`）；Phase 3 PAUSED 状态机（R6 规则联动） |
| **下游影响** | 时间锁保护全局风控参数；治理哨兵保护协议级安全；Kill Switch 保护极端场景资产 |
| **接口 Owner** | 财务风控组（时间锁、Kill Switch）；安全组（Zone C 配合）；区块链底层组（治理哨兵 Adapter Registry 联动） |

### ⛔ 本任务禁止事项

```
⛔ 禁止时间锁作用域扩展至策略状态转换（时间锁仅管全局风控参数修改，策略状态转换不受时间锁约束）
⛔ 禁止 Guardian Veto 操作被时间锁延迟（Veto 本身须即时生效）
⛔ 禁止治理哨兵的 CRITICAL/HIGH 状态变更不立即写 Adapter Registry（实时风险标记不允许延迟）
⛔ 禁止 Kill Switch 预生成的 Revoke 模板未经定期测试（须每月在测试网验证一次模板有效性）
```

---

### 子任务 3.1：部署 24 小时 Admin Timelock

**输入：**
- `5.3_全局断路器与时间锁.md` §2.1 时间锁合约规范
- 时间锁作用域参数列表

**输出：**
- 链上时间锁合约部署（可使用 OpenZeppelin TimelockController），接管以下全局风控参数：
  - 合约白名单（Adapter Registry 生产注册）
  - 全局最大滑点阈值
  - 资金池比例上限
  - 大额审批绝对/相对阈值
  - 水位线容忍度 `watermark_max_lag`
  - MPC 节点配置
- 24h 等待期：修改提交后强制等待，期间 Guardian 多签账号可一票否决（Veto）
- Guardian 配置：独立高隔离多签钱包（硬件钱包），与安全组 Lead 控制

**不产出（明确排除）：**
- 不包含策略状态转换的时间锁（STAGED → LIVE 管理员批准立即生效，不受时间锁约束）

---

### 子任务 3.2：Adapter Registry 与时间锁联动

**输入：**
- Phase 1 任务 3 产出的 Adapter Registry `Register()` 接口
- 子任务 3.1 产出的时间锁合约

**输出：**
- 生产环境 Adapter Registry `Register()` 接口接入时间锁：新协议注册提交 → 时间锁合约记录 → 24h 等待 → Guardian 未 Veto → 自动写入 Registry
- 时间锁等待期间 `IsAllowedAddress()` 对新地址返回 `false`（白名单未生效）
- 已注册协议的**状态变更**（`SetStatus()`，供治理哨兵调用）不受时间锁约束（风险标记须实时生效）
- 测试：等待期内地址被拒绝 → 等待期后自动放行 → Guardian Veto 后注册被中止

---

### 子任务 3.3：部署 Global Kill Switch

**输入：**
- `5.3_全局断路器与时间锁.md` §1.3 Kill Switch 执行序列
- 各协议的 `emergency_exit()` 接口（Adapter 中定义）
- Phase 2 Admin Web Portal（触发入口）

**输出：**
- 预生成 Revoke 交易模板（`scripts/revoke_templates/`）：按协议优先级排序的 `Approve(0)` 交易，存储为已签名 Payload，紧急时直接广播
- `services/kill_switch/main.go`：
  1. 冻结 Go-Execution-Engine（停止接受新 `ExecuteTrade`）
  2. 并行广播 Revoke 模板（批量，按协议优先级）
  3. 触发各协议 `emergency_exit()`（顺序由 Adapter 优先级决定）
  4. 发送 P0 全员告警（PagerDuty 电话 + SMS + Slack）
- 性能目标：触发至所有协议 Approve 额度清零 ≤ 60 秒
- 模板有效性维护：每月在测试网执行一次干跑（`--dry-run`），确认模板未因合约升级失效

---

### 子任务 3.4：开发 Governance Sentinel 治理风险哨兵

**输入：**
- Snapshot.org API、Tally.xyz API、链上 Governor 合约事件
- `5.2_治理风险哨兵.md` 四级风险分类
- **降级预案 D**（Snapshot/Tally 持续异常处理）

**输出：**
- `services/governance_sentinel/main.go`：
  - Snapshot.org API（每 5 分钟轮询，含 retry 3 次 + 指数退避）
  - Tally.xyz API（每 5 分钟轮询）
  - 链上 Governor 合约事件（每区块实时，兜底数据源）
- 四级风险映射：

| 级别 | 触发场景 | Adapter Registry 操作 | PAUSED 触发 |
|---|---|---|---|
| CRITICAL | 协议紧急暂停/合约迁移 | 立即 → SUSPENDED | 是（R6 规则） |
| HIGH | LTV 下调 > 20% / 费率上涨 > 50% | → HIGH_RISK | 暂停新建仓 |
| MEDIUM | LTV 微调 < 20% | 不变 | 否（发通知） |
| LOW | 非核心参数调整 | 不变 | 否（记日志） |

- 自动恢复：提案否决 / 时间锁到期未执行 / Guardian Veto → Registry 状态自动恢复 ACTIVE
- API 健康监控：`defi_governance_api_poll_success{source}` Counter；API 静默 15 分钟触发 `GovernanceAPISilent` P0
- 本地缓存：48 小时历史提案缓存（PostgreSQL），API 失败时以缓存作为治理状态基准

---

## 任务 4：AI 审计闭环与进化飞轮

### 📐 架构定位

| 项目 | 内容 |
|---|---|
| **架构位置** | Zone A AI-Feedback-Service，连接 Zone B 审计数据与大模型 Prompt 优化 |
| **对应架构文档** | `6.4_AI反馈飞轮.md` §2–§4；`4.2_资产估值与审计.md` §5 软滑点与毒性流计算 |
| **在请求链路中的位置** | 交易 CONFIRMED → Zone B 财务组 60s 内投递 FeedbackEvent → Redis Stream → Zone A AI-Feedback-Service → Vector DB → Prompt 优化 |
| **上游依赖** | Phase 2 任务 4（双分录账本，软滑点写 ledger_entries）；Phase 3 任务 4.4（DEPRECATED 退役数据归档，负样本来源）|
| **下游影响** | 驱动策略 AI 迭代，是系统自我学习能力的完整闭合 |
| **接口 Owner** | 财务风控组（Zone B 数据投递）；AI 与量化策略组（Zone A 消费处理） |

### ⛔ 本任务禁止事项

```
⛔ 禁止软滑点计算超过 CONFIRMED + 60 秒才写入 Redis Stream（必须触发 P1 告警）
⛔ 禁止 Vector DB 存储省略 trace_id 字段（所有 embedding 须携带完整 trace_id 和 strategy_id）
⛔ 禁止毒性流检测在 CONFIRMED 后 180 秒内未完成（+10 区块约 120 秒，加 60 秒处理，超时 P1 告警）
⛔ 禁止退役策略负样本写入时省略失败原因字段（须包含 pause_reason 和 deprecate_reason）
```

---

### 子任务 4.1：计算并投递审计结果（60 秒 SLA）

**输入：**
- `trade_tasks` 表 CONFIRMED 事件触发
- `TradeRequest.proposalPrice`（策略发单时预期价格）
- 链上执行收据（实际成交价）
- `4.2_资产估值与审计.md` §5 软滑点计算公式

**输出：**
- `pkg/audit/feedback_producer.go`：交易 CONFIRMED 后触发，在 **60 秒内**完成：
  1. **软滑点计算**：`slippage_bps = (actual_price - proposal_price) / proposal_price × 10000`，写入 `ledger_entries`（`entry_type = SLIPPAGE`）
  2. 异步写入 Redis Stream `stream:internal:accounting`，字段：`trace_id`、`strategy_id`、`slippage_bps`、`market_context`（波动率、流动性深度、Gas 价格、水位线滞后）
- **毒性流检测**（CONFIRMED + 10 区块后执行，允许 CONFIRMED + 180 秒内完成）：
  - 检测确认区块后 +10 区块内价格是否持续向不利方向移动，超过配置阈值则标记 `ToxicFlowDetected`
  - 连续 3 次 `ToxicFlowDetected` 触发 PAUSED R3 规则（联动 Phase 3 任务 4）
- SLA 监控：`defi_feedback_delivery_latency_ms` Histogram；> 60000ms 触发 `FeedbackSLAViolation` P1 告警

---

### 子任务 4.2：打通 Vector DB 长期记忆

**输入：**
- Redis Stream `stream:internal:accounting` 中的 `FeedbackEvent`
- `6.4_AI反馈飞轮.md` §3 向量化规范

**输出：**
- `services/ai_feedback/vector_store.go`：消费 FeedbackEvent，向量化存储：
  - **特征向量**：市场状态（波动率、流动性深度、Gas 价格、水位线滞后、链上活跃地址数）
  - **执行结果**：软滑点 bps、毒性流标记、FSM 最终状态、实际 Gas 消耗
  - **必要元数据**：`trace_id`、`strategy_id`、`block_number`、`chain_id`
- 300 秒内完成向量化写入（`defi_vector_db_write_latency_ms` 监控）

---

### 子任务 4.3：Prompt 动态生成机制

**输入：**
- Vector DB 历史记录
- 当前市场状态特征向量

**输出：**
- `services/ai_feedback/prompt_generator.go`：策略进入迭代周期时：
  1. 以当前市场特征向量检索 Vector DB，召回最相似历史场景（Top-K，K=5）
  2. 拼装优化 Prompt，格式示例：
     ```
     在波动率 > 3%、流动性 < 5M 的场景中（trace_id: xxxxxx），
     上次 CLAMM 策略遭遇 30% 实际滑点（vs 预期 0.5%），
     根因：水位线滞后 4 个区块（sync_block lag = 4），信号已过时。
     建议本次：检查水位线滞后 < 2，或采用 TWAP 拆单，或放弃该区块发单。
     ```
- **DEPRECATED 策略负样本**：接入 Phase 3 任务 4.4 归档的退役策略数据，批量写入 Vector DB（标注 `sample_type: negative`，含 `pause_reason`、`deprecate_reason`），增强飞轮学习多样性

---

## 任务依赖关系

```
任务 1（Zone C 物理签名 + 差异化审批）
  ├─ 前置条件：Phase 0 §C.3 MPC 密钥仪式完成（硬性约束，不可跳过）
  ├─ 阻塞 → 任务 2（MEV Router 需要真实 signed_tx 才能广播）
  └─ 阻塞 → 全部真实广播能力

任务 1 + 任务 2（真实签名 + 广播就绪）
  └─ 阻塞 → Phase 4 STAGED 阶段真实资金拨测

任务 3（断路器 + 时间锁 + 治理哨兵）
  ├─ 子任务 3.1（时间锁合约）阻塞 → 子任务 3.2（Registry 联动）
  ├─ 子任务 3.3（Kill Switch）可与 3.1/3.2 并行
  └─ 子任务 3.4（治理哨兵）可与 3.1/3.2 并行，但依赖 Phase 1 Adapter Registry

任务 4（AI 飞轮）
  ├─ 依赖 → Phase 3 任务 4.4（DEPRECATED 数据归档，负样本来源）
  └─ 可与任务 1、2、3 并行启动（但需要 Phase 3 CONFIRMED 事件触发，须待真实广播就绪）

外部审计（第三方）
  └─ 依赖任务 1、3 交付后启动（Zone C Whitelist Proxy + 时间锁合约 + Kill Switch 是审计核心对象）
  └─ 阻塞 → 系统进入 LIVE 状态（审计报告出具是前置条件）

可并行执行：任务 2（MEV Router）+ 任务 3（时间锁 + 治理哨兵）+ 任务 4（AI 飞轮基础）
关键路径：MPC 密钥仪式 → 任务 1 → 任务 2 → 外部审计 → LIVE
```

---

## 🏁 Phase 4 阶段验收标准（DoD）

| # | 验收标准 | 验收方法 | 验收责任人 |
|---|---|---|---|
| 1 | **外部安全审计通过** | 第三方审计机构出具官方报告，确认：Zone C Whitelist Proxy 明文 Hash 逻辑无绕过漏洞；`IS_KILL_SWITCH` 防伪造机制有效；时间锁合约无提权漏洞；`IsAllowedAddress()` 无注入漏洞 | 安全组 + 第三方审计机构 |
| 2 | **Kill Switch 60 秒演练** | 测试网模拟单节点被黑，触发 Kill Switch，确认：≤ 60 秒所有协议 Approve 额度清零；Admin Portal 实时展示执行进度；HSM 审计日志可查 | 安全组 + 架构与 SRE 组 |
| 3 | **防夹能力实证（G8 基础）** | MEV Router 在 STAGED 灰度期构造 10 笔 Flashbots Bundle，链上浏览器确认交易未经公共 Mempool；三明治机器人无法夹击（通过区块数据分析验证） | 区块链底层组（MEV 研究员） |
| 4 | **时间锁与 Registry 联动** | 提交新协议注册，确认等待期内 `IsAllowedAddress()` 返回 `false`；等待期结束后自动返回 `true`；Guardian 在等待期内执行 Veto，注册被中止 | 安全组 + 区块链底层组 |
| 5 | **AI 飞轮 60 秒 SLA** | 执行一笔真实交易并确认，Grafana 监控 `defi_feedback_delivery_latency_ms`，P99 < 60000ms；查询 `stream:internal:accounting`，确认 60 秒内有对应 `trace_id` 的 `FeedbackEvent`；Vector DB 300 秒内完成向量化写入 | 财务风控组 + AI 策略组 |
| 6 | **差异化审批阈值验证** | STAGED 场景：构造 $9,999 交易直接通过，$10,001 交易进入 PENDING_APPROVAL；LIVE 场景：构造首次调用新合约地址，进入 PENDING_APPROVAL；批准后 FSM 正确推进 | 区块链底层组 Tech Lead + 财务风控组交叉确认 |
| 7 | **Flashbots 降级逻辑验证** | 模拟 Flashbots Relay 不可用（停止 Mock Relay），确认：3 个区块未打包后降级至公共路由；降级日志可在 Grafana 检索；`broadcast_route: flashbots_degraded` 字段正确写入 `trade_tasks` | 区块链底层组（MEV 研究员）+ 架构与 SRE 组 |
| 8 | **治理哨兵实证** | 在测试治理合约发起一个 Mock CRITICAL 提案，确认：哨兵在 1 个区块内检测到；Adapter Registry 状态变更至 SUSPENDED；涉及该协议的策略自动触发 PAUSED（R6）；提案否决后状态恢复 ACTIVE | 财务风控组 + 区块链底层组 |
| 9 | **Snapshot/Tally API 降级验证** | 停止 Mock Snapshot API 响应，确认：连续 3 次失败触发 P1 告警；15 分钟静默触发 P0 告警；链上 Governor 事件仍正常处理；API 恢复后补扫机制执行，补扫结果写入审计日志 | 区块链底层组 + 架构与 SRE 组 |
| 10 | **STAGED → LIVE 全流程验收（G8）** | ≥ 7 天 STAGED 运行，零对账差异，管理员离线多签确认；第一笔由 AI 决策、经硬件 MPC 签名、通过 Flashbots 私有路由广播的真实主网交易成功落地；系统进入 LIVE 状态；写入 `gate_results`（`gate_id='G8', status='PASS'`） | 全体团队 Tech Lead 联合验收 |
