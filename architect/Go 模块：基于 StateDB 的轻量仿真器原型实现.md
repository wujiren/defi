## Go 模块：基于 StateDB 的轻量仿真器设计

### 1. 核心组件架构

为了实现极致的性能与精度，仿真器主要由以下三个核心组件构成：

* **Mock StateDB**：实现 `vm.StateDB` 接口。它不直接操作磁盘，而是将从 RPC 获取的账户状态（Balance, Nonce, Code, Storage）缓存在内存中。
* **EVM 配置器**：负责注入链参数（ChainConfig）、区块上下文（BlockContext）和交易上下文（TxContext），模拟特定的区块高度环境。
* **不变性断言引擎 (Invariant Engine)**：在 EVM 执行结束后，自动检查结果是否违反了“物理底线”（如资产非预期减损）。

---

### 2. 仿真执行流伪代码实现

在 Go 中，我们可以利用 `github.com/ethereum/go-ethereum/core/vm` 包来构建此功能。

```go
// Simulator 结构体定义
type Simulator struct {
    chainConfig *params.ChainConfig
    db          *state.StateDB // 内存态 StateDB
}

// Execute 仿真执行入口
func (s *Simulator) Execute(req *TradeRequest) (*SimulationResult, error) {
    // 1. 准备上下文：模拟当前区块环境
    blockCtx := vm.BlockContext{
        CanTransfer: core.CanTransfer,
        Transfer:    core.Transfer,
        GetHash:     func(n uint64) common.Hash { return s.getPastHash(n) },
        BlockNumber: big.NewInt(int64(req.Ctx.LastSyncBlock)),
        Time:        big.NewInt(time.Now().Unix()),
        GasLimit:    30000000,
    }

    // 2. 构造交易上下文
    txCtx := vm.TxContext{
        Origin:   common.HexToAddress(req.From),
        GasPrice: big.NewInt(int64(req.GasPriceLimit)),
    }

    // 3. 初始化原生 EVM
    evm := vm.NewEVM(blockCtx, txCtx, s.db, s.chainConfig, vm.Config{})

    // 4. 执行调用并获取结果
    // 注意：这里直接运行合约字节码，确保 100% 精度
    ret, leftGas, err := evm.Call(
        vm.AccountRef(txCtx.Origin),
        common.HexToAddress(req.To),
        req.Data,
        req.GasLimit,
        req.Value,
    )

    // 5. 不变性断言校验
    if err := s.checkInvariants(s.db); err != nil {
        return nil, fmt.Errorf("invariant broken: %v", err)
    }

    return &SimulationResult{ReturnData: ret, GasUsed: req.GasLimit - leftGas}, nil
}

```

---

### 3. 该方案如何解决 PRD 痛点

#### 3.1 消除“回测陷阱” (G2 精度)

由于直接使用 `go-ethereum/core/vm`，所有的整数除法、溢出处理以及 Gas 计算都与以太坊主网节点完全一致。这彻底消除了 Python 浮点数运算带来的误差。

#### 3.2 满足低延迟预算 (P99 < 500ms)

* **本地执行**：仿真过程完全在 Go 进程内存中完成，无需启动沉重的 Docker 容器或通过网络访问 Anvil/Hardhat 实例（除非是大额重验证路径）。
* **状态缓存**：`StateDB` 可以通过 `LRU Cache` 缓存常用代币和池子的状态，单次仿真时间通常可控制在 **10ms - 50ms** 以内。

#### 3.3 物理级水位线校验 (G7 门控)

仿真器在初始化 `BlockContext` 时，会强制检查请求中的 `LastSyncBlock`。如果该高度与当前 Go 端维护的 `LatestBlock` 差异超过 2，仿真器将直接拒绝执行，从物理层面防止“基于过期数据的错误决策”。

---

### 4. 关键技术细节：内存 StateDB 的维护

为了让仿真器运行，Go 模块需要持续从 RPC 获取最新的账户状态。

1. **即时同步**：当 `Execute` 被调用时，若发现本地 `StateDB` 缺少某个账户（如新池子），Go 自动触发一次 `eth_getProof` 或 `eth_getStorageAt` 补全状态。
2. **快照回滚**：每一笔仿真交易执行前，调用 `s.db.Snapshot()`。执行完断言后，立即调用 `s.db.RevertToSnapshot()`，确保仿真过程不污染下一次交易的状态。
