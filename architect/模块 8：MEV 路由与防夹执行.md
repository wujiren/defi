## 🏛️ 模块 8：MEV 路由与防夹执行 (MEV Routing & Anti-Sandwich)

### 8.1 路由架构：公共池与隐私池的智能切换

为了防止公共 Mempool 中的三明治攻击（Sandwich Attacks），系统在 **模块 3 (FSM)** 的 `BROADCASTED` 阶段引入 **MEV Router**。

*   **默认路径 (Private Path):** 优先通过 Flashbots Relay (mev-geth) 发送 `Flashbots Bundle`。此路径下的交易不会广播到 P2P 网络，矿工要么打包，要么丢弃，绝无被夹可能。
*   **降级路径 (Public Path):** 若 Flashbots 服务宕机或 Bundle 连续 3 个区块未被打包，自动降级为 `eth_sendRawTransaction` 广播至公共 Mempool，但需同时调高滑点容忍度。

### 8.2 Go 技术实现：MEV 广播器接口

```go
// mev_router.go

type ITxBroadcaster interface {
    Broadcast(ctx context.Context, signedTx *types.Transaction) (string, error)
}

// Flashbots 实现
type FlashbotsBroadcaster struct {
    relayURL string
    signer   *types.PrivateKey // 用于签署 Bundle 的身份 Key (非交易私钥)
}

func (f *FlashbotsBroadcaster) Broadcast(ctx context.Context, signedTx *types.Transaction) (string, error) {
    // 1. 构建 Bundle：包含目标交易 + (可选) 贿赂交易
    bundle := &mev.Bundle{
        Txs:      []*types.Transaction{signedTx},
        BlockNum: currentBlock + 1, // 目标下个区块
    }
    
    // 2. 模拟 Bundle (关键步骤：预先检查是否会 Revert)
    if err := f.Simulate(bundle); err != nil {
        return "", fmt.Errorf("MEV simulation failed: %v", err)
    }

    // 3. 发送至 Relay
    resp, err := f.client.SendBundle(bundle)
    return resp.BundleHash, err
}

// 路由决策逻辑
func (router *MEVRouter) RouteAndSend(tx *types.Transaction, valueUSD *big.Float) {
    // 阈值判断：小额交易直接走公网以节省成本，大额走 Flashbots
    threshold := big.NewFloat(1000.0) // $1000

    if valueUSD.Cmp(threshold) > 0 {
        // 高价值 -> 走隐私路由
        router.flashbots.Broadcast(context.TODO(), tx)
    } else {
        // 低价值 -> 走公共 Mempool
        router.public.Broadcast(context.TODO(), tx)
    }
}
```

### 8.3 评价标准 (Evaluation Standards)

| 维度 | **好架构 (MEV Protected)** | **坏架构 (Sandwich Victim)** |
| :--- | :--- | :--- |
| **执行隐私** | 交易 Hash 在上链前对公网不可见，Attackers 无法侦测。 | 交易一发出，Mempool 爬虫立刻检测到并插入 Front-run 交易。 |
| **失败处理** | Bundle 若未被打包则静默失效，不消耗 Gas。 | 交易在公网 Revert，不仅亏损 Gas 费，还暴露了策略意图。 |
| **回测真实性** | 回测引擎能区分“实盘数据”和“历史文件”，代码零修改。 | 回测需要重写一套逻辑读取 CSV，导致逻辑与实盘不一致。 |

