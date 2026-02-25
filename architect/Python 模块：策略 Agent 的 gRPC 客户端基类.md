## Python 模块：策略 Agent 客户端基类设计

### 1. 核心基类实现 (`base_agent.py`)

我们将使用 Python 的 `decimal` 模块来处理高精度计算，并将其无缝转化为 Protobuf 契约中的 `string` 格式。

```python
import grpc
from decimal import Decimal
import uuid
import time
from typing import Optional
from defi.v1 import trading_service_pb2, trading_service_pb2_grpc

class BaseStrategyAgent:
    """
    策略 Agent 基类：处理 gRPC 通信、精度转换及全链路追踪
    """
    def __init__(self, agent_id: str, grpc_endpoint: str):
        self.agent_id = agent_id
        self.channel = grpc.insecure_channel(grpc_endpoint)
        self.stub = trading_service_pb2_grpc.TradingEngineStub(self.channel)
        # 本地维护的最新的水位线
        self.last_seen_block = 0

    def _to_big_amount(self, value: Decimal, decimals: int, symbol: str) -> trading_service_pb2.BigAmount:
        """
        将 Python Decimal 转换为强类型的 Protobuf String，杜绝精度丢失
        """
        raw_int_value = int(value * (10 ** decimals))
        return trading_service_pb2.BigAmount(
            raw_value=str(raw_int_value),
            decimals=decimals,
            symbol=symbol
        )

    def create_context(self) -> trading_service_pb2.Context:
        """
        构造包含 Trace ID 和水位线的上下文
        """
        return trading_service_pb2.Context(
            trace_id=str(uuid.uuid4()),
            agent_id=self.agent_id,
            last_sync_block=self.last_seen_block,
            timestamp_ms=int(time.time() * 1000)
        )

    def execute_swap(self, 
                     protocol_id: str, 
                     token_in: str, 
                     amount_in: Decimal, 
                     token_out: str,
                     slippage_bps: int,
                     is_shadow: bool = False):
        """
        执行 Swap 决策：支持实盘与影子模式
        """
        # 组装 Payload (此处仅为示例，实际应根据协议适配器序列化)
        request = trading_service_pb2.TradeRequest(
            ctx=self.create_context(),
            protocol_id=protocol_id,
            action=trading_service_pb2.SWAP,
            max_slippage_bps=str(slippage_bps),
            # 业务逻辑交由 Go 的 Adapter 处理
        )

        try:
            # 根据模式选择调用接口
            if is_shadow:
                response = self.stub.ShadowTrade(request)
            else:
                response = self.stub.ExecuteTrade(request)
            return response
        except grpc.RpcError as e:
            # 捕获 StaleDataError 或仿真失败异常
            print(f"Trade Execution Failed: {e.details()}")
            return None

    def on_data_received(self, data):
        """
        接收来自 Go 端 L2 推送的实时数据，并更新水位线
        """
        self.last_seen_block = data.block_number
        self.evaluate_strategy(data)

    def evaluate_strategy(self, data):
        """
        由具体的策略子类实现推理逻辑
        """
        raise NotImplementedError

```

---

### 2. 设计关键点深度解析

#### 2.1 精度与数学计算的分工 (The Division of Labor)

* **Python 侧**：仅负责**逻辑决策**。例如：“如果价格跌破 X，则卖出”。所有的计算使用 `Decimal` 类型，在发送给 Go 之前将其转化为 `string`。
* **Go 侧**：负责**数学验证**。Go 收到 `string` 后转化为 `big.Int`，带入 **G2 仿真引擎**。这遵循了“不要让 Python 做复杂 EVM 状态运算”的指导原则。

#### 2.2 影子模式的无缝切换 (Shadow Mode Transition)

* 通过 `is_shadow` 参数，策略可以在不改动任何 AI 模型逻辑的情况下，从“只读演练”切换到“实盘操作”。
* 在影子模式下，Go 端会跑完所有的 **G1-G7 门控**，但会在 **G8 签名关卡** 之前物理截断，从而验证 AI 决策在真实市场水位线下的表现。

#### 2.3 水位线感知的决策 (Watermark-Aware Decisions)

* 基类通过 `last_seen_block` 强制记录当前 AI 看到的“世界状态”。
* 如果 Python 端处理推理耗时过长，导致发送请求时该区块号已落后，Go 端的执行引擎会基于 **G7 门控** 自动拦截该过时决策，防止因“数据过期”导致的滑点超限。

---

### 3. 该模块的评价标准

| 维度 | **好 Agent 基类 (Industrial)** | **坏 Agent 基类 (Junior)** |
| --- | --- | --- |
| **精度处理** | 使用 `Decimal` 并在接口层锁定 `String`。 | 在 Python 内部使用 `float`，导致发送给链上的金额有微小偏差。 |
| **异常感知** | 能够清晰区分 `StaleDataError`（水位线落后）与 `SimulationError`（策略逻辑错误）。 | 只要交易不成功就只返回一个通用的 `Fail` 状态。 |
| **追踪能力** | 每一笔决策都自动携带 UUID 性质的 `Trace_ID`。 | 无法追溯某笔链上交易是由哪一个版本的 AI 模型生成的。 |
