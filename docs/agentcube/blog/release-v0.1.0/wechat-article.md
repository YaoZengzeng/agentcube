# AgentCube v0.1.0 正式发布：让 AI Agent 成为 Kubernetes 的一等公民

AgentCube 是 Volcano 社区的子项目，将 AI Agent 和代码解释器建模为 Kubernetes 原生的 Serverless 工作负载。v0.1.0 是 AgentCube 的首个正式版本。

---

## 为什么需要 AgentCube？

AI Agent 正在改变软件的构建和运行方式。对话式智能体、自动化代码执行、多步推理与工具调用——这些场景都需要一种**有状态、可隔离、低延迟**的运行时环境。

然而，Kubernetes 体系是为**长驻微服务**设计的。面对高频创建、短时运行、需要会话级隔离的 Agent 工作负载，两者之间存在显著的鸿沟：

| 痛点 | 描述 |
|------|------|
| **冷启动延迟高** | 每次 Agent 会话都要拉起完整 Pod，交互式体验受阻且运行成本显著增加 |
| **运行时管理混乱** | Agent 运行时散落在业务代码中，缺乏统一的安全策略与生命周期管理 |
| **资源利用率低** | 轻量级突发型 Agent 负载使用重量级基础设施原语，浪费 CPU/GPU 资源 |
| **缺少统一抽象** | 没有 Kubernetes 原生的方式来声明、启动和管理 Agent 会话 |

AgentCube 的目标就是填补这个空白。它将 AI Agent 和代码解释器建模为 Kubernetes 的**一等公民**——像 Serverless 函数一样按需调度、自动伸缩，同时拥有微虚拟机级别的强隔离保障。

---

## 整体架构

AgentCube 的架构分为三层：**数据平面（Router）**、**控制平面（Workload Manager）** 和 **沙箱运行时（PicoD）**，通过 Redis/ValKey 共享会话状态，实现各组件的水平扩展。

![AgentCube 整体架构图](./images/agentcube.svg)

<p align="center"><em>图 1：AgentCube 整体架构 —— Router 接收请求并路由至沙箱，Workload Manager 管理沙箱生命周期，PicoD 负责沙箱内执行</em></p>

核心组件一览：

| 组件 | 角色 | 关键能力 |
|------|------|----------|
| **Router** | 数据平面入口 | HTTP 反向代理、会话路由、JWT 签名、并发控制 |
| **Workload Manager** | 控制平面 | 沙箱创建/删除、预热池管理、双策略 GC |
| **PicoD** | 沙箱内守护进程 | 代码执行、文件 I/O、JWT 认证、路径沙箱化 |
| **Session Store** | 状态存储 | Redis/ValKey 支持，Sorted Set 索引加速查询 |

一次完整的调用流程如下：

```
┌─────────┐     ① HTTP 请求         ┌──────────┐     ② 查询/创建沙箱    ┌──────────────────┐
│  Client  │ ───────────────────────▶│  Router  │ ──────────────────────▶│ Workload Manager │
└─────────┘  x-agentcube-session-id  └──────────┘                       └──────────────────┘
                                          │                                     │
                                          │ ④ JWT签名 + 反向代理                 │ ③ 创建/领取沙箱 Pod
                                          ▼                                     ▼
                                    ┌──────────┐                        ┌───────────────┐
                                    │  PicoD   │ ◀───── 运行在 ────────│  Sandbox Pod  │
                                    └──────────┘                        └───────────────┘
                                          │
                                          │ ⑤ 验证 JWT → 执行命令 → 返回结果
                                          ▼
                                    stdout / stderr / exit_code
```

---

## 核心特性深度解读

### 一、Session-Based MicroVM Agent Routing：有状态的 Agent 路由

AI Agent 工作负载本质上是**有状态和交互式**的。一个 Agent 会话可能横跨多次调用——工具调用、环境探查、多步推理——每一步都需要同一个隔离执行环境的上下文保持。

AgentCube 通过 **Session ID → 沙箱 Pod** 的映射解决了这一问题：

- 客户端首次调用时不携带 `x-agentcube-session-id` 头，Router 自动通过 Workload Manager 分配新沙箱
- 响应中返回 `x-agentcube-session-id`，客户端后续请求携带此 ID 即可复用同一沙箱
- 整个过程对客户端**完全透明**，无需任何沙箱配置知识

从代码实现上看，Router 基于 **Gin 框架**构建，核心路由 handler `handleInvoke()` 的处理逻辑如下：

```go
// 从请求头提取 Session ID
sessionID := c.GetHeader("x-agentcube-session-id")

// 通过 SessionManager 查找或创建沙箱
sandbox, err := s.sessionManager.GetSandboxBySession(
    c.Request.Context(), sessionID, namespace, name, kind)

// 更新会话最后活跃时间（用于 GC 判定）
s.storeClient.UpdateSessionLastActivity(ctx, sandbox.SessionID, time.Now())

// 反向代理转发至沙箱
s.forwardToSandbox(c, sandbox, path)
```

Router 还支持 **HTTP/2 (h2c)** 透传以降低连接延迟，并内置可配置的**并发请求上限**（默认 1000），防止突发流量冲垮沙箱集群。

**Router 暴露的 API 端点：**

```
POST /v1/namespaces/{ns}/agent-runtimes/{name}/invocations/*path
POST /v1/namespaces/{ns}/code-interpreters/{name}/invocations/*path
```

---

### 二、Agent as First-Class Citizen：两种 CRD，两种工作负载范式

AgentCube 引入了两个新的 Kubernetes CRD，对应 AI Agent 领域的两类典型工作负载：

#### AgentRuntime —— 通用 AI Agent 运行时

面向需要丰富 Kubernetes 能力的对话式/工具调用型 Agent。接受完整的 `PodSpec` 模板，可以挂载 Volume、注入凭据、配置 Sidecar 容器：

```yaml
apiVersion: runtime.agentcube.volcano.sh/v1alpha1
kind: AgentRuntime
metadata:
  name: my-agent
spec:
  podTemplate:
    # 完整 PodSpec —— Volume、ServiceAccount、Sidecar 均可配置
    containers:
      - name: agent
        image: my-agent:latest
        ports:
          - containerPort: 8080
  targetPort:
    - pathPrefix: "/"
      port: 8080
      protocol: HTTP
  sessionTimeout: 15m        # 空闲超时
  maxSessionDuration: 8h     # 绝对最大时长
```

#### CodeInterpreter —— 安全代码解释器

面向多租户安全代码执行场景（Notebook、REPL、"运行代码"按钮）。相比 AgentRuntime 更加锁定，使用受约束的 `CodeInterpreterSandboxTemplate`，限制镜像、资源和运行时类：

```yaml
apiVersion: runtime.agentcube.volcano.sh/v1alpha1
kind: CodeInterpreter
metadata:
  name: my-interpreter
spec:
  template:
    image: ghcr.io/volcano-sh/picod:latest
    runtimeClassName: kata      # 可选，指定 Kata/kuasar 隔离级别
    resources:
      requests: { cpu: "500m", memory: "512Mi" }
      limits:   { cpu: "2",    memory: "2Gi"   }
  warmPoolSize: 3               # 预热池大小
  authMode: picod               # RSA/JWT 认证（默认）
  sessionTimeout: 15m
  maxSessionDuration: 8h
```

两者的设计差异体现了"安全默认 vs 灵活可控"的取舍：用 `AgentRuntime` 拥抱 Kubernetes 的全部能力，用 `CodeInterpreter` 获得开箱即用的安全沙箱。

---

### 三、Warm Pool：预热池消除冷启动

交互式 Agent 场景下，从零创建微虚拟机沙箱带来的冷启动延迟是不可接受的。AgentCube 引入了**预热池机制**：

```
┌────────────────────────────────────────────────────────────┐
│                    Warm Pool (预热池)                        │
│                                                            │
│   ┌─────────┐   ┌─────────┐   ┌─────────┐                │
│   │ Sandbox │   │ Sandbox │   │ Sandbox │   ← 空闲待命     │
│   │  (Idle) │   │  (Idle) │   │  (Idle) │                 │
│   └─────────┘   └─────────┘   └─────────┘                │
│        │                                                   │
│        ▼  新会话到来 → SandboxClaim 认领                     │
│   ┌─────────┐                                              │
│   │ Sandbox │   ← 立即投入使用，零等待                        │
│   │ (Active)│                                              │
│   └─────────┘                                              │
│                                                            │
│   池自动异步补充，保持 steady-state                            │
└────────────────────────────────────────────────────────────┘
```

实现上，Workload Manager 中的 `CodeInterpreterReconciler` 通过 **SandboxTemplate + SandboxWarmPool + SandboxClaim** 三层 CRD 协作：

1. **`SandboxTemplate`**：定义沙箱 Pod 模板，注入 PicoD 认证公钥
2. **`SandboxWarmPool`**：声明预热副本数（`replicas = spec.warmPoolSize`）
3. **`SandboxClaim`**：新会话到来时，通过 Claim 模式从池中"认领"一个已就绪的沙箱

整个过程是**幂等**的——Reconciler 在每次调和循环中检查预热池状态，仅在状态不一致时做最小变更，与 Kubernetes 声明式设计一脉相承。

---

### 四、PicoD：替代 SSH 的轻量级沙箱守护进程

传统代码沙箱方案依赖 SSH 进行远程执行，但 SSH 带来了过重的协议开销：密钥管理、多路复用协商、持久会话维护……对于本质上是**单请求 RPC** 的场景而言，这些都是不必要的负担。

PicoD (Pico Daemon) 用一个仅需几百 KB 的 HTTP 守护进程替代了 SSH，通过 RESTful API 完成沙箱内的所有操作：

| API 端点 | 方法 | 功能 |
|----------|------|------|
| `/api/execute` | POST | 执行任意命令，支持超时、工作目录、环境变量 |
| `/api/files` | POST | 文件上传/写入（multipart 或 base64 JSON） |
| `/api/files/*path` | GET | 文件下载/读取，流式传输 |
| `/health` | GET | 健康检查（免认证） |

代码执行的实现遵循了**安全优先**原则：

```go
// 默认 60 秒超时，可通过请求自定义
ctx, cancel := context.WithTimeout(ctx, timeout)
defer cancel()

cmd := exec.CommandContext(ctx, command[0], command[1:]...)
cmd.Dir = workspaceDir      // 强制工作目录
cmd.Env = mergedEnv         // 环境变量隔离

// 超时返回退出码 124（兼容 GNU timeout 标准）
if ctx.Err() == context.DeadlineExceeded {
    exitCode = 124
}
```

安全防护措施包括：
- **路径沙箱化**：所有文件操作被限制在配置的 workspace 根目录下，通过 `sanitizePath()` 函数阻止目录穿越攻击
- **请求体限制**：32 MB 上限，防止内存耗尽攻击
- **无状态设计**：每个请求独立处理，无持久连接和会话追踪

---

### 五、JWT 安全链：Router → PicoD 认证

沙箱 Pod 是临时的，随时可能被替换。在集群配置中嵌入共享密钥既脆弱又难以轮换。AgentCube 建立了一条基于 RSA 非对称加密的信任链：

```
┌──────────────┐                     ┌─────────────────────────┐
│    Router    │                     │     Kubernetes Secret   │
│              │  启动时生成           │  picod-router-identity  │
│  RSA-2048    │ ──────────────────▶ │                         │
│  密钥对      │                     │  private.pem (Router用)  │
│              │                     │  public.pem  (PicoD用)   │
└──────┬───────┘                     └───────────┬─────────────┘
       │                                         │
       │ 用私钥签发                                │ Workload Manager
       │ 5分钟有效JWT                              │ 注入环境变量
       │                                         │ PICOD_AUTH_PUBLIC_KEY
       ▼                                         ▼
┌──────────────┐    RS256 JWT Token       ┌──────────────┐
│   请求签名    │ ──────────────────────▶  │    PicoD     │
│   (私钥)     │                          │  验证签名     │
└──────────────┘                          │  (公钥)       │
                                          └──────────────┘
```

关键安全设计：
- **5 分钟超短有效期**：即使 Token 泄漏，爆炸半径也极小
- **1 分钟时钟偏移容忍**：PicoD 验证时允许合理的时钟漂移
- **私钥不出 Router**：仅 Router 持有私钥，PicoD 仅需公钥验证
- **自动密钥管理**：启动时自动生成，无需运维手动配置证书

---

### 六、双策略 GC：沙箱资源的自动回收

Agent 会话终止或被客户端遗弃后，必须自动回收资源以避免资源耗尽。AgentCube 在 Workload Manager 中实现了**双重垃圾回收策略**：

| 策略 | 触发条件 | 默认值 |
|------|----------|--------|
| **空闲超时（Idle TTL）** | 沙箱在 `sessionTimeout` 内无任何请求 | 15 分钟 |
| **绝对最大时长（Max Duration）** | 沙箱创建时间超过 `maxSessionDuration` | 8 小时 |

GC 循环每 **15 秒**执行一次，每轮最多审查 **100 个候选沙箱**（防止单次 GC 阻塞过长）。底层使用 Redis Sorted Set 的 `ZRANGEBYSCORE` 高效查询过期和不活跃的沙箱：

```
Redis 数据结构：
  session:{sessionID}       → SandboxInfo JSON     # 会话详情
  session:expiry            → Sorted Set (score=到期时间戳)   # TTL 索引
  session:last_activity     → Sorted Set (score=最后活跃时间)  # 活跃度索引
```

删除操作是**原子化**的：GC 同时清理 Kubernetes 中的 Sandbox/SandboxClaim CR 和 Redis 中的会话记录，确保不会出现"幽灵沙箱"或"孤儿记录"。

---

## 生态集成

AgentCube v0.1.0 为主流 AI 框架提供了现成的集成方案：

### Python SDK

```python
from agentcube import CodeInterpreterClient

interpreter = CodeInterpreterClient()

# 执行代码
result = interpreter.run_code("python", "print('Hello, AgentCube!')")
print(result)

# 文件操作
interpreter.write_file(content="data", remote_path="/workspace/data.txt")
interpreter.download_file("/workspace/data.txt", "./local_data.txt")
```

### LangChain / LangGraph 集成

AgentCube 可以作为 LangChain 的 `@tool` 接入 ReAct Agent 工作流，代码执行环节自动在安全沙箱中运行，应用层无需关心基础设施细节。

### Dify 插件

`integrations/dify-plugin/` 提供了 Dify 平台的工具集成，Dify 用户可以直接使用 AgentCube 的沙箱能力。

---

## 快速上手

### 前置条件

- Kubernetes 集群 v1.24+
- Redis 或 ValKey 实例
- `sigs.k8s.io/agent-sandbox` v0.1.1 CRD 已安装

### Helm 安装

```bash
helm install agentcube manifests/charts/base \
  --namespace agentcube-system --create-namespace \
  --set redis.addr=<redis-host>:6379 \
  --set redis.password="<password>"
```

### 创建你的第一个 CodeInterpreter

```yaml
apiVersion: runtime.agentcube.volcano.sh/v1alpha1
kind: CodeInterpreter
metadata:
  name: my-interpreter
  namespace: default
spec:
  template:
    image: ghcr.io/volcano-sh/picod:latest
    resources:
      requests: { cpu: "500m", memory: "512Mi" }
      limits:   { cpu: "2",    memory: "2Gi"   }
  warmPoolSize: 2
  sessionTimeout: 15m
  maxSessionDuration: 8h
```

### 发起调用

```bash
curl -X POST \
  http://<router-host>/v1/namespaces/default/code-interpreters/my-interpreter/invocations/api/execute \
  -H "Content-Type: application/json" \
  -d '{"command": ["python3", "-c", "print(1+1)"]}'
```

响应中的 `x-agentcube-session-id` 头即为你的会话 ID，后续请求携带它即可复用同一沙箱环境。

---

## 致谢

感谢所有为 AgentCube v0.1.0 做出贡献的开发者：

[@YaoZengzeng](https://github.com/YaoZengzeng)、[@acsoto](https://github.com/acsoto)、[@hzxuzhonghu](https://github.com/hzxuzhonghu)、[@Sagar-6203620715](https://github.com/Sagar-6203620715)、[@mahil-2040](https://github.com/mahil-2040)、[@t2wang](https://github.com/t2wang)、[@FAUST-BENCHOU](https://github.com/FAUST-BENCHOU)、[@tjucoder](https://github.com/tjucoder)、[@LaynePeng](https://github.com/LaynePeng)、[@yashisrani](https://github.com/yashisrani)、[@katara-Jayprakash](https://github.com/katara-Jayprakash)、[@LiZhenCheng9527](https://github.com/LiZhenCheng9527)、[@Tweakzx](https://github.com/Tweakzx)、[@warjiang](https://github.com/warjiang)、[@LeslieKuo](https://github.com/LeslieKuo)、[@MahaoAlex](https://github.com/MahaoAlex)、[@VanderChen](https://github.com/VanderChen)、[@kevin-wangzefeng](https://github.com/kevin-wangzefeng)、[@ifelseend](https://github.com/ifelseend)、[@cairon-ab](https://github.com/cairon-ab)、[@RushabhMehta2005](https://github.com/RushabhMehta2005)、[@Sanchit2662](https://github.com/Sanchit2662)、[@qizha](https://github.com/qizha)、[@ssfffss](https://github.com/ssfffss)、[@wjf295004046](https://github.com/wjf295004046)

---

## 相关链接

- **GitHub 仓库**：https://github.com/volcano-sh/agentcube
- **Volcano 社区**：https://volcano.sh
- **设计文档**：https://github.com/volcano-sh/agentcube/tree/main/docs/design
- **Python SDK**：https://github.com/volcano-sh/agentcube/tree/main/sdk-python

---

AgentCube 是一个开源项目，欢迎社区开发者参与。无论是提交 Issue、贡献 PR，还是在你的项目中试用 AgentCube，都是对项目的支持。

GitHub 地址：https://github.com/volcano-sh/agentcube
