# Phase 0：全局契约定义与基建锁定（修订版 v2.0）

**⏱️ 预计耗时**：1–2 周（强依赖，需集中攻坚）
**🎯 核心目标**：构建"物理管道"，固化跨语言通信的数据结构，产出各团队可直接调用的 Mock/Stub 代码。同步部署可观测性基础设施，确保 Phase 1 起各阶段的开发质量有数据支撑。

> **v2.0 修订说明（相较 v1.1）**
> - 新增：全局文档导航地图（§0）
> - 新增：接口所有权与争议仲裁机制（§A）
> - 新增：环境隔离策略（§B）
> - 新增：密钥与证书生命周期计划（§C）
> - 新增：灾难恢复预案与 RTO/RPO 目标（§D）
> - 新增：各任务架构定位说明
> - 新增：任务依赖关系图
> - 新增：各子任务输入/输出规范
> - 新增：DoD 验收方法与责任人
> - 新增：⛔ 禁止事项统一标注

---

## 📍 本阶段在整体架构中的位置

```
本阶段目标：搭建骨架，三区网络物理隔离，所有通信管道打通 Mock

[Zone A]                    [Zone B]                    [Zone C]
────────────────            ────────────────────────    ──────────────────
★ Python Stub               ★ Go gRPC Mock Server       ★ 假签名服务(mTLS)
  (能发 TradeRequest)          (收到即返回 SIMULATED)       (收到即返回假 Hash)
★ Admin Portal(空壳)        ★ PostgreSQL(Schema就绪)
                            ★ Redis Stream(两个 Topic)
                            ★ Prometheus + Grafana
                            ★ Alertmanager + Loki

★ = 本 Phase 新建    — = 尚未存在    所有模块均为 Mock/空壳
```

**本阶段不涉及**：任何真实 EVM 仿真、任何链上交互、任何真实签名。

---

## §0 全局文档导航地图

> 本节为全项目共享参考，开发者遇到问题时按此索引查阅，避免重复询问。

| 我想了解…… | 应查阅文档 | 关键章节 |
|---|---|---|
| 整体架构与三区划分 | `1.1_三区网络拓扑.md` | §2–§4 三区定义 |
| 容器资源配额与性能目标 | `1.2_容器资源配额.md` | §3.1 Go-Engine 配置 |
| 网络安全规则与 mTLS | `1.3_网络安全策略.md` | §4 跨 Zone 通信规范 |
| Protobuf 接口完整定义 | `2.1_Protobuf契约.md` | §2–§4 三个 proto 文件 |
| Redis Stream 消息格式 | `2.2_Redis_Stream实时管道.md` | §2 消息结构 |
| 历史数据归档规范 | `2.4_历史数据归档.md` | §4 数据格式与分区 |
| 多链数据统一规范 | `2.3_多链数据统一规范.md` | §3 Canonical ID |
| FSM 全部状态与流转条件 | `3.3_幂等状态机FSM.md` | §2 状态定义 |
| EVM 仿真器工作原理 | `3.2_验证层_EVM仿真与三源预言机.md` | §2–§3 |
| Go 仿真器参考实现 | `8.1_Go仿真器参考实现.md` | §2–§4 代码 |
| Python Agent 基类参考 | `8.2_Python_Agent基类参考实现.md` | §2–§3 |
| 策略生命周期 8 个状态 | `策略生命周期定义说明文档.md` | §3 各状态定义 |
| G1–G8 门控通过标准 | `验证门控标准说明文档.md` | §3–§5 |
| 25 个评估指标计算口径 | `评估指标体系说明文档.md` | §3–§7 |
| 双分录记账逻辑 | `4.1_双分录会计系统.md` | §3 标准分录规则 |
| MtM 估值与审计 | `4.2_资产估值与审计.md` | §2–§4 |
| MPC 签名与白名单拦截 | `5.1_MPC签名与白名单拦截.md` | §2–§3 |
| Kill Switch 执行序列 | `5.3_全局断路器与时间锁.md` | §1.3 |
| 时间锁适用边界 | `5.3_全局断路器与时间锁.md` | §2.3 |
| Prometheus 指标命名规范 | `6.2_Prometheus指标定义.md` | §2–§7 |
| 16 条告警规则与阈值 | `6.3_告警规则集.md` | §2–§7 |
| 全链路 Trace ID 规范 | `6.1_全链路Trace_ID规范.md` | §3 日志格式 |
| AI 反馈飞轮机制 | `6.4_AI反馈飞轮.md` | §2–§4 |
| 回测引擎与数据接口 | `7.1_回测引擎与数据接口.md` | §2–§3 |
| MEV 路由与防夹 | `3.4_MEV路由与防夹执行.md` | §2–§3 |
| 数据库 Schema 汇总 | `3.5_数据库Schema汇总.md` | §2–§5 全部表结构 |
| 非功能性量化指标（NFR）| `非功能性量化指标说明文档.md` | 两档延迟预算 |

---

## §A 接口所有权与争议仲裁机制

> **为什么需要这一节？** Phase 2 的并行开发中，多个团队共享同一套接口。若接口定义发生冲突，没有明确仲裁路径会导致协调地狱。

### A.1 接口所有权表

| 接口 / 模块 | 所有者（Owner）| 协作方 | 说明 |
|---|---|---|---|
| `common.proto` / `trading.proto` / `data_stream.proto` | 架构与 SRE 组 | 所有组 | Proto 变更须 Owner 审批，走 PR Review |
| `trade_tasks` / `nonce_locks` 表结构 | 区块链底层组 | 财务风控组 | Schema 变更须双方 Lead 签字后走 Migration |
| `ledger_entries` / `accounts` 表结构 | 财务风控组 | 区块链底层组 | 同上 |
| `protocol_adapters`（Adapter Registry）表 | 区块链底层组 | AI 策略组、安全组 | 新增协议须经安全组白名单审查 |
| Redis Stream Topic 格式 | 数据工程组 | AI 策略组、财务风控组 | 消息格式变更须提前 3 天通知下游消费方 |
| Prometheus 指标命名空间 `defi_` | 架构与 SRE 组 | 所有组 | 新增指标须提 PR，Owner 审批后方可合并 |
| Admin Web Portal API | AI 策略组（Web 方向）| 区块链底层组（API）| 前后端接口契约须双方 Lead 确认 |
| Kill Switch 控制器 | 安全组 | 财务风控组、SRE 组 | IS_KILL_SWITCH 标记签名密钥由安全组独占保管 |

### A.2 争议仲裁路径

```
日常接口分歧（不阻塞进度）：
  → 两方 Tech Lead 在接口 PR 中留评论，24 小时内达成共识

阻塞性分歧（影响当前 Sprint）：
  → 升级至项目技术负责人，当天同步决策，结论写入 ADR（架构决策记录）

跨团队边界争议（如 Zone B 要缓存 Zone C 的白名单）：
  → 走安全组最终审查，安全组 Lead 一票否决权，结论同步全员
```

### A.3 接口变更冻结时间点

| 里程碑 | 冻结内容 |
|---|---|
| Phase 0 DoD 通过后 | Proto 三文件冻结，后续变更须走完整 RFC 流程 |
| Phase 1 DoD 通过后 | DB Schema 核心表冻结，变更须走 SQL Migration |
| Phase 2 DoD 通过后 | Prometheus 指标命名冻结，不允许重命名已上线指标 |

---

## §B 环境隔离策略

### B.1 三套环境定义

| 环境 | 用途 | 链上交互 | 资金 | 说明 |
|---|---|---|---|---|
| **DEV**（本地开发）| Phase 0–2 本地开发与单元测试 | 无（全 Mock）| 无 | 每个开发者本机运行 docker-compose |
| **STAGING**（集成测试）| Phase 3 影子模式、Phase 4 STAGED 阶段 | 测试网（Sepolia/Arbitrum Goerli）| 测试网代币 | 共享环境，SRE 组维护 |
| **PROD**（生产）| Phase 4 LIVE 阶段 | 主网 | 真实资金 | 专用物理机，Zone C 独立隔离 |

### B.2 环境间配置隔离规则

```
⛔ 禁止：PROD 配置（RPC 端点、MPC 密钥路径、主网合约地址）出现在 DEV/STAGING 的任何配置文件中
⛔ 禁止：DEV 环境的测试账户私钥提交至代码仓库（须使用 .env.local，已加入 .gitignore）
✅ 要求：三套环境使用独立的 PostgreSQL 实例，数据库名称带环境前缀（dev_ / staging_ / prod_）
✅ 要求：Adapter Registry 中的合约地址按 chain_id 区分（Mainnet=1，Sepolia=11155111）
✅ 要求：Redis Stream Topic 名称带环境前缀（dev:stream:chain:events / prod:stream:chain:events）
```

### B.3 STAGED 阶段资金账户隔离

- STAGED 阶段使用独立的 EOA 钱包，与主资金池钱包完全分离
- 单次最大拨测金额由安全组预先批准，写入 `staged_max_capital_usd` 配置项
- STAGED 结束后，测试账户余额通过单独审批流程归还主资金池，不走策略 FSM

---

## §C 密钥与证书生命周期计划

> **关键约束**：Phase 0 生成的 mTLS 证书有效期 90 天。如果不提前规划自动化轮换，Phase 3–4 期间证书将过期，导致 Zone B → Zone C 通信中断。

### C.1 证书轮换时间表

| 证书类型 | 生成时间 | 到期时间 | 自动轮换脚本交付时间 | 轮换责任人 |
|---|---|---|---|---|
| Zone B 客户端证书（mTLS）| Phase 0 Week 1 | Phase 0 + 90 天 | **Phase 2 Week 1**（必须在到期前 30 天完成）| 架构与 SRE 组 |
| Zone C 服务端证书（mTLS）| Phase 0 Week 1 | Phase 0 + 90 天 | **Phase 2 Week 1** | 安全组 |
| Intermediate CA 证书 | Phase 0 Week 1 | Phase 0 + 1 年 | Phase 3 前完成年度轮换预案 | 安全组 |

### C.2 证书自动化轮换脚本要求（Phase 2 交付）

```
自动轮换流程：
  1. 到期前 14 天触发告警（Prometheus 指标：defi_cert_expiry_days）
  2. 到期前 7 天自动向 Intermediate CA 申请新证书
  3. 新旧证书并行有效 24 小时（双证书模式，不中断业务）
  4. 旧证书到期后自动吊销
  5. 轮换完成后写审计日志，通知安全组确认
```

### C.3 MPC 密钥分片仪式（Key Ceremony）计划

> 密钥仪式须在 Phase 4 正式 MPC 上线前完成，建议提前 2 周规划。

| 步骤 | 时间 | 参与人 | 输出物 |
|---|---|---|---|
| 确认 N-of-M 方案（如 3-of-5）| Phase 3 末期 | 安全组 + 管理层 | 签署的方案确认书 |
| 购置/准备物理 HSM 设备 | Phase 3 末期 | 安全组 | N 台独立物理机就绪 |
| 执行密钥生成仪式（录像存证）| Phase 4 Week 1 | 安全组 + 见证人 | 分片存储确认、销毁中间材料的视频证明 |
| Zone C 连通性测试 | Phase 4 Week 1 | 安全组 + 区块链底层组 | 测试网签名成功日志 |

### C.4 Root CA 私钥保管规定

- Root CA 私钥**离线存储**，保存于 2 个独立的加密 USB 设备
- USB 设备分别由安全组 Lead 和 CTO/技术负责人各持一份
- Root CA 私钥使用场景：**仅限**签发 Intermediate CA 证书，每次使用须双人到场并记录

---

## §D 灾难恢复预案与 RTO/RPO 目标

### D.1 RTO / RPO 定义

| 组件 | RPO（最大可接受数据丢失）| RTO（最大可接受恢复时间）| 备份策略 |
|---|---|---|---|
| PostgreSQL（账本核心）| 5 分钟 | 30 分钟 | WAL 流式复制 + 每日全量快照，保留 30 个版本 |
| Redis Stream | 0（AOF 持久化，重启后恢复）| 5 分钟 | AOF 持久化 + Consumer Group PEL 重放 |
| 历史 Parquet 数据（MinIO/S3）| 24 小时 | 4 小时 | 跨区域双副本，每日增量备份 |
| Prometheus 指标数据 | 1 小时 | 1 小时 | Prometheus 本地存储（可接受指标丢失，业务不依赖）|

### D.2 历史数据损坏重建预案

> 数据基础设施规范明确：L3/L4 数据可从 L1 原始数据完整重建。

| 损坏层级 | 重建方法 | 预估重建时间 | 重建期间策略状态 |
|---|---|---|---|
| L4（业务语义层）损坏 | 从 L3 重跑聚合计算 | 2–4 小时 | 策略继续运行（L4 仅影响报表）|
| L3（聚合计算层）损坏 | 从 L2 重跑 OHLCV 聚合 | 4–8 小时 | 策略继续运行（实时数据不受影响）|
| L2（协议解析层）损坏 | 从 L1 重跑事件解析 | 8–24 小时 | **策略暂停**，等待 L2 就绪后恢复 |
| L1（链上原始层）损坏 | 从 Archive Node 重新索引 | 1–7 天 | **策略暂停** + 管理员介入 |

### D.3 链重组（Reorg）超过 3 个区块处理预案

```
触发条件：Chain-Adapter 检测到 reorg_depth > 3

处理流程：
  1. 立即暂停所有处于 BROADCASTED 状态的 FSM 任务
  2. 触发 P0 告警，通知 On-Call 工程师
  3. 工程师核查受影响区块范围内的所有 trade_tasks 记录
  4. 对每笔受影响交易：
     - 若链上最终包含该交易 → FSM 推进至 CONFIRMED，正常对账
     - 若链上最终不包含该交易 → 标记为 REORG_DROPPED，释放 Nonce 锁，触发重广播
  5. 财务风控组核查 ledger_entries 是否需要补录或回撤分录
  6. 完成后在审计日志中记录 reorg 事件详情

数据回滚范围：仅回滚受影响区块高度范围内的 L1–L2 索引数据，不回滚账本数据
```

### D.4 PostgreSQL 账本损坏恢复流程

```
触发条件：主实例不可用或数据损坏

恢复步骤：
  1. 切换至只读副本（Read Replica），策略暂停发单，允许查询
  2. 从最近一个全量快照 + WAL 日志重建主实例（目标 < 30 分钟）
  3. 重建完成后，运行 Reconciler 全量对账，确认账本与链上数据一致
  4. 对账通过后，恢复策略发单

注意：WAL 流式复制落后期间（最多 5 分钟的数据）须通过链上 Event Log 重新补录
```

---

## ⛔ 全局禁止事项

> 以下禁令适用于整个项目所有阶段，违反将导致安全漏洞、精度问题或审计失败。首次违规须在 Code Review 中记录，重复违规须走正式缺陷处理流程。

```
金融精度类：
  ⛔ 禁止用 float / double 表示金额（必须 string Wei → big.Int，全程无精度损失）
  ⛔ 禁止用 int64 / int32 表示链上金额（uint256 会溢出）
  ⛔ 禁止回测中使用当前 Gas 价格估算历史成本（必须使用历史 baseFee + priorityFee 记录）

链上数据类：
  ⛔ 禁止策略代码直接调用 RPC（必须通过 SDK DataAPI，由 Go 侧代理）
  ⛔ 禁止使用 eth_call 的 latest 标签（必须指定 block_number，防止重放不一致）
  ⛔ 禁止在回测中读取 target_block + 1 的任何数据（物理级零 Look-Ahead）
  ⛔ 禁止使用 DEX Slot0 即时价作为 MtM 估值基准（必须使用三源预言机中位价）

安全类：
  ⛔ 禁止 Zone A 任何代码持有 Zone C 的通信凭证
  ⛔ 禁止 Zone B 向 Zone C 传送 Hash（必须传明文 Payload，由 Zone C 自行计算 Hash）
  ⛔ 禁止策略代码或 Zone A 自行设置 IS_KILL_SWITCH 标记
  ⛔ 禁止合约白名单在 Whitelist Proxy 内部硬编码（必须从 Adapter Registry 动态读取）
  ⛔ 禁止跨 Zone 连接使用自签名证书（必须由统一内部 CA 签发）

状态机类：
  ⛔ 禁止 FSM 状态变更不经过数据库事务（状态变更与分录写入必须原子）
  ⛔ 禁止跳过 Sync Sentinel 水位线校验直接进入仿真（StaleDataError 是硬拦截，不可绕过）
  ⛔ 禁止自行发明私有 Prometheus 指标命名（必须遵循 defi_ 命名空间规范）
```

---

## 任务 1：API 契约固化与分发

### 📐 架构定位

| 项目 | 内容 |
|---|---|
| **架构位置** | 全系统通信基础层，Zone A ↔ Zone B 的唯一序列化协议 |
| **对应架构文档** | `2.1_Protobuf契约.md` §2–§5 |
| **在请求链路中的位置** | Python Agent → [Proto 序列化] → gRPC → [Proto 反序列化] → Go 执行引擎 |
| **上游依赖** | 无（Phase 0 第一任务） |
| **下游影响** | 所有模块均依赖本任务产出，是最高优先级阻塞点 |
| **接口 Owner** | 架构与 SRE 组 |

### ⛔ 本任务禁止事项

```
⛔ 禁止在 proto 中用 float/double/int64 定义金额字段（使用 string 传 Wei 整数）
⛔ 禁止将 INIT 定义为 TradeStatus 的零值（零值必须是 TRADE_STATUS_UNSPECIFIED = 0）
⛔ 禁止在未经 Owner 审批的情况下修改已发布的 proto 字段编号
```

---

### 子任务 1.1：编写 `common.proto`

**输入：**
- `2.1_Protobuf契约.md` §2 中的完整类型定义规范
- `策略生命周期定义说明文档.md` §3 中的 TradeStatus 枚举定义

**输出：**
- `common.proto` 文件，包含：`BigAmount`（string raw_value + int32 decimals + string symbol）、`Context`（trace_id + agent_id + last_sync_block + timestamp_ms）、`TradeStatus` 枚举（UNSPECIFIED=0，业务状态从 1 起）
- 文件顶部注释：明确说明 BigAmount.raw_value 必须为 string 类型的 Wei 原始整数，严禁 float/double

**不产出（明确排除）：**
- 不包含任何 RPC 服务定义（服务定义在 trading.proto 和 data_stream.proto 中）

---

### 子任务 1.2：编写 `trading.proto`

**输入：**
- `2.1_Protobuf契约.md` §3 中的 TradingEngine 服务定义
- `策略生命周期定义说明文档.md` §6 SDK 接口约定

**输出：**
- `trading.proto` 文件，包含：`TradingEngine` service（ExecuteTrade / ShadowTrade / GetSystemStatus 三个 RPC）、`TradeRequest`（含 ctx / protocol_id / action / payload bytes / max_slippage_bps / gas_price_limit）、`TradeResponse`、`SystemStatus`

**不产出（明确排除）：**
- 不包含链上事件订阅相关定义（在 data_stream.proto 中）

---

### 子任务 1.3：编写 `data_stream.proto`

**输入：**
- `2.1_Protobuf契约.md` §4 中的 DataSentinel 服务定义
- `2.2_Redis_Stream实时管道.md` §2 消息结构

**输出：**
- `data_stream.proto` 文件，包含：`DataSentinel` service（SubscribeEvents server-streaming RPC）、`SubscribeRequest`、`ChainEvent`（block_number 作为水位线字段）

---

### 子任务 1.4：编译与分发环境搭建

**输入：**
- 上述三个 .proto 文件
- `go_package` 选项：`github.com/holobit/proto/v1;defiv1`

**输出：**
- Python `_pb2.py` / `_pb2_grpc.py` 桩代码
- Go `.pb.go` / `_grpc.pb.go` 桩代码
- 内部私有包仓库（Git Submodule），两端工程师可直接 import

**验证方法：**
- Python：`from trading_pb2 import TradeRequest; r = TradeRequest(); assert r.ctx.trace_id == ""`
- Go：`import defiv1 "github.com/holobit/proto/v1"; _ = defiv1.TradeRequest{}`，`go build ./...` 无报错

---

## 任务 2：金融级数据库 Schema 锁定

### 📐 架构定位

| 项目 | 内容 |
|---|---|
| **架构位置** | Zone B → PostgreSQL，是账本和 FSM 状态的持久化基础 |
| **对应架构文档** | `3.5_数据库Schema汇总.md` §2–§5 全部表结构 |
| **在请求链路中的位置** | FSM 状态推进 → [写 trade_tasks] → 对账 → [写 ledger_entries] |
| **上游依赖** | 无（可与任务 1 并行） |
| **下游影响** | Phase 2 所有涉及 FSM、账本、Nonce 的开发均依赖此 Schema |
| **接口 Owner** | 财务风控组（账本表）+ 区块链底层组（交易表） |

### ⛔ 本任务禁止事项

```
⛔ 禁止用 DECIMAL(18,8) 或 FLOAT 定义金额字段（必须 NUMERIC(78,0) 支持 uint256）
⛔ 禁止用 VARCHAR 定义 status 字段（必须用 ENUM 或 CHECK 约束，与 TradeStatus 枚举严格对应）
⛔ 禁止时间字段用 TIMESTAMP 不带时区（必须 TIMESTAMP WITH TIME ZONE）
⛔ Schema 冻结后（Phase 0 DoD 通过后），任何字段变更必须通过 SQL Migration 脚本执行，禁止手动修改
```

---

### 子任务 2.1：初始化交易核心表

**输入：**
- `3.5_数据库Schema汇总.md` §2 trade_tasks 表定义
- `3.3_幂等状态机FSM.md` §2 状态枚举

**输出：**
- `trade_tasks` 表 DDL：含 id（UUID PK）、strategy_id、trace_id、status（ENUM，与 TradeStatus 对应）、nonce、tx_hash、internal_tx_id、created_at / updated_at（TIMESTAMP WITH TIME ZONE）
- `nonce_locks` 表 DDL：含 address、chain_id、nonce（BIGINT）、locked_by、locked_at、expires_at；主键为 (address, chain_id)
- 索引：`trade_tasks(status, strategy_id)`、`trade_tasks(trace_id)`、`trade_tasks(tx_hash)`

**不产出（明确排除）：**
- 不包含账本类表（由子任务 2.2 负责）

---

### 子任务 2.2：初始化双分录会计核心表

**输入：**
- `3.5_数据库Schema汇总.md` §3–§5 账本表定义
- `4.1_双分录会计系统.md` §2 账户体系

**输出：**
- `accounts` 表 DDL：含 id（UUID PK）、account_type（STRATEGY/PROTOCOL/GAS/BRIDGE_IN_TRANSIT）、strategy_id、protocol_id、chain_id
- `ledger_entries` 表 DDL：含 id、internal_tx_id、trace_id、debit_account_id、credit_account_id、amount（NUMERIC(78,0)）、entry_type（ENUM：TRADE/GAS_FEE/PROTOCOL_FEE/REWARD/SLIPPAGE/MEV_LOSS）、created_at
- `asset_balances` 表 DDL：含 account_id、token_address、chain_id、raw_balance（NUMERIC(78,0)）、last_sync_block
- `unclaimed_rewards` 表 DDL：含 strategy_id、protocol_id、token_address、raw_amount（NUMERIC(78,0)）、last_sync_block
- CHECK 约束：`ledger_entries` 中每个 `internal_tx_id` 对应的所有条目 `SUM(debit_amount) = SUM(credit_amount)`（通过触发器或应用层保证）

---

### 子任务 2.3：部署与权限分配

**输入：**
- 上述所有 DDL 文件
- 环境隔离策略（§B）中的数据库命名规范

**输出：**
- DEV 环境 PostgreSQL 实例就绪（数据库名：`dev_defi_core`）
- Go-Execution-Engine 读写账号（权限：`trade_tasks`、`nonce_locks`、`ledger_entries`、`asset_balances`、`unclaimed_rewards` 的 SELECT/INSERT/UPDATE）
- 只读账号（供 Admin Web Portal 查询用）
- 连接池初始配置：`max_connections = 200`，`idle_timeout = 10min`

---

## 任务 3：隔离网络与中间件拉起

### 📐 架构定位

| 项目 | 内容 |
|---|---|
| **架构位置** | 三区网络物理骨架，所有组件的通信基础设施 |
| **对应架构文档** | `1.1_三区网络拓扑.md` §2–§4、`1.3_网络安全策略.md` §2–§3 |
| **在请求链路中的位置** | 所有跨 Zone 通信的物理载体 |
| **上游依赖** | 无（可与任务 1、2 并行） |
| **下游影响** | Zone C 假签名服务依赖 mTLS 体系；Redis Stream 是数据流的骨干 |
| **接口 Owner** | 架构与 SRE 组 |

### ⛔ 本任务禁止事项

```
⛔ 禁止 fortress_net（Zone C）设置 internal: false（必须 internal: true，物理阻断出站公网）
⛔ 禁止 intelligence_net（Zone A）容器路由至 fortress_net（Zone C）（网络层物理阻断）
⛔ 禁止使用自签名证书用于 mTLS（必须由统一内部 CA 签发）
⛔ 禁止 Redis 使用无密码的默认配置（须配置 requirepass）
```

---

### 子任务 3.1：配置三区 Docker 网络

**输入：**
- `1.3_网络安全策略.md` §2.1 完整 Docker 网络配置
- §B 环境隔离策略

**输出：**
- `docker-compose.yml` 网络定义段：`intelligence_net`（10.0.1.0/24，internal: false）、`sandbox_net`（10.0.2.0/24，internal: false）、`fortress_net`（10.0.5.0/24，**internal: true**）
- 容器网络归属配置：go-execution-engine 双归属 sandbox_net + intelligence_net；mpc-signer-node 仅归属 fortress_net

**验证方法：**
```bash
# 从 Zone A 容器 ping Zone C，预期超时
docker exec python-agent-1 curl -m 3 http://mpc-signer:8443 || echo "BLOCKED(预期)"
# 从 Zone B 容器 ping Zone C，预期可达（mTLS 层才拒绝）
docker exec go-execution-engine curl -m 3 --insecure https://mpc-signer:8443 | grep "certificate"
```

---

### 子任务 3.2：建立 mTLS 证书体系

**输入：**
- `1.3_网络安全策略.md` §5 内部 CA 体系
- §C 密钥与证书生命周期计划

**输出：**
- Root CA 证书（离线存储，仅用于签发 Intermediate CA）
- Intermediate CA - Zone B（签发 Zone B 客户端证书）
- Intermediate CA - Zone C（签发 Zone C 服务端证书）
- Zone B 客户端证书（有效期 90 天）→ 挂载至 go-execution-engine 和 mev-router 的 Docker Secret
- Zone C 服务端证书（有效期 90 天）→ 挂载至 mpc-signer 的 Docker Secret
- **证书到期提醒事项**：在项目 Milestone 日历中标记"Phase 0 + 76 天"为证书轮换截止日（留 14 天余量）

**验证方法：**
```bash
# 使用错误证书访问 Zone C，预期被拒绝
curl --cert wrong.crt --key wrong.key https://mpc-signer:8443  # 预期: SSL handshake failed
# 使用正确证书访问，预期成功握手（内容为假签名 Mock 响应）
curl --cert zone_b.crt --key zone_b.key --cacert ca.crt https://mpc-signer:8443  # 预期: {"fake_sig": "0x..."}
```

---

### 子任务 3.3：拉起 Redis 消息总线

**输入：**
- `2.2_Redis_Stream实时管道.md` §2 Stream 结构
- `1.2_容器资源配额.md` §3.2 Redis 配置（2 Core / 4 GB / AOF 持久化）

**输出：**
- Redis Cluster 实例就绪，AOF 持久化模式开启
- 初始化脚本显式创建两个 Stream：
  - `stream:chain:events`（链上事件总线，MAXLEN ~10000）
  - `stream:internal:accounting`（财务审计事件，MAXLEN ~5000）
- 两个 Consumer Group：`cg:agents`（50+ Agent 消费 chain events）和 `cg:accounting`（财务模块消费 accounting events）

---

## 任务 4：可观测性基础设施部署

### 📐 架构定位

| 项目 | 内容 |
|---|---|
| **架构位置** | Zone B 横切关注点，所有服务的指标和日志汇聚层 |
| **对应架构文档** | `6.1_全链路Trace_ID规范.md`、`6.2_Prometheus指标定义.md`、`6.3_告警规则集.md` |
| **在请求链路中的位置** | 所有模块的 /metrics 端点 → Prometheus → Grafana；所有模块的结构化日志 → Loki |
| **上游依赖** | 无（可与任务 1–3 并行） |
| **下游影响** | Phase 1 起所有 DoD 的量化验收均依赖本模块数据 |
| **接口 Owner** | 架构与 SRE 组 |

---

### 子任务 4.1：部署 Prometheus + Alertmanager

**输入：**
- `6.2_Prometheus指标定义.md` §8.1 抓取配置
- `6.3_告警规则集.md` 全部 16 条告警规则

**输出：**
- `prometheus.yml`：配置三个 scrape target（go-execution-engine:9090 / python-agent:9091 / mev-router:9092），抓取间隔 15s
- `alertmanager.yml`：CRITICAL 走 PagerDuty + Slack，WARNING 走 Slack，告警路由按 severity + group 分组
- 16 条告警规则录入（此时多数指标尚未埋点，规则处于 inactive 状态，随各模块交付逐步激活）
- `defi_cert_expiry_days` 指标预留（§C 证书轮换监控）

---

### 子任务 4.2：部署 Grafana + Loki 日志聚合

**输入：**
- `6.1_全链路Trace_ID规范.md` §3 标准日志格式（必须包含 trace_id、component、sync_block 字段）
- `6.2_Prometheus指标定义.md` §9 仪表盘建议

**输出：**
- Grafana 实例就绪，接入 Prometheus 和 Loki 两个数据源
- Loki 配置各容器日志收集，强制 JSON 结构化格式
- 4 个初始仪表盘空壳（系统健康 / 交易执行 / 财务 / 安全），各模块随交付填充数据
- Grafana 访问地址文档化，分发给所有团队

---

### 子任务 4.3：Prometheus 指标埋点规范分发

**输入：**
- `6.2_Prometheus指标定义.md` 全文

**输出：**
- 一份面向开发者的埋点规范文档（2 页）：命名规范（defi_{subsystem}_{name}）、Label 规范、Histogram Bucket 配置、Exemplars 注入规范（用于关联 trace_id）
- 分发给所有组 Tech Lead，各组须签字确认已阅读
- 明确：**禁止自行发明私有指标命名**，新增指标须提 PR 经 SRE Owner 审批

---

## 任务 5：制造假通道（Mocking）以解除阻塞

### 📐 架构定位

| 项目 | 内容 |
|---|---|
| **架构位置** | 各 Zone 间通信管道的临时替代实现 |
| **对应架构文档** | 本阶段特有，Phase 2 完成后全部废弃 |
| **在请求链路中的位置** | Python Agent → [假 gRPC Server] → [假签名服务] → 假 tx_hash |
| **上游依赖** | 任务 1（Proto 定义）、任务 3（网络和 mTLS）|
| **下游影响** | 解除 Phase 1 的开发阻塞，让 AI 策略组可立即开始编写 BaseStrategyAgent |
| **接口 Owner** | 各组 Tech Lead 协作，SRE 组统一部署 |

---

### 子任务 5.1：Zone B 假 gRPC Server

**输入：**
- `trading.proto` 编译产出的 Go 桩代码

**输出：**
- Go gRPC 服务，监听 50051 端口
- 收到 `TradeRequest` 后：打印 `[MOCK] received trace_id={trace_id}, agent_id={agent_id}`；校验 `trace_id` 非空（否则返回 INVALID_ARGUMENT）；直接返回 `TradeResponse{status: SIMULATED, internal_tx_id: uuid.New()}`
- 同时向 Prometheus 写入请求计数（即使是 Mock，也验证指标通路正常）

---

### 子任务 5.2：Zone C 假签名服务

**输入：**
- mTLS 证书（任务 3.2 产出）

**输出：**
- Go HTTP Server，监听 8443 端口，配置 mTLS
- 收到请求后返回：`{"fake_sig": "0xdeadbeef...", "trace_id": "{input_trace_id}"}`
- 不做任何白名单校验（Mock 阶段，Phase 4 替换为真实实现）

---

### 子任务 5.3：假数据推送脚本

**输入：**
- `data_stream.proto` 中的 `ChainEvent` 消息格式
- Redis Stream Topic 名称

**输出：**
- Python 脚本 `scripts/mock_chain_events.py`：每秒向 `stream:chain:events` 写入一条随机 `ChainEvent`，包含递增的 `block_number`（模拟水位线）和 `sync_timestamp`
- 脚本支持 `--lag N` 参数，模拟水位线滞后 N 个区块（用于测试 Sync Sentinel 熔断逻辑）

---

## 任务依赖关系

```
任务 1（Proto 契约）
  └─ 阻塞 → 任务 5.1（假 gRPC Server 需要 Go 桩代码）
  └─ 阻塞 → 任务 5.2（假签名服务使用 Proto 消息格式）
  └─ 阻塞 → Phase 1 任务 4（Python Agent 基类需要 Proto）

任务 2（DB Schema）
  └─ 阻塞 → Phase 2 任务 3（FSM 和 Nonce 锁依赖 trade_tasks 表）
  └─ 阻塞 → Phase 2 任务 4（双分录账本依赖 ledger_entries 表）

任务 3（网络 + mTLS）
  └─ 阻塞 → 任务 5.2（假签名服务需要 mTLS 证书）
  └─ 阻塞 → 任务 5.3（假数据推送需要 Redis 就绪）

任务 4（可观测性）
  └─ 无直接阻塞，但强烈建议在任务 1–3 完成前并行交付
  └─ 解锁 → Phase 1 DoD 中的可观测性验收项

任务 5（Mocking）
  └─ 依赖任务 1 + 任务 3
  └─ 解除阻塞 → Phase 1 所有 AI 策略组开发任务

可并行执行：任务 1 + 任务 2 + 任务 3 + 任务 4（四任务可同时启动）
串行依赖：任务 5 必须在任务 1 和任务 3 完成后才能启动
```

---

## 🏁 Phase 0 阶段验收标准（DoD）

| # | 验收标准 | 验收方法 | 验收责任人 |
|---|---|---|---|
| 1 | Proto 无阻碍引用：Python 和 Go 均可成功 import 生成的桩代码 | Python: `python -c "from trading_pb2 import TradeRequest"`；Go: `go build ./...`，均无报错 | 架构与 SRE 组 |
| 2 | 网络隔离有效：Zone A 无法访问 Zone C | `docker exec python-agent-1 curl -m 3 http://mpc-signer:8443`，预期结果：连接超时或拒绝 | 架构与 SRE 组 |
| 3 | mTLS 鉴权有效：错误证书被拒，正确证书通过 | 分别用 `wrong.crt` 和 `zone_b.crt` 访问假签名服务，前者 SSL 握手失败，后者返回假 Hash | 安全组 |
| 4 | 数据库精度正常：256 位大整数精确存储 | Go 测试代码向 `trade_tasks` 插入 `amount = "115792089237316195423570985008687907853269984665640564039457584007913129639935"`（2^256-1），读回后字符串完全一致 | 财务风控组 |
| 5 | 可观测性就绪：Grafana 可访问，至少一个 /metrics 端点被抓取 | 访问 Grafana URL，Loki 中能检索到假 gRPC Server 的请求日志（含 trace_id 字段）；Prometheus Targets 页面至少一个 target 为 UP | 架构与 SRE 组 |
| 6 | 假通道联通：端到端 Mock 调用完整 | Python 脚本发送 TradeRequest → 假 gRPC Server 返回 SIMULATED → 触发假签名请求（经 mTLS）→ 返回假 Hash；全链路 trace_id 一致 | 各组 Tech Lead |
| 7 | 接口所有权表已签发 | 所有 Tech Lead 在接口所有权表上确认签字（电子签名或 Git commit 确认）| 项目技术负责人 |
| 8 | 环境隔离规则已知悉 | 所有开发者完成环境隔离策略（§B）的阅读确认；DEV 环境配置文件中无任何主网 RPC 地址 | 架构与 SRE 组 |
