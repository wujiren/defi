## 🏛️ 部署架构：基于 Docker 的区域隔离与分发

### 1. 三层逻辑区域的容器分布 (The 3-Zone Topology)

我们将容器按安全等级划分为三个物理或逻辑隔离的 Docker 网络环境：

#### **Zone A：策略执行区 (Intelligence Network)**

* **Python-Agent-Pool (50+ Containers)**：
* **职责**：运行具体的 AI 策略逻辑。
* **配置**：每个容器独立运行，通过 gRPC 向上游 Go 服务发起请求。
* **网络限制**：仅允许出站访问 L3 消息总线（Redis）和 Go 端的 gRPC 接口。


* **AI-Feedback-Service**：处理模块 6.2 的 Prompt 反馈流。

#### **Zone B：验证与状态区 (Sandbox & State Network)**

* **Go-Execution-Engine (Core)**：
* **职责**：运行 FSM 状态机、EVM 仿真器和双分录对账逻辑。
* **容器特性**：高优先级（High Priority），配置核心转储以便故障复盘。


* **Infrastructure Cluster**：
* **PostgreSQL**：持久化交易任务和财务账本。
* **Redis-Cluster**：作为 L3 消息总线，处理高频数据分发。
* **Anvil-Node**：运行本地分叉环境进行 G4/G6 级压力测试。



#### **Zone C：核心安全区 (Fortress Network)**

* **MPC-Signer-Node**：
* **职责**：执行私钥分片签名。
* **网络限制**：**物理隔离级别**。仅接受来自 Zone B 的经过白名单校验的签名请求。


* **Admin-Approval-API**：处理大额交易的人工审批工作流。

---

### 2. 容器资源规格与配额 (Resource Constraints)

为了保证 **P99 < 500ms** 的性能底线，必须严格限制每个容器的资源，防止 Python Agent 的 OOM 或计算饥饿影响全局。

| 容器名称 | 基础镜像 | CPU 配额 | 内存配额 | 存储要求 |
| --- | --- | --- | --- | --- |
| **Go-Core-Engine** | `golang:1.22-alpine` | 4.0 Cores | 8GB | 高速 NVMe (用于 StateDB 缓存) |
| **Python-Agent** | `python:3.11-slim` | 1.0 Cores | 2GB | 仅读只读挂载 (策略逻辑) |
| **Redis-Stream** | `redis:7.2-alpine` | 2.0 Cores | 4GB | 内存持久化 AOF |
| **Anvil-Simulator** | `foundry-rs/foundry` | 2.0 Cores | 4GB | 临时内存文件系统 |

---

### 3. Docker 网络拓扑与安全 (Networking)

通过 Docker 自定义网络实现 PRD 定义的“物理级隔离”逻辑：

```yaml
networks:
  intelligence_net:
    internal: true # 限制 Agent 容器访问外网，仅限 Zone 内通讯
  sandbox_net:
    driver: bridge
  fortress_net:
    ipam:
      config:
        - subnet: 10.0.5.0/24 # 极高安全级别网段

```

* **隔离策略**：Zone A 的 Python 容器**严禁**直接连接 Zone C 的签名机。
* **通讯契约**：所有跨容器调用必须强制执行 `.proto` 定义的强类型检查，并在网关层通过 Trace ID 进行打标。

---

### 4. 评价标准 (Evaluation Standards)

| 维度 | **好部署架构 (Industrial Grade)** | **坏部署架构 (Junior)** |
| --- | --- | --- |
| **故障隔离** | 某个 Python Agent 内存溢出不会影响 Go 端的 Nonce 锁管理器。 | 所有服务跑在一个容器或一个进程里，一挂全挂。 |
| **伸缩性** | 通过 `docker-compose up --scale` 可以在数秒内横向扩展 50 个策略实例。 | 扩展新策略需要手动修改配置文件并重启核心引擎。 |
| **安全性** | 签名私钥分片所在的容器永远无法直接被 AI Agent 访问。 | 策略层可以直接调用签名接口，缺乏物理拦截。 |
