# FinceptTerminal AI Agent 系统架构分析

> 生成日期：2026-05-08
> 框架：Agno (Python LLM Agent Framework)
> LLM 提供商：40+
> 内置工具：100+ / MCP 工具：237+

---

## 一、系统总览

FinceptTerminal 的 AI Agent 系统是一个全栈多代理框架，C++ 端负责 UI、编排、安全和 DataHub 集成，Python 端负责 LLM 调用、工具执行、Agent 协调和高级模块。两端通过 JSON over QProcess 通信。

### 1.1 整体架构

```
┌────────────────────────────────────────────────────────────────────┐
│                         C++ 端 (Qt6)                               │
│                                                                    │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
│  │ AgentConfig  │  │ ChatMode     │  │ Workflow Agent Nodes     │  │
│  │ Screen (8面板)│  │ Screen       │  │ (AI Agent / Tool Picker) │  │
│  └──────┬───────┘  └──────┬───────┘  └───────────┬──────────────┘  │
│         │                 │                       │                 │
│  ┌──────▼─────────────────▼───────────────────────▼──────────────┐ │
│  │                    AgentService (Singleton)                    │ │
│  │  run_agent / run_agent_streaming / run_team / run_workflow    │ │
│  │  route_query / create_plan / execute_plan / paper_trading    │ │
│  │  store_memory / recall_memories / search_knowledge           │ │
│  └──────┬──────────────────────────┬────────────────────────────┘ │
│         │                          │                               │
│  ┌──────▼──────┐  ┌───────────────▼───────────────────────────┐  │
│  │  LlmService │  │        TerminalToolBridge (HTTP)           │  │
│  │ (多提供商)   │  │  POST /tool → McpProvider → 本地工具执行    │  │
│  └─────────────┘  │  GET /tools → 列出可用 MCP 工具             │  │
│                    └───────────────────────────────────────────┘  │
│         │ PythonRunner / QProcess(stdin)                          │
└─────────┼────────────────────────────────────────────────────────┘
          │ JSON over stdout / stdin
┌─────────▼────────────────────────────────────────────────────────┐
│                      Python 端 (Agno Framework)                   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                   finagent_core/main.py                     │ │
│  │              统一入口：分发所有 Agent 操作                    │ │
│  └──────┬──────────────────────────────────────────────────────┘ │
│         │                                                         │
│  ┌──────▼──────┐  ┌────────────┐  ┌───────────────────────────┐ │
│  │  CoreAgent   │  │ SuperAgent │  │ Agent Factory / Loader    │ │
│  │ (单 Agent 实例)│  │ (路由分发)  │  │ (动态发现 / 创建 Agent)   │ │
│  └──────┬──────┘  └─────┬──────┘  └───────────────────────────┘ │
│         │               │                                         │
│  ┌──────▼───────────────▼─────────────────────────────────────┐  │
│  │                  高级模块系统 (可插拔)                       │  │
│  ├──────────┬───────────┬──────────┬──────────┬───────────────┤  │
│  │ Memory   │  Team     │ Workflow │ Knowledge│ Guardrails    │  │
│  │ Reasoning│ Evaluation│ Tracing  │Compression│ Hooks        │  │
│  └──────────┴───────────┴──────────┴──────────┴───────────────┘  │
│                                                                  │
│  ┌──────────────────┐  ┌──────────────────────────────────────┐ │
│  │ Models Registry  │  │ Tools Registry (100+ Tools)          │ │
│  │ (40+ LLM 提供商)  │  │ Finance / Search / Web / Dev / DB   │ │
│  └──────────────────┘  └──────────────────────────────────────┘ │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │              专用 Agent (30+)                                 ││
│  │  Geopolitical (19) │ Hedge Fund (8) │ Investors (2) │ Econ  ││
│  └──────────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────────┘
```

### 1.2 Agent 执行模式

| 模式 | C++ 入口 | Python 分发 | 适用场景 |
|------|----------|-------------|----------|
| **单 Agent** | `run_agent()` | `core_agent.py` | 标准查询、分析 |
| **流式输出** | `run_agent_streaming()` | `core_agent.py` + stream | 实时对话、长回复 |
| **结构化输出** | `run_agent_structured()` | `core_agent.py` + Pydantic | 金融信号、分析报告 |
| **路由查询** | `route_query()` + `execute_routed_query()` | `super_agent.py` | 自动选择最佳 Agent |
| **多查询并行** | `execute_multi_query()` | 多 Agent 并行 | 多维度分析 |
| **团队协作** | `run_team()` | Team Module | 多角色协作 |
| **工作流** | `run_workflow()` | Workflow Module | 复杂多步任务 |
| **规划执行** | `create_plan()` + `execute_plan()` | Planner Module | 动态任务分解 |

---

## 二、C++ 端实现

### 2.1 AgentService — Agent 编排中心

**位置：** `src/services/agents/AgentService.h/cpp`

AgentService 是 C++ 端的 Agent 编排单例，管理所有 Agent 生命周期和执行。

#### 核心数据结构 (`AgentTypes.h`)

```cpp
struct AgentInfo {
    QString id, name, description, category;
    QStringList capabilities;
    QString provider, version;
};

struct AgentExecutionResult {
    bool success;
    QString response, error;
    double elapsed_ms;
    QString request_id;
};

struct RoutingResult {
    QString agent_id, intent;
    double confidence;
    QStringList matched_keywords;
};

struct TeamConfig {
    QString name;
    QVector<TeamMember> members;    // 多个 Agent 角色
    QString mode;                   // coordinate | route | collaborate
};

struct ExecutionPlan {
    QString plan_id;
    QVector<PlanStep> steps;
};
```

#### 核心接口

**基础执行：**

```cpp
// 标准 Agent 执行
void run_agent(const QString& agent_id, const QString& query,
               const QJsonObject& context, Callback cb);

// 流式执行（Token-by-Token）
void run_agent_streaming(const QString& agent_id, const QString& query,
                         StreamCallback on_token, Callback on_done);

// 结构化输出（Pydantic 模型验证）
void run_agent_structured(const QString& agent_id, const QString& query,
                          const QString& output_model, Callback cb);
```

**智能路由：**

```cpp
// 自动选择最佳 Agent
void route_query(const QString& query, RoutingCallback cb);

// 路由后执行
void execute_routed_query(const RoutingResult& route, const QString& query,
                          Callback cb);

// 多 Agent 并行查询
void execute_multi_query(const QStringList& queries, MultiCallback cb);
```

**团队与工作流：**

```cpp
// 团队协作执行
void run_team(const TeamConfig& team, const QString& task, Callback cb);

// 工作流执行
void run_workflow(const QString& workflow_id, const QJsonObject& params, Callback cb);

// 动态规划
void create_plan(const QString& goal, PlanCallback cb);
void execute_plan(const ExecutionPlan& plan, Callback cb);
```

**金融专用工作流：**

```cpp
void run_stock_analysis(const QString& symbol, Callback cb);
void run_portfolio_rebalancing(const QString& portfolio_id, Callback cb);
void run_risk_assessment(const QString& portfolio_id, Callback cb);
void run_portfolio_analysis(const QString& portfolio_id, Callback cb);
```

**记忆与知识：**

```cpp
void store_memory(const QString& key, const QString& content, Callback cb);
void recall_memories(const QString& query, int limit, Callback cb);
void search_knowledge(const QString& query, Callback cb);
void save_memory_repo(const QString& content, Callback cb);
void search_memories_repo(const QString& query, int limit, Callback cb);
```

**模拟交易：**

```cpp
void paper_execute_trade(const QString& symbol, const QString& side,
                         double quantity, Callback cb);
void paper_get_portfolio(Callback cb);
void paper_get_positions(Callback cb);
```

#### Python 集成方式

AgentService 通过两种方式调用 Python：

```
小载荷 (<8KB)：
  PythonRunner::run("agents/finagent_core/main.py", args, callback)
  参数通过命令行传递

大载荷 (>=8KB)：
  QProcess → python main.py → 通过 stdin 写入 JSON payload
  避免 Windows 命令行长度限制
```

#### DataHub 集成

AgentService 实现了 **push-only Producer**，发布以下 Topic：

| Topic | 数据 |
|-------|------|
| `agent:output:<run_id>` | Agent 执行结果 |
| `agent:stream:<run_id>` | 流式 Token 输出 |
| `agent:status:<run_id>` | 执行状态更新 |
| `agent:routing:<run_id>` | 路由决策结果 |
| `agent:error:<run_id>` | 错误信息 |

#### 缓存策略

| 缓存对象 | TTL |
|----------|-----|
| Agent 列表 | 5 分钟 |
| 工具列表 | 5 分钟 |
| 模型列表 | 10 分钟 |
| 响应缓存 | 2 分钟 |

### 2.2 LlmService — LLM 统一接口

**位置：** `src/ai_chat/LlmService.h/cpp`

多提供商 LLM 客户端，支持：

| 提供商 | 模型 |
|--------|------|
| OpenAI | GPT-4o, GPT-4, o1, o3, o4 |
| Anthropic | Claude Sonnet/Opus/Haiku |
| Google | Gemini Pro/Ultra |
| DeepSeek | DeepSeek-V3, DeepSeek-R1 |
| Groq | Llama, Mixtral (高速推理) |
| 本地 | Ollama, LM Studio, vLLM |

**核心功能：**
- 流式响应（Server-Sent Events）
- 工具调用（Function Calling）
- Profile 解析（不同场景使用不同模型配置）
- 模型列表获取

### 2.3 TerminalToolBridge — 工具桥接

**位置：** `src/screens/chat_mode/TerminalToolBridge.h/cpp`

这是连接 Python Agent 和 C++ 内部 MCP 工具的关键桥梁。

```
Python Agent (finagent_core)
    │
    │ HTTP POST /tool
    │ {"tool": "get_quote", "args": {"symbol": "AAPL"}}
    │
    ▼
TerminalToolBridge (QHttpServer)
    │
    │ 验证 UUID Token
    │ 过滤 UI-only 工具
    │
    ▼
McpProvider::call_tool_async()
    │
    ▼
本地工具执行 (237+ MCP 工具)
    │
    ▼
HTTP Response → Python Agent
```

**安全机制：**
- UUID Token 认证（每进程唯一）
- 工具过滤（排除 UI-only 类别：navigation, system, settings）
- 破坏性操作阻止（Agent 模式下禁止未授权的写操作）
- `is_call_in_progress()` 标记区分 Agent 调用与用户直接调用

### 2.4 Agent 配置界面 (`src/screens/agent_config/`)

**AgentConfigScreen** — 8 面板导航界面：

| 面板 | 布局 | 功能 |
|------|------|------|
| **AgentsViewPanel** | 三栏：列表 / 配置 / 测试 | 浏览 Agent、编辑配置、实时测试 |
| **CreateAgentPanel** | 三栏：已保存 / 表单 / 测试 | 创建新 Agent：基本信息、LLM、指令、工具、功能开关 |
| **TeamsViewPanel** | 列表 + 编辑器 | 创建多 Agent 团队，分配角色和模型 |
| **WorkflowsViewPanel** | 列表 + 执行面板 | 运行预定义金融工作流 |
| **PlannerViewPanel** | 规划器 + 步骤监控 | 创建和执行动态计划 |
| **ToolsViewPanel** | 分类工具浏览 | 浏览和搜索可用 MCP 工具 |
| **AgentChatPanel** | 对话界面 | 与配置好的 Agent 聊天，支持组合上下文 |
| **SystemViewPanel** | 系统信息 | 查看 Agent 系统能力、管理 LLM Profile |

**CreateAgentPanel 可配置项：**

```
┌─ 基本信息 ──────────────────────────────────┐
│  名称、描述、分类                              │
├─ LLM 配置 ──────────────────────────────────┤
│  Provider 选择、Model 选择                    │
│  Temperature、Max Tokens                      │
├─ 指令编辑器 ─────────────────────────────────┤
│  Role + Goal + Instructions 组合              │
├─ 工具分配 ───────────────────────────────────┤
│  搜索选择工具、MCP 服务器工具                   │
├─ 功能开关 ───────────────────────────────────┤
│  ☐ Reasoning   ☐ Memory     ☐ Knowledge      │
│  ☐ Guardrails  ☐ Tracing    ☐ Compression    │
│  ☐ Evaluation  ☐ Hooks                        │
├─ 知识库配置 ─────────────────────────────────┤
│  Embedder 选择、Vector DB 选择                 │
├─ 记忆设置 ───────────────────────────────────┤
│  后端选择、存储策略                             │
└──────────────────────────────────────────────┘
```

### 2.5 Chat 界面 (`src/screens/chat_mode/`)

**ChatModeScreen** — 全屏 AI 聊天界面：

| 组件 | 功能 |
|------|------|
| `ChatSessionPanel` | 会话管理（创建/切换/删除） |
| `ChatMessagePanel` | 消息显示（气泡、打字指示器、代码高亮） |
| `ChatAgentPanel` | Agent 选择、参数调整、执行控制 |
| `TerminalToolBridge` | 工具桥接（生命周期与 Chat 绑定） |

### 2.6 工作流 Agent 节点

**位置：** `src/services/workflow/nodes/AgentNodes.cpp`

**AI Agent Node：**
```cpp
// 在工作流中执行已配置的 Agent
// 参数：agent_id, llm_profile_id, extra_instructions
// 加载 AgentConfig → 解析 LLM Profile → 执行 → 回调
```

**Tool Picker Node：**
```cpp
// LLM 智能选择工具
// 输入：自然语言描述
// LLM 分析 → 返回 {tool_name, args}
// 供后续 MCP Tool 节点使用
```

### 2.7 Agent 持久化

**AgentConfigRepository** — SQLite 存储：

| 表 | 内容 |
|----|------|
| agent_configs | Agent 配置（ID/名称/模型/指令/工具...） |
| profile_assignments | LLM Profile 分配 |
| tool_associations | Agent-Tool 关联 |
| active_states | 活跃状态标记 |

---

## 三、Python 端实现

### 3.1 FinAgent Core — Agent 核心框架

**位置：** `scripts/agents/finagent_core/`

#### 3.1.1 main.py — 统一入口

所有 Agent 操作的 Python 入口，由 C++ 通过 PythonRunner 调用。

```python
# 调用方式
python main.py --action <action> --payload '<json>'

# 支持的 action
run_agent           # 标准 Agent 执行
run_agent_streaming # 流式执行
run_agent_structured # 结构化输出
run_team            # 团队执行
run_workflow        # 工作流执行
create_plan         # 创建计划
execute_plan        # 执行计划
route_query         # 查询路由
list_agents         # 列出 Agent
get_agent_info      # Agent 详情
store_memory        # 存储记忆
recall_memories     # 回忆记忆
search_knowledge    # 搜索知识
paper_execute_trade # 模拟交易
paper_get_portfolio # 获取组合
paper_get_positions # 获取持仓
```

**特性：**
- 40+ LLM 提供商自动解析
- 流式输出支持
- 5 秒发现超时保护
- 从 C++ 接收活跃 LLM 模型配置

#### 3.1.2 CoreAgent — 核心 Agent 实现

```python
class CoreAgent:
    """
    单可配置 Agno Agent，使用 PersonaRegistry 实现用户/Agent 隔离。
    支持 100+ 工具，集成所有可选模块。
    """

    # 核心方法
    def run(query, context) → AgentResponse
    def run_streaming(query, context) → Iterator[Token]
    def run_structured(query, output_model) → PydanticModel
```

**PersonaRegistry：**
- LRU 缓存 PersonaRuntime 实例
- 按 user_id + agent_id 隔离状态
- 避免重复创建 Agent 实例

**CoreAgentBuilder（Builder 模式）：**
```python
agent = (CoreAgentBuilder()
    .with_model("openai", "gpt-4o")
    .with_instructions("You are a financial analyst...")
    .with_tools(["yfinance", "duckduckgo"])
    .with_memory(backend="sqlite")
    .with_guardrails(enabled=True)
    .with_reasoning(min_steps=3, max_steps=10)
    .build())
```

#### 3.1.3 AgentFactory & AgentLoader

**AgentLoader — 动态 Agent 发现：**
```python
class AgentLoader:
    """
    从 JSON 配置和 Python 文件中发现 Agent。
    优化：仅扫描已知目录，不做递归 rglob。
    缓存发现结果，TTL 过期后重新扫描。
    超时保护：5 秒上限。
    """
    def discover_agents() → List[AgentConfig]
    def load_agent(agent_id) → Agent
```

**AgentFactory — Agent 创建：**
```python
class AgentFactory:
    """
    从配置创建 Agno Agent 实例。
    处理：模型解析、工具实例化、模块装配。
    """
    def create(config: AgentConfig) → Agent
```

#### 3.1.4 SuperAgent — 智能路由

```python
class SuperAgent:
    """
    分诊路由：将查询分发到最合适的 Agent。
    """

    # 路由模式
    def route(query) → RoutingResult

    # LLM 路由（主要模式）
    # LLM 分析查询意图 → 选择最佳 Agent → 返回路由结果

    # 关键词路由（降级模式，无 API Key 时）
    # 关键词匹配 → 选择 Agent → 返回路由结果
```

**支持的路由意图：**

| 意图 | 目标 Agent 类型 |
|------|----------------|
| Trading | 交易类 Agent |
| Portfolio | 组合管理 Agent |
| Analysis | 分析类 Agent |
| Risk | 风险评估 Agent |
| News | 新闻分析 Agent |
| Geopolitics | 地缘政治 Agent |
| Economics | 经济分析 Agent |
| Research | 研究类 Agent |
| General | 通用 Agent |

**特性：** 置信度评分、多 Agent 路由（最多 3 个）、响应聚合、降级处理。

### 3.2 Agent 配置系统

#### JSON 配置格式

```json
{
    "id": "warren_buffett",
    "name": "Warren Buffett Agent",
    "description": "Value investing analysis based on Buffett's philosophy",
    "category": "investor",
    "model": {
        "provider": "openai",
        "model_id": "gpt-4o",
        "temperature": 0.7,
        "max_tokens": 4096
    },
    "instructions": "You are Warren Buffett...",
    "tools": ["yfinance", "sec_filings", "financial_datasets"],
    "capabilities": ["stock_analysis", "value_investing"]
}
```

**动态特性：**
- 指令可由 `role` + `goal` + `instructions` 组合构建
- 工具可从顶层提升
- LLM 配置可提升为 model

### 3.3 LLM 提供商 (40+)

**位置：** `scripts/agents/finagent_core/models_registry.py`

| 家族 | 提供商 |
|------|--------|
| **OpenAI 系列** | OpenAI, Azure OpenAI, DeepSeek, xAI (Grok), DashScope |
| **Anthropic** | Claude 3.5/4 (Sonnet/Opus/Haiku) |
| **Google** | Gemini Pro/Ultra, VertexAI |
| **Meta** | Llama 3/4 系列 |
| **本地部署** | Ollama, LM Studio, LlamaCpp, vLLM |
| **高速推理** | Groq, Cerebras |
| **其他** | Mistral, Cohere, Together, Fireworks, AI21, Anyscale |

**特性：**
- 自动 API Key 解析（环境变量或运行时注入）
- 提供商特定参数映射（如 `max_tokens` vs `max_output_tokens`）
- 自定义 `base_url` 支持 OpenAI 兼容端点
- 懒加载提升性能

### 3.4 工具注册表 (100+)

**位置：** `scripts/agents/finagent_core/tools_registry.py`

| 类别 | 工具 |
|------|------|
| **金融** | yfinance, financial_datasets, openbb |
| **搜索** | duckduckgo, tavily, exa, serpapi |
| **网页** | 网站工具、爬虫、内容提取 |
| **开发** | python, shell, file, docker |
| **DevOps** | github, gitlab, jira, notion |
| **数据库** | sql, postgres, duckdb |
| **通讯** | email, slack, discord |
| **AI/ML** | openai, dalle, replicate |
| **研究** | arxiv, pubmed, wikipedia |

**特性：**
- 懒加载（仅在使用时初始化）
- API Key 注入
- 无状态工具缓存

### 3.5 高级模块系统

所有模块可插拔，通过 Agent 配置启用/禁用。

#### 3.5.1 Memory — 持久记忆

```python
AgenticMemoryModule:
    - 事实存储（带重要性评分）
    - 按查询/类型/数量回忆
    - 用户/Agent 隔离
    - 后端：SQLite / PostgreSQL
```

#### 3.5.2 Team — 多 Agent 协作

```python
TeamModule:
    - Agent 角色定义（role + tools）
    - Coordinator 模型管理团队
    - 并行执行支持
    - 团队级响应聚合
```

**团队协调模式：**

| 模式 | 说明 |
|------|------|
| `coordinate` | Coordinator 分发子任务，汇总结果 |
| `route` | 根据意图路由到对应 Agent |
| `collaborate` | Agent 之间可以互相通信协作 |

#### 3.5.3 Workflow — 工作流编排

```python
WorkflowModule:
    组件：
    - Parallel  → 并行执行多个 Agent
    - Condition → 条件分支
    - Loop      → 循环（带终止条件）
    - Router    → 路由到子工作流

    金融模板：
    - Stock Analysis Pipeline
    - Portfolio Rebalancing
    - Risk Assessment
```

#### 3.5.4 Knowledge — RAG 知识库

```python
KnowledgeModule:
    - 向量数据库：LanceDB / PgVector / FAISS
    - 嵌入器：OpenAI / 本地模型
    - 知识搜索（相关性评分）
    - 动态知识库更新
```

#### 3.5.5 Reasoning — 推理增强

```python
ReasoningModule:
    - Chain-of-Thought 推理
    - 逐步分析
    - 支持推理模型（o1, o3, o4）
    - 可配置最小/最大推理步数
```

#### 3.5.6 Guardrails — 安全护栏

```python
GuardrailsModule:
    - Financial PII：检测并遮蔽金融个人身份信息
    - Prompt Injection：防止提示注入攻击
    - Trading Compliance：确保交易规则合规
    - Output Validation：验证结构化输出

    功能：输入/输出检查、自定义规则、警告生成
```

#### 3.5.7 Evaluation — Agent 评估

```python
EvaluationModule:
    评估器：
    - Accuracy    → 预测准确度
    - Performance → 速度、成本、可靠性
    - Reliability → 时间一致性
    - Agent Judge → LLM 评审

    功能：预测 vs 实际追踪、响应质量评估
```

#### 3.5.8 其他模块

| 模块 | 功能 |
|------|------|
| **Tracing** | Span 追踪、性能指标、审计日志、错误追踪 |
| **Compression** | Token 优化（金融数据压缩、摘要、关键信息提取） |
| **Hooks** | 前置/后置处理（输入验证、输出格式化、限流、成本追踪） |

#### 3.5.9 结构化输出模型

```python
# Pydantic 模型，用于 run_agent_structured()
class TradeSignal(BaseModel):
    symbol: str
    action: Literal["buy", "sell", "hold"]
    confidence: float
    reasoning: str
    target_price: Optional[float]

class StockAnalysis(BaseModel):
    symbol: str
    rating: Literal["strong_buy", "buy", "hold", "sell", "strong_sell"]
    target_price: float
    key_factors: List[str]
    risks: List[str]

class PortfolioAnalysis(BaseModel): ...
class RiskAssessment(BaseModel): ...
class MarketAnalysis(BaseModel): ...
class ResearchReport(BaseModel): ...
```

### 3.6 模拟交易桥接

**位置：** `scripts/agents/finagent_core/paper_trading_bridge.py`

```python
PaperTradingBridge:
    # 组合管理
    create_portfolio(name) → portfolio_id
    get_portfolio(portfolio_id) → Portfolio
    reset_portfolio(portfolio_id)
    delete_portfolio(portfolio_id)

    # 订单操作
    place_order(symbol, side, quantity, order_type) → Order
    cancel_order(order_id)
    fill_order(order_id, fill_price)

    # 持仓追踪
    get_positions(portfolio_id) → List[Position]
    get_trade_history(portfolio_id) → List[Trade]

    # 统计
    get_statistics(portfolio_id) → Statistics
```

**订单类型：** Market / Limit / Stop

**桥接模式：**
- Host Bridge：通过 C++ IPC 执行真实模拟交易
- Local Simulation：本地模拟模式（测试用）

---

## 四、专用 Agent 清单

### 4.1 地缘政治 Agent（19 个）

#### 大棋局框架 (Grand Chessboard)

| Agent ID | 分析维度 |
|----------|----------|
| `american_primacy` | 美国全球领导力战略 |
| `eurasian_balkans` | 中亚地缘政治博弈 |
| `heartland` | 麦金德心脏地带控制论 |
| `pivots` | 关键地缘政治支点国家 |
| `players` | 主要全球大国角色分析 |

#### 地理囚徒框架 (Prisoners of Geography)

| Agent ID | 地区 |
|----------|------|
| `russia_geography` | 俄罗斯：出海口困境、战略纵深 |
| `china_geography` | 中国：岛链封锁、一带一路 |
| `usa_geography` | 美国：两洋优势、资源独立 |
| `europe_geography` | 欧洲：碎片化、能源依赖 |
| `middle_east_geography` | 中东：石油、宗教冲突、大国博弈 |
| `africa_geography` | 非洲：资源诅咒、发展约束 |
| `india_pakistan_geography` | 南亚：印巴对抗、喜马拉雅屏障 |
| `japan_korea_geography` | 东亚：岛国困境、朝鲜半岛 |
| `latin_america_geography` | 拉美：资源依赖、地缘边缘化 |
| `arctic_geography` | 北极：航道争夺、资源竞争 |

#### 世界秩序框架 (World Order)

| Agent ID | 秩序观 |
|----------|--------|
| `american_order` | 自由国际秩序 |
| `chinese_order` | 儒家和谐世界观 |
| `european_order` | 权力平衡体系 |
| `islamic_order` | 伊斯兰治理原则 |

### 4.2 对冲基金 Agent（8 个）

| Agent | 策略风格 | AUM | 内部角色 |
|-------|----------|-----|----------|
| **Bridgewater Associates** | 全球宏观、风险平价 | $124B | 宏观分析师 + 风险平价策略师 |
| **Citadel** | 多策略、量化 | $62B | 量化研究员 + 策略执行 |
| **Renaissance Technologies** | 量化模型 | $55B | Portfolio Manager + Quant Researcher + Risk Quant + Signal Scientist |
| **Two Sigma** | AI/ML、系统化交易 | $60B | ML 工程师 + 数据科学家 |
| **D.E. Shaw** | 计算金融 | $60B | 量化分析师 + 算法交易员 |
| **Elliott Management** | 激进主义、困境债务 | $56B | 激进投资分析师 + 困境债务专家 |
| **Pershing Square** | 激进价值投资 | $16B | 价值分析师 + 激进策略师 |
| **ARQ Capital** | 因子投资、量化 | $90B | 因子分析师 + 风险经理 |

**Renaissance Technologies 示例内部结构：**

```
Renaissance Technologies Agent
├── Portfolio Manager  → 策略分配
├── Quant Researcher   → 统计建模
├── Risk Quant         → 风险管理
└── Signal Scientist   → 信号生成
```

每个角色有专门的工具集和合规集成。

### 4.3 传奇投资者 Agent（2 个）

| Agent | 投资哲学 |
|-------|----------|
| **Warren Buffett** | 价值投资：护城河、能力圈、长期持有、恐惧贪婪论 |
| **Benjamin Graham** | 深度价值：安全边际、市场先生、内在价值计算 |

### 4.4 经济分析 Agent（1 个）

| Agent | 功能 |
|-------|------|
| **Economic Analysis** | 宏观经济分析、政策解读、经济指标预测 |

---

## 五、交易系统

### 5.1 Agno Trading (`scripts/agno_trading/`)

| 组件 | 功能 |
|------|------|
| `BaseAgent` | 核心交易 Agent，集成实时市场数据 |
| `CompetitionRuntime` | 多模型交易竞赛运行时 |
| 特性 | 实时数据、模拟执行、组合管理、绩效追踪、排行榜 |

### 5.2 Alpha Arena (`scripts/alpha_arena/`)

| 组件 | 功能 |
|------|------|
| LLM Call Subprocess | 独立 LLM 调用进程，带成本追踪 |
| Wire Protocol | 安全 API Key 交换（IPC） |
| 特性 | 多提供商支持、成本估算、延迟追踪 |

### 5.3 AI Quant Lab (`scripts/ai_quant_lab/`)

| 模块 | 功能 |
|------|------|
| Qlib Core | 模型训练、回测、组合优化 |
| RL Trading | 强化学习交易 Agent |
| GS Quant | 风险指标、组合分析、Greeks |
| Functime | 时间序列预测、异常检测 |
| RD-Agent | 自主因子/模型研究 |
| Deep Agent | LangGraph 多 Agent 系统 |

---

## 六、端到端执行流

### 6.1 标准 Agent 执行

```
用户输入 "分析 AAPL 的投资价值"
    │
    ▼
AgentConfigScreen / ChatModeScreen
    │
    ▼
AgentService::run_agent("stock_analyst", "分析 AAPL 的投资价值", {})
    │
    ├─ 1. 构建 JSON payload
    │      {"action": "run_agent", "agent_id": "stock_analyst",
    │       "query": "分析 AAPL 的投资价值", "context": {...}}
    │
    ├─ 2. PythonRunner::run("agents/finagent_core/main.py", ...)
    │
    ▼
Python: finagent_core/main.py
    │
    ├─ 3. 解析 payload → action = "run_agent"
    │
    ├─ 4. AgentFactory.create(config) → Agno Agent
    │      ├─ 加载模型配置 (GPT-4o)
    │      ├─ 加载工具 (yfinance, sec_filings, ...)
    │      ├─ 装配模块 (memory, guardrails, ...)
    │      └─ 构建指令 (system prompt)
    │
    ├─ 5. Agent.run(query, context)
    │      ├─ LLM 分析查询
    │      ├─ 工具调用：yfinance("AAPL") → 获取财务数据
    │      ├─ 工具调用：sec_filings("AAPL") → 获取 SEC 文件
    │      ├─ LLM 综合分析
    │      └─ 生成结构化报告
    │
    ├─ 6. 输出 JSON 到 stdout
    │
    ▼
C++: PythonRunner callback
    │
    ├─ 7. extract_json() 解析结果
    │
    ├─ 8. DataHub::publish("agent:output:<run_id>", result)
    │
    ▼
UI: AgentChatPanel / AgentsViewPanel
    └─ 显示分析结果
```

### 6.2 路由查询执行

```
用户输入 "中东局势对油价的影响"
    │
    ▼
AgentService::route_query("中东局势对油价的影响")
    │
    ▼
Python: SuperAgent.route(query)
    │
    ├─ LLM 分析意图 → "geopolitics"
    ├─ 选择 Agent: "middle_east_geography"
    ├─ 置信度: 0.92
    │
    ▼
AgentService::execute_routed_query(routing_result, query)
    │
    ▼
AgentService::run_agent("middle_east_geography", query, ...)
    └─ [标准执行流程]
```

### 6.3 带工具桥接的执行

```
Python Agent 需要获取实时行情
    │
    ▼
Agent 调用工具 → HTTP POST localhost:<port>/tool
    {"tool": "get_quote", "args": {"symbol": "AAPL"},
     "token": "uuid-token"}
    │
    ▼
TerminalToolBridge (C++)
    │
    ├─ 验证 UUID Token
    ├─ 检查工具不在 UI-only 列表
    ├─ McpProvider::call_tool_async("get_quote", {"symbol": "AAPL"})
    │      └─ 调用 MarketDataService
    │              └─ PythonWorker → yfinance
    │
    ▼
HTTP Response → Python Agent
    {"ok": true, "result": {"price": 150.25, "change": 2.5}}
```

---

## 七、安全体系

### 7.1 多层安全

```
┌─ LLM 层 ────────────────────────────┐
│  Guardrails Module                   │
│  ├─ Financial PII 检测/遮蔽          │
│  ├─ Prompt Injection 防护            │
│  ├─ Trading Compliance 检查          │
│  └─ Output Validation                │
├─ 工具层 ────────────────────────────┤
│  TerminalToolBridge                  │
│  ├─ UUID Token 认证                  │
│  ├─ 工具过滤（排除 UI-only）         │
│  ├─ 破坏性操作阻止                   │
│  └─ Agent 调用标记                    │
├─ 数据层 ────────────────────────────┤
│  API Key 管理                        │
│  ├─ SecureStorage (AES-256-GCM)     │
│  ├─ 环境变量白名单注入               │
│  └─ 不透传 shell 环境敏感信息        │
└─ 评估层 ────────────────────────────┘
   Evaluation Module
   ├─ 预测 vs 实际追踪
   └─ 响应质量评估
```

### 7.2 工具访问控制

| 调用来源 | 权限 |
|----------|------|
| 用户直接调用 | 全部工具（含破坏性操作） |
| Agent 调用 | 仅非破坏性工具 + 白名单 |
| Agent 需破坏性操作 | 需显式授权（Capability Token） |

---

## 八、设计模式

| 模式 | 应用 |
|------|------|
| **Singleton** | AgentService, LlmService, McpProvider |
| **Builder** | CoreAgentBuilder (Fluent API) |
| **Factory** | AgentFactory, ChartFactory |
| **Registry** | ModelsRegistry, ToolsRegistry, PersonaRegistry |
| **Bridge** | TerminalToolBridge (C++ ↔ Python) |
| **Strategy** | 多 LLM 提供商、多工具后端 |
| **Observer** | Qt Signal/Slot 异步结果传递 |
| **LRU Cache** | PersonaRegistry (Agent 实例)、工具缓存 |
| **Module/Plugin** | 可插拔高级模块系统 |
| **Triage/Router** | SuperAgent 查询路由 |

---

## 九、统计总结

| 维度 | 数量 |
|------|------|
| LLM 提供商 | 40+ |
| 可用工具 (Python) | 100+ |
| MCP 工具 (C++) | 237+ |
| 专用 Agent | 30+ |
| 高级模块 | 10 (Memory/Team/Workflow/Knowledge/Reasoning/Guardrails/Evaluation/Tracing/Compression/Hooks) |
| 金融工作流模板 | 3 (Stock Analysis/Portfolio Rebalancing/Risk Assessment) |
| 结构化输出模型 | 6 (TradeSignal/StockAnalysis/PortfolioAnalysis/RiskAssessment/MarketAnalysis/ResearchReport) |
| C++ 界面面板 | 8 (AgentConfig) + 4 (ChatMode) |
| 路由意图类别 | 9 (Trading/Portfolio/Analysis/Risk/News/Geopolitics/Economics/Research/General) |
| Agent 执行模式 | 8 种 |
