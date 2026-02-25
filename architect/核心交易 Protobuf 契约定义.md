## 核心交易 Protobuf 契约定义 (v1.0.0)

### 1. 基础类型与资产定义 (`types.proto`)

在该部分，我们杜绝使用任何浮点数类型。所有金额字段均以字符串形式传输，代表其在链上的原始整数值（如 Wei）。

```proto
syntax = "proto3";
package defi.v1;

// 资产金额：强制使用 String 传输原始整数位，防止精度丢失
message BigAmount {
  string raw_value = 1;      // 例如: "1000000000000000000" (1 ETH)
  int32 decimals = 2;       // 精度，例如: 18
  string symbol = 3;        // 代币符号，例如: "ETH"
}

// 交易元数据：用于全链路追踪和水位线校验
message Context {
  string trace_id = 1;       // 全局 Trace ID
  string agent_id = 2;       // 发起决策的 AI Agent ID
  uint64 last_sync_block = 3; // 决策基于的数据水位线区块号
  int64 timestamp_ms = 4;    // 决策生成的时间戳
}

```

### 2. 核心交易服务 (`trading_service.proto`)

定义了 Python Agent 如何向 Go 执行层提交决策，以及 Go 如何反馈流式状态。

```proto
service TradingEngine {
  // 提交交易决策：执行层将进行仿真验证、风控检查及签名
  rpc ExecuteTrade(TradeRequest) returns (TradeResponse);

  // 影子模式流：仅记录决策，不执行签名
  rpc ShadowTrade(TradeRequest) returns (TradeResponse);
  
  // 心跳与同步：确保 Python 层的同步水位线未过期
  rpc GetSystemStatus(Empty) returns (SystemStatus);
}

// 详细的交易载荷
message TradeRequest {
  Context ctx = 1;
  string protocol_id = 2;    // 目标协议，如 "UNISWAP_V3"
  TradeAction action = 3;    // 动作类型：SWAP, ADD_LIQUIDITY, REMOVE_LIQUIDITY
  bytes payload = 4;         // 具体的协议参数（序列化后的各协议特定参数）
  
  // 核心风控参数
  string max_slippage_bps = 5; // 最大滑点（以万分之一为单位）
  string gas_price_limit = 6;  // Gas 价格上限 (Gwei)
}

enum TradeAction {
  ACTION_UNSPECIFIED = 0;
  SWAP = 1;
  ADD_LIQUIDITY = 2;
  REMOVE_LIQUIDITY = 3;
}

message TradeResponse {
  string internal_tx_id = 1; // 系统内部交易 ID
  TradeStatus status = 2;
  string tx_hash = 3;        // 链上 Hash（若已广播）
  string error_message = 4;  // 失败原因：如 "StaleDataError"
}

enum TradeStatus {
  INIT = 0;
  SIMULATED = 1;             // 已通过本地仿真校验
  PENDING_APPROVAL = 2;      // 触发大额阈值，等待人工审批
  SIGNED = 3;
  BROADCASTED = 4;
  CONFIRMED = 5;
  REVERTED = 6;
}

```

---

## 3. 设计关键点深度解析

### 3.1 精度保护协议

* **字段选型**：在 `BigAmount` 中，`raw_value` 使用 `string` 类型。
* **处理逻辑**：
* **Python 端**：使用 `decimal.Decimal` 或 `int` 进行逻辑计算后，转为 `str` 发送。
* **Go 端**：接收后直接转换为 `math/big.Int` 进入 **模块 1.1** 的 EVM 仿真逻辑，无缝对接 `go-ethereum` 原生库。



### 3.2 水位线强约束 (G7 门控)

* `last_sync_block` 字段是执行层的“硬门控”。
* 当 Go 端的 **模块 4.1 Sync Sentinel** 收到 `TradeRequest` 时，会对比当前的 `CurrentBlock`。
* 如果 `CurrentBlock - last_sync_block > 2`，Go 会直接返回 `StaleDataError`，拒绝将过时的 AI 决策送入仿真或签名流程。

### 3.3 协议解耦与扩展

* `payload` 使用 `bytes` 字段，允许针对不同协议（Uniswap, Curve, Aave）定义各自的参数结构。
* 这配合了 **模块 1.2 Adapter Registry** 的设计，使得新增协议支持时，只需在 Proto 中增加对应的 `message` 定义，而无需修改核心 `TradingEngine` 接口。
