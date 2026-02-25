## 🏛️ DeFi 资管系统 Protobuf 契约汇编 (v1.0.0)

### 1. 基础类型定义 (`common.proto`)

该文件定义了跨模块共享的原子数据结构，核心在于通过字符串处理大数以杜绝精度风险。

```proto
syntax = "proto3";
package defi.v1;

option go_package = "github.com/holobit/proto/v1;defiv1";

// 财务级大数：强制使用 String 传输原始整数位 (Wei)
message BigAmount {
  string raw_value = 1;      // 例如: "1000000000000000000" (1 ETH)
  int32 decimals = 2;       // 精度位，如 18
  string symbol = 3;        // 代币符号，如 "ETH"
}

// 治理与追踪上下文
message Context {
  string trace_id = 1;       // 全链路追踪 ID
  string agent_id = 2;       // 发起决策的 AI Agent ID
  uint64 last_sync_block = 3; // 决策时参考的数据水位线
  int64 timestamp_ms = 4;    // 信号产生的时间戳 (毫秒)
}

// 交易状态枚举
enum TradeStatus {
  TRADE_STATUS_UNSPECIFIED = 0;
  INIT = 1;                 // 任务创建
  SIMULATED = 2;            // 通过 EVM 仿真与不变性断言
  PENDING_APPROVAL = 3;     // 触发大额阈值，等待人工审批
  SIGNED = 4;               // 完成 MPC 签名
  BROADCASTED = 5;          // 已推送到 mempool/Flashbots
  CONFIRMED = 6;            // 链上确认并完成对账
  REVERTED = 7;             // 链上执行失败
  FAILED = 8;               // 系统内部拦截或熔断
}

```

---

### 2. 核心交易服务 (`trading.proto`)

定义了策略大脑（Python）与执行肢体（Go）之间的交互逻辑，支持双轨延迟路由。

```proto
import "common.proto";

service TradingEngine {
  // 实盘交易执行：包含完整 FSM 流转与 MPC 签名
  rpc ExecuteTrade(TradeRequest) returns (TradeResponse);

  // 影子模式执行：跑完所有验证逻辑但不进行真实签名
  rpc ShadowTrade(TradeRequest) returns (TradeResponse);
}

message TradeRequest {
  Context ctx = 1;           // 包含水位线与追踪 ID
  string protocol_id = 2;    // 目标协议，如 "UNISWAP_V3"
  TradeAction action = 3;    // 动作类型
  bytes payload = 4;         // 协议特定的编码参数 (Adapter 专用)
  
  // 执行参数限制
  string max_slippage_bps = 5; // 最大滑点 (万分位)
  string gas_price_limit = 6;  // Gas 价格上限
}

enum TradeAction {
  ACTION_UNSPECIFIED = 0;
  SWAP = 1;
  ADD_LIQUIDITY = 2;
  REMOVE_LIQUIDITY = 3;
}

message TradeResponse {
  string internal_tx_id = 1; // 数据库任务主键
  TradeStatus status = 2;    // 当前状态
  string tx_hash = 3;        // 链上交易哈希
  string error_message = 4;  // 详细错误信息 (如 "StaleDataError")
}

```

---

### 3. 数据流推送服务 (`data_stream.proto`)

该文件支持 50+ Agent 的高频数据分发，实现事件驱动架构。

```proto
import "common.proto";

service DataSentinel {
  // 实时行情与水位线推送流
  rpc SubscribeEvents(SubscribeRequest) returns (stream ChainEvent);
}

message SubscribeRequest {
  string agent_id = 1;
  repeated string protocol_ids = 2; // 订阅的协议列表
}

message ChainEvent {
  uint64 block_number = 1;    // 当前水位线
  string block_hash = 2;
  repeated bytes logs = 3;    // 原始 Logs 数据
  int64 server_time_ms = 4;   // Go 端接收数据的时间戳
}

```

---

## 🛠️ 契约设计的工业级特性说明

1. **精度保护协议**：所有的金额字段在 Proto 中均定义为 `string`，在 Python 侧映射为 `Decimal`，在 Go 侧映射为 `big.Int`。这完美解决了 PRD 1.1 中提到的浮点数陷阱。
2. **物理水位线拦截**：`last_sync_block` 字段被强制要求包含在每一个请求的 `Context` 中。Go 端执行器在仿真前会检查该高度，若延迟超过 2 个区块则根据 **G7 门控** 直接物理打回决策。
3. **高度解耦的 Payload**：`TradeRequest` 中的 `payload` 采用 `bytes` 类型。这意味着当新增协议（如 Curve）时，只需在适配器模块定义新的 Proto 消息，而无需修改核心 `TradingEngine` 的接口定义。
4. **全链路审计**：`trace_id` 被定义为必填项。从 AI 产生信号到数据库存证，再到 MPC 签名日志，该 ID 贯穿始终，支撑模块 6.1 的全局监控。
