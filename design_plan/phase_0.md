# Phase 0 详细分解：全局契约定义与基建锁定（修订版 v1.1）

**⏱️ 预计耗时**：1-2 周（强依赖，需集中攻坚）
**🎯 核心目标**：构建"物理管道"，固化跨语言通信的"数据结构"，产出各团队可以直接调用的 Mock/Stub（存根）代码。**同步部署可观测性基础设施，确保 Phase 1 起各阶段的开发质量有数据支撑。**

> **v1.1 修订说明：**
> - 新增任务 4：可观测性基础设施部署（Prometheus / Alertmanager / Grafana / Loki）
> - 原任务 4（Mocking）调整为任务 5，内容不变

---

## 任务 1：API 契约固化与分发 (API-First Contracts)

**主导团队**：架构与 SRE 组
**协办团队**：AI 策略组（Zone A）的 Python 骨干、执行组（Zone B）的 Go 骨干

这是整个并行开发的第一块基石。必须明确 Python 到 Go 的数据结构。

* **子任务 1.1：编写 `common.proto`**
  * **工作内容**：定义全局共享的枚举和基础类型。
  * **硬性约束**：强制定义 `message BigAmount { string value = 1; }`。必须在注释中写明：所有金额传输必须为 `string` 类型的 Wei 原始单位，严禁使用 `float/double`。同时定义 `TradeStatus` 枚举，零值使用 `TRADE_STATUS_UNSPECIFIED = 0` 占位，业务状态从 `1` 起。
  * **交付物**：`common.proto` 文件。

* **子任务 1.2：编写 `trading.proto`**
  * **工作内容**：定义 Zone A 到 Zone B 的核心交易 RPC 接口，含 `ExecuteTrade`、`ShadowTrade`、`GetSystemStatus` 三个方法。
  * **关键字段**：`TradeRequest` 必须包含 `trace_id`、`agent_id`、`last_sync_block`（数据水位线）、`strategy_id`，以及使用 `bytes payload` 承载不同协议的适配器参数。
  * **交付物**：`trading.proto` 文件。

* **子任务 1.3：编写 `data_stream.proto`**
  * **工作内容**：定义 Zone B 向 Zone A 推送链上事件的流式 RPC 接口。
  * **交付物**：`data_stream.proto` 文件，包含 `SubscribeEvents` 和 `ChainEvent` 消息定义（含 `block_number` 水位线字段）。

* **子任务 1.4：编译与分发环境搭建**
  * **工作内容**：使用 `protoc` 生成 Python 的 `_pb2.py` 和 Go 的 `.pb.go` 桩代码。
  * **交付物**：搭建内部私有包仓库（或 Git Submodule），让两端工程师可以直接 `import` 这些结构体。

---

## 任务 2：金融级数据库 Schema 锁定

**主导团队**：财务风控组与 DBA 组

在写业务代码前，必须把核心表结构定死，任何对这些表的修改在 Phase 0 结束后必须走严格的 SQL Migration 流程。

* **子任务 2.1：初始化交易核心表**
  * **工作内容**：编写 `trade_tasks`（FSM 生命周期）和 `nonce_locks`（Nonce 分配）的 DDL 语句。
  * **硬性约束**：时间字段统一使用 `TIMESTAMP WITH TIME ZONE`；`status` 字段必须与 `TradeStatus` 枚举完全对应。

* **子任务 2.2：初始化双分录会计核心表**
  * **工作内容**：编写 `accounts`、`ledger_entries`、`asset_balances`、`unclaimed_rewards` 的 DDL。
  * **硬性约束**：金额字段强制定义为 `NUMERIC(78, 0)` 以支持 uint256 精度，防止溢出。`entry_type` 字段须覆盖：`TRADE`、`GAS_FEE`、`PROTOCOL_FEE`、`REWARD`、`SLIPPAGE`、`MEV_LOSS`。

* **子任务 2.3：部署与权限分配**
  * **工作内容**：在开发环境拉起 PostgreSQL 实例；为 Go-Execution-Engine 分配读写账号，设置初始连接池参数（`max_connections = 200`）。

---

## 任务 3：隔离网络与中间件拉起

**主导团队**：架构与 SRE 组、数据工程组

搭建系统运行的骨架，确立三区隔离的物理边界。

* **子任务 3.1：配置三区 Docker/VPC 网络**
  * **工作内容**：在开发环境通过 `docker-compose` 或 Kubernetes 配置 `intelligence_net`（Zone A）、`sandbox_net`（Zone B）和 `fortress_net`（Zone C）。
  * **硬性约束**：`fortress_net` 设置 `internal: true`，物理阻断任何出站公网连接；`intelligence_net` 内的容器无法直接路由到 `fortress_net`。

* **子任务 3.2：建立 mTLS 证书体系**
  * **工作内容**：生成内部 Root CA，签发 Zone B 的客户端证书和 Zone C 的服务端证书，有效期 90 天。
  * **交付物**：将证书文件作为 Docker Secret 挂载到对应的开发容器中。

* **子任务 3.3：拉起 Redis 消息总线**
  * **工作内容**：在 Zone B 拉起 Redis Cluster，配置 AOF 持久化模式。
  * **交付物**：通过初始化脚本显式创建 `stream:chain:events`（链上事件总线）和 `stream:internal:accounting`（财务审计事件），并设置初步的保留策略（`MAXLEN ~10000`）。

---

## 任务 4：可观测性基础设施部署 *(新增)*

**主导团队**：架构与 SRE 组
**协办团队**：各组 Tech Lead（协助定义各自模块的指标埋点规范）

**为什么要在 Phase 0 部署可观测性？** 如果等到 Phase 3/4 才建，前两个阶段的开发和验收就缺乏有效的度量手段——Nonce 锁的并发安全、仿真器的 P99 延迟、回测数据的完整性，这些都需要有指标才能量化验收。

* **子任务 4.1：部署 Prometheus + Alertmanager**
  * **工作内容**：在 Zone B 部署 Prometheus 实例，配置各服务的 `/metrics` 抓取端点（Go-Execution-Engine `:9090`、Python-Agent `:9091`、MEV-Router `:9092`）；部署 Alertmanager，配置告警路由（CRITICAL 走 PagerDuty + Slack，WARNING 走 Slack）。
  * **初始告警框架**：录入 `6_3_告警规则集.md` 中的 16 条告警规则（此时部分指标还未埋点，规则暂处于 `inactive` 状态，随各模块交付逐步激活）。
  * **交付物**：`prometheus.yml` + `alertmanager.yml` 配置文件，各团队可按此格式添加自己的指标。

* **子任务 4.2：部署 Grafana + Loki 日志聚合**
  * **工作内容**：部署 Grafana，接入 Prometheus 和 Loki 两个数据源；部署 Grafana Loki，配置各容器的日志收集（JSON 结构化格式，强制要求 `trace_id` 字段）。
  * **初始仪表盘**：建立 4 个基础面板（系统健康 / 交易执行 / 财务 / 安全），各模块随交付逐步填充数据。
  * **交付物**：可访问的 Grafana 地址，各团队开发时可直接查看自己模块的日志和指标。

* **子任务 4.3：Prometheus 指标埋点规范分发**
  * **工作内容**：将 `6_2_Prometheus指标定义.md` 中的 10 类、14 个核心指标定义（命名空间 `defi_`、Label 规范、Histogram Bucket 配置）整理为开发规范文档，分发给各组 Tech Lead。各组须在自己的模块中按规范埋点，不允许自行发明私有指标命名。

---

## 任务 5：制造假通道（Mocking）以解除阻塞

**主导团队**：各组 Tech Lead 协作

这是并行开发的关键。为了让后续阶段大家能立刻开始写逻辑，必须提供"假接口"。

* **子任务 5.1：Zone B 的假 gRPC Server**
  * **工作内容**：Go 团队启动一个监听端口的 gRPC 服务，收到 Python 发来的 `TradeRequest` 后，不经过任何仿真，直接打印日志，并返回状态为 `SIMULATED` 的 `TradeResponse`。

* **子任务 5.2：Zone C 的假签名服务**
  * **工作内容**：安全团队在 Zone C 启动一个带有 mTLS 的简单 Web Server。收到 Zone B 请求后，不走硬件 MPC，直接返回一个硬编码的假 Hash。

* **子任务 5.3：假数据推送脚本**
  * **工作内容**：数据工程组写一个简单的 Python 或 Shell 脚本，每秒向 Redis Stream 发送一条符合 `data_stream.proto` 格式的随机假价格事件（含 `block_number` 和 `sync_timestamp` 字段）。

---

## 🏁 Phase 0 阶段合并与验收标准（DoD）

1. **代码无阻碍引用**：Python 工程师和 Go 工程师都能在各自的 IDE 中成功 `import` Protobuf 生成的代码，无编译报错。

2. **网络隔离有效性**：从 Zone A 的测试容器内部执行 `curl` 访问 Zone C 的测试容器，预期结果为**超时/拒绝连接**。

3. **鉴权有效性**：Zone B 使用错误的证书访问 Zone C 的假签名服务，被拒绝；使用正确 mTLS 证书，成功获取假签名。

4. **数据库连接正常**：Go 测试代码能够成功连接 PostgreSQL，向 `trade_tasks` 表插入包含 256 位级别大整数（转 String）的数据，并以 `NUMERIC(78,0)` 精确存储。

5. **可观测性就绪** *(新增)*：Grafana 仪表盘可访问，至少能看到来自假 gRPC Server 的请求日志（带 `trace_id` 字段），Prometheus 已开始抓取至少一个服务的 `/metrics` 端点（即使指标数量为 0）。
