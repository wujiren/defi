# 🏛️ 模块 3：确定性执行引擎 (Deterministic Execution Engine)

## 3.1 核心设计哲学 (The Philosophy)

状态机不仅是记录状态，更是一套**原子化操作流程**。为了在充满对抗的链上环境中承载千万美金级资产，每一笔交易必须绑定一个唯一的 `Internal_Tx_ID`，并遵循 **“先写日志，后发交易”** 的原则，确保在任何进程崩溃或网络异常的情况下，系统行为是确定且可预测的。

---

## 3.2 状态流转与逻辑细分 (State Transitions)

每一笔交易必须严格遵循以下状态流转，且状态变更必须伴随数据库事务的成功提交。

| 状态 (State) | 执行动作与持久化要求 | 自愈/重试逻辑 (Self-healing) |
| --- | --- | --- |
| **INIT** | 接收来自 Python Agent 的 gRPC 提议，在本地 DB 创建记录，分配 `Internal_Tx_ID`。 | 崩溃重启后：若 DB 有记录但无后续状态，重新进入仿真流程。 |
| **SIMULATED** | 在本地 Go EVM 仿真器中执行，通过所有不变性断言。 | 崩溃重启后：重新模拟以校验当前链上水位线是否已发生不利变化。 |
| **SIGNED** | **核心点**：从 Nonce 锁管理器获取 Nonce，完成 MPC 签名。**严禁重签**。 | 若已签名且 Nonce 已分配，重启后必须提取原签名 Payload，不得递增 Nonce。 |
| **BROADCASTED** | 将 Signed Payload 推送至 RPC 或 Flashbots，保存 `Tx_Hash`。 | 重启后：通过 Hash 查询。若超时则触发阶梯 Gas 加速（替换原 Nonce 交易）。 |
| **CONFIRMED** | 监控到 Block Inclusion 且达到配置的安全确认数。 | 更新账本（模块 4.1），释放锁定资产。 |
| **REORG** | 检测到链重组，当前 Hash 失效。 | 回滚至 **SIMULATED** 状态，重新评估滑点并决定是否重新发单。 |

---

## 3.3 数据库物理实现 (PostgreSQL Schema)

为了支撑上述逻辑，我们采用 PostgreSQL 实现强一致性的存储与锁机制。

### 3.3.1 核心交易任务表 (`trade_tasks`)

```sql
CREATE TYPE trade_status AS ENUM (
    'INIT', 'SIMULATED', 'SIGNED', 'BROADCASTED', 'CONFIRMED', 'REVERTED', 'FAILED'
);

CREATE TABLE trade_tasks (
    internal_tx_id UUID PRIMARY KEY,         -- 唯一 ID
    trace_id UUID NOT NULL,                  -- 全链路追踪 ID
    agent_id VARCHAR(64) NOT NULL,           -- 发起决策的 Agent ID
    
    chain_id INTEGER NOT NULL,
    from_address CHAR(42) NOT NULL,
    nonce BIGINT,                            -- 物理 Nonce 锁绑定
    
    status trade_status NOT NULL DEFAULT 'INIT',
    raw_payload JSONB NOT NULL,              -- 原始交易数据
    tx_hash CHAR(66) UNIQUE,                 -- 广播后的 Hash
    
    sync_block BIGINT NOT NULL,              -- 决策时的水位线
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT unique_nonce_per_account UNIQUE (from_address, chain_id, nonce)
);

```

### 3.3.2 分布式 Nonce 锁 (`nonce_locks`)

采用数据库行级锁，确保高并发下同一个地址的 Nonce 绝不冲突。

```sql
CREATE TABLE nonce_locks (
    address CHAR(42) NOT NULL,
    chain_id INTEGER NOT NULL,
    next_nonce BIGINT NOT NULL DEFAULT 0,
    locked_at TIMESTAMP WITH TIME ZONE,
    PRIMARY KEY (address, chain_id)
);

```

---

## 3.4 关键组件设计 (Key Components)

### 3.4.1 原子化双写与 Nonce 分配

Go 执行层在 `SIGNED` 阶段必须执行一个原子事务：

1. **获取锁**：`SELECT next_nonce FROM nonce_locks WHERE address = ? FOR UPDATE`。
2. **状态更新**：更新 `trade_tasks` 为 `SIGNED` 并填入该 Nonce。
3. **递增锁**：将 `next_nonce` 加 1。
4. **提交**：此过程确保了即使 50 个 Agent 同时请求，每个请求也会拿到唯一且连续的 Nonce。

### 3.4.2 跨链库存再平衡 (Active Rebalancing)

针对多链资金调度，采用“集散式”控制流：

* **监控层**：计算 $Util\_Rate = (Total - Idle) / Total$。
* **执行层**：调用状态机执行 `Withdraw -> Bridge -> Deposit` 原子链条。每一跳（Hop）都是一个独立的 FSM 任务，前一跳 `CONFIRMED` 是后一跳开始的触发器。
### 3.4.3 恢复管理器 (Recovery Manager) 设计

恢复管理器作为 Go 执行层的后台服务，在系统启动时以及运行期间（定时触发）执行扫描任务。

#### 1. 系统启动时的状态对齐 (Boot Recovery)

当系统重启时，FSM 扫描器首先处理所有未完成的任务：

```go
// StartRecoveryManager 启动自愈扫描器
func (m *RecoveryManager) StartRecovery(ctx context.Context) {
    // 1. 处理已签名但未广播的任务 (SIGNED 状态且 tx_hash 为空)
    m.handleSignedTasks(ctx)

    // 2. 处理已广播但未确认的任务 (BROADCASTED 状态)
    m.handleBroadcastedTasks(ctx)
}

```

#### 2. 核心状态补偿逻辑

| 异常场景 | 数据库检测条件 | 自愈动作 (Deterministic Resolution) |
| --- | --- | --- |
| **签名后崩溃** | `status = 'SIGNED' AND tx_hash IS NULL` | **重新广播**：直接从 DB 提取已签名的 Payload 再次推送，严禁重新分配 Nonce。 |
| **发送后断网** | `status = 'BROADCASTED' AND tx_hash IS NOT NULL` | **链上轮询**：使用 `tx_hash` 查询收据。若 Hash 在链上找不到，则降级为 `SIGNED` 重新发送。 |
| **Gas 过低卡单** | `status = 'BROADCASTED' AND updated_at < NOW() - 5m` | **阶梯加速**：使用同一 Nonce，提升 Gas Price 后重新签名广播（替换交易）。 |
| **链重组 (Reorg)** | `status = 'CONFIRMED' AND 链高度回滚` | **状态回滚**：检测到原区块哈希变化，将状态回滚至 `SIMULATED` 重新评估。 |

---

### 3.4.4 阶梯 Gas 加速实现 (Gas Laddering)

针对 `BROADCASTED` 状态下长时间未确认的任务，自愈逻辑会自动触发加速流程：

```go
func (m *RecoveryManager) handleStuckTask(task *TradeTask) {
    // 1. 检查是否达到加速时间阈值
    if time.Since(task.UpdatedAt) < 5 * time.Minute {
        return
    }

    // 2. 获取当前市场 Gas 价格并上浮 15%
    newGasPrice := m.gasOracle.SuggestPrice().Mul(115).Div(100)

    // 3. 使用相同 Nonce 重新生成 Payload
    // 关键：保持 internal_tx_id 不变，实现幂等覆盖
    newPayload := m.adapter.UpdateGas(task.RawPayload, newGasPrice)
    
    // 4. 更新数据库并标记为重新广播
    m.db.UpdateTaskGas(task.InternalTxID, newGasPrice, "SIGNED")
}

```

---

## 3.5 评价标准 (Evaluation Standards) - 自愈专项

| 维度 | **好自愈 (Resilient)** | **坏自愈 (Fragile)** |
| --- | --- | --- |
| **幂等性** | 无论重启多少次，同一 `internal_tx_id` 绝不产生两笔不同 Nonce 的交易。 | 重启后由于丢失 Nonce 锁记录，导致产生 Nonce 冲突交易或空洞。 |
| **资产安全** | 在加速或替换交易时，严格校验新旧交易的 `to_address` 必须一致。 | 只要能发出交易即可，不校验交易内容的原子性。 |
| **自愈耗时** | 进程崩溃后，在 15 秒内完成状态扫描并恢复广播。 | 依赖人工手动处理卡单，处理延迟以分钟计。 |
| **账本同步** | 自愈完成（CONFIRMED）后，立即触发 **模块 4.1** 的双分录对账。 | 交易成功了但数据库账本未同步，导致资产净值虚假。 |
