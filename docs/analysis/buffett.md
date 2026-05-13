# 巴菲特分析（Warren Buffett Agent）实现分析

## 概述

巴菲特分析在 FinceptTerminal 中有两条独立的实现路径：

1. **AI Agent 路径** — `warren_buffett_agent`，基于 LLM + Agno 框架，用精心设计的 prompt 模拟巴菲特价值投资框架，结合实时金融数据 API 完成个股分析
2. **市场指标路径** — `get_buffett_indicator()`，基于 AKShare 的 `stock_buffett_index_lg()` 函数，获取巴菲特指数（总市值/GDP）用于评估整体市场估值水平

两条路径独立运作，分别服务于不同的分析场景。

---

## 一、AI Agent 路径：完整执行链路

### 1.1 配置层 — Agent 定义

**文件**: `fincept-qt/scripts/agents/TraderInvestorsAgent/configs/agent_definitions.json`（第 4–133 行）

Agent 以 JSON 配置定义，不包含任何硬编码的 Python 分析逻辑。核心是一个精心构造的 LLM system prompt（`config.instructions` 字段），指示 LLM 按照巴菲特框架进行分析。

```json
{
  "id": "warren_buffett_agent",
  "name": "Warren Buffett Investment Agent",
  "category": "TraderInvestorsAgent",
  "config": {
    "model": { "provider": "openai", "model_id": "gpt-4-turbo", "temperature": 0.3 },
    "instructions": "<约 2000 字的巴菲特框架 prompt>",
    "tools": ["yfinance", "financial_datasets", "duckduckgo", "tavily"],
    "memory": true,
    "agentic_memory": true,
    "scoring_weights": { ... },
    "thresholds": { ... },
    "output_schema": { "type": "InvestmentSignal", "fields": { ... } }
  }
}
```

**巴菲特分析框架（6 个维度）** 定义在 prompt 中：

| 维度 | 分析内容 | 评判标准 |
|------|---------|---------|
| 护城河 (MOAT) | 品牌/转换成本/网络效应/成本优势/规模/监管 | 必须命名一个具体的护城河来源 |
| 资本回报 (RETURNS ON CAPITAL) | ROE、ROIC | ROE ≥ 15%（10 年中 7 年），ROIC > WACC |
| 盈利可预测性 (EARNINGS PREDICTABILITY) | 正收益年份、营业利润率标准差 | ≥8/10 年正收益，利润率标准差 < 5 个百分点 |
| 资产负债表 (BALANCE SHEET) | D/E 比率、利息覆盖率 | D/E < 0.5，利息覆盖率 > 5x |
| 管理层评估 (MANAGEMENT) | 资本配置记录 | 回购定价合理性、M&A 质量、高 ROIC 再投资 |
| 估值 (VALUATION) | 所有者收益/市值，10% 折现 | 净利润 + D&A - 维护性资本支出 |

**评分权重**:

```json
{
  "moat_analysis": 0.30,          // 护城河权重最高
  "earnings_predictability": 0.25, // 盈利可预测性
  "financial_strength": 0.20,     // 财务实力
  "management_quality": 0.15,     // 管理层质量
  "valuation": 0.10               // 估值（权重最低，因为巴菲特更看重质而非价）
}
```

**硬性阈值**:

```json
{
  "roe_excellence": 0.15,            // ROE 优秀线 15%
  "roe_years_required": 7,           // 需要达标年数
  "debt_to_equity_max": 0.3,         // D/E 上限 0.3
  "positive_earnings_years": 8,      // 正收益年数要求
  "positive_fcf_years": 8,           // 正 FCF 年数要求
  "margin_minimum": 0.15,            // 利润率最低 15%
  "margin_stability_max": 0.05,      // 利润率波动上限 5 个百分点
  "revenue_growth_min": 0.05,        // 营收增长下限 5%
  "revenue_growth_max": 0.15,        // 营收增长上限 15%
  "current_ratio_min": 1.5,          // 流动比率最低 1.5
  "bullish_score": 8,                // 看好评级最低分
  "bearish_score": 4                  // 看淡评级最高分
}
```

**数据来源** (4 个工具):

| 工具 | 用途 | API Key |
|------|------|---------|
| `yfinance` | 股价、财务报表、现金流、10 年历史数据 | 免费 |
| `financial_datasets` | 标准化财务指标（ROE、ROIC、D/E 等） | `FINANCIAL_DATASETS_API_KEY` |
| `duckduckgo` | 竞争地位、管理层变动、资本配置事件新闻 | 免费 |
| `tavily` | 高级搜索，补充新闻和深度调研 | `TAVILY_API_KEY` |

**输出格式** (`InvestmentSignal`):

```json
{
  "signal": "bullish | neutral | bearish",
  "confidence": 0.0-1.0,
  "moat_score": 0-10,
  "earnings_predictability": 0-10,
  "financial_strength": 0-10,
  "management_quality": 0-10,
  "valuation_score": 0-10,
  "reasoning": "string"
}
```

### 1.2 发现层 — Agent 加载

**文件**: `fincept-qt/scripts/agents/finagent_core/agent_loader.py`

Agent 的发现和加载流程：

1. **`get_loader()`** 获取单例 `AgentLoader` 实例
2. 扫描 `KNOWN_AGENT_DIRS`（包含 `TraderInvestorsAgent`）目录
3. 在每个目录的 `configs/` 子目录中查找 `agent_definitions.json`
4. 将每个 agent 条目解析为 `AgentCard` 对象
5. `AgentCard.from_dict()` 将 JSON 转为内存中的数据结构
6. 如果顶层 JSON 缺少 `config.instructions`，会自动将 `instructions`、`role`、`goal` 等字段提升到 `config` 中

```python
# agent_loader.py 第 29-35 行
KNOWN_AGENT_DIRS = [
    "TraderInvestorsAgent",    # ← 巴菲特 agent 所在目录
    "hedgeFundAgents",
    "EconomicAgents",
    "GeopoliticsAgents",
    "finagent_core",
]
```

`AgentCard` 数据结构：

```python
@dataclass
class AgentCard:
    id: str                          # "warren_buffett_agent"
    name: str                        # "Warren Buffett Investment Agent"
    description: str                 # 描述文本
    category: str                    # "TraderInvestorsAgent"
    version: str                     # "2.0.0"
    provider: str                    # "local"
    capabilities: List[str]          # ["moat_analysis", "owner_earnings", ...]
    config: Dict[str, Any]           # 完整配置（instructions, tools, model, etc.）
    module_path: Optional[str]       # None（纯配置驱动，无自定义模块）
    class_name: Optional[str]        # None
```

### 1.3 执行层 — Agno Agent 构建

**核心调用链**:

```
C++ AgentService → Python main.py → dispatch_action("run") → CoreAgent.run()
    → PersonaRegistry.get_or_create()
        → PersonaRuntime.build()
            → build_agno_agent() → CoreAgent._create_agent()
                → ModelsRegistry.create_model()
                → ToolsRegistry.get_tools()
                → Agent(**kwargs)    # Agno Agent 实例
    → PersonaRuntime.run(query)
        → Agno Agent.run(input=query)  # LLM 执行
```

#### 1.3.1 C++ 入口 — AgentService

**文件**: `fincept-qt/src/services/agents/AgentService.h` + `AgentService.cpp`

`AgentService` 是 C++ 端的 singleton 服务，同时是 DataHub 的 `Producer`（push-only 模式）。

```cpp
// AgentService.h 第 39-40 行
QString run_agent(const QString& query, const QJsonObject& config);
QString run_agent_streaming(const QString& query, const QJsonObject& config);
```

执行流程：

1. UI 层（`AgentChatPanel` 或 `AgentsViewPanel`）调用 `run_agent_streaming(query, config)`
2. `AgentService` 构建 JSON payload：

```json
{
  "action": "run",
  "api_keys": { "openai": "...", "FINANCIAL_DATASETS_API_KEY": "..." },
  "params": { "query": "Analyze AAPL using Buffett framework", "session_id": "...", "stream": true },
  "config": { "agent_id": "warren_buffett_agent", "model": { "provider": "openai", "model_id": "gpt-4-turbo" } }
}
```

3. 通过 `PythonRunner`（轻量调用）或 `QProcess + stdin`（大 payload）发送给 Python
4. 返回 `request_id` 用于关联异步结果
5. 结果通过 Qt signals 和 DataHub topics 发布：
   - `agent:output:<request_id>` — 最终结果
   - `agent:stream:<request_id>` — 流式 token（50ms 合并）
   - `agent:status:<request_id>` — 思考/工具调用状态
   - `agent:error:*` — 错误信息

#### 1.3.2 Python 入口 — main.py

**文件**: `fincept-qt/scripts/agents/finagent_core/main.py`

`dispatch_action("run", ...)` 处理流程（第 261-332 行）：

```python
if action == "run":
    agent_id = config.get("agent_id", "")
    # 1. 加载 agent 卡配置
    card = get_loader().registry.get(agent_id)  # 获取 AgentCard
    config = {**card.config, **config}           # 合并用户覆盖

    # 2. 创建 CoreAgent
    agent = CoreAgent(api_keys=api_keys, user_id=params.get("user_id"))

    # 3. 设置模块（guardrails, tracing, evaluation 等）
    _setup_agent_modules(agent, config, params)

    # 4. 执行
    response = agent.run(query, full_config, session_id, stream)
    return {"success": True, "response": agent.get_response_content(response)}
```

#### 1.3.3 Persona 运行时

**文件**: `fincept-qt/scripts/agents/finagent_core/persona_runtime.py`

`PersonaRuntime` 是每个 (user_id, agent_id) 组合的有状态运行时实例。由 `PersonaRegistry`（LRU 缓存，默认 8 个槽位）管理。

`PersonaRuntime.build()` 组装流程（第 132-210 行）：

```python
@classmethod
def build(cls, user_id, agent_id, config, api_keys):
    # 1. 创建 persona 目录（存储 memory、storage、knowledge）
    resources.ensure_persona_dir(user_id, agent_id)

    # 2. 构建 Memory 后端（SQLite）
    memory_backend = _build_memory_backend(db_path=resources.memory_db(user_id, agent_id))

    # 3. 构建 Storage 后端（SQLite，session 持久化）
    storage_backend = _build_storage_backend(db_path=resources.sessions_db(user_id, agent_id))

    # 4. 构建 Knowledge 模块（RAG）
    knowledge_module = _build_knowledge_module(knowledge_cfg, user_id, agent_id, api_keys)

    # 5. 构建 Agno Agent（注入 memory/storage/knowledge）
    agno_agent = build_agno_agent(config, api_keys, memory_backend, storage_backend, knowledge_agent_kwargs)

    # 6. 构建 Agentic Memory（自动记忆模块）
    agentic_memory = AgenticMemoryModule.from_config({...})

    return cls(user_id, agent_id, agno_agent, memory_backend, storage_backend, knowledge_module, agentic_memory)
```

#### 1.3.4 Agno Agent 构建

**文件**: `fincept-qt/scripts/agents/finagent_core/core_agent.py`，`_create_agent()` 方法（第 538-748 行）

`build_agno_agent()` 调用 `CoreAgent._create_agent()` 进行最终组装，注入 11 个组件：

```
1. Model        → ModelsRegistry.create_model(provider="openai", model_id="gpt-4-turbo")
2. Identity     → name="Warren Buffett Investment Agent", description=...
3. Instructions → 巴菲特框架 prompt（约 2000 字）
4. Tools        → ToolsRegistry.get_tools(["yfinance", "financial_datasets", "duckduckgo", "tavily"])
5. Memory       → AgentMemory(SqliteMemoryDb) — 对话历史 + 用户记忆
6. Knowledge    → KnowledgeModule — "buffett_philosophy" 知识库
7. Reasoning    → 可选扩展思考
8. Output       → structured_outputs / response_model
9. Agentic Loop → tool_call_limit = min(max_iterations, 20)
10. Storage     → SqliteStorage — session 持久化
11. Session     → session_id 绑定
```

### 1.4 工具层 — 数据获取

**文件**: `fincept-qt/scripts/agents/finagent_core/registries/tools_registry.py`

`ToolsRegistry` 管理所有可用工具的注册和创建：

```python
# 巴菲特 agent 使用的 4 个工具
tool_names = ["yfinance", "financial_datasets", "duckduckgo", "tavily"]
tools = ToolsRegistry.get_tools(tool_names, api_keys=api_keys)
```

工具执行流程：

1. Agent 运行时，LLM 判断需要获取数据
2. LLM 生成 tool_call（如 `yfinance.get_stock_info("AAPL")`）
3. Agno 框架调用对应工具函数
4. 工具返回数据给 LLM
5. LLM 可能进行多轮 tool_call（最多 `tool_call_limit` 轮，默认 10 轮）
6. LLM 最终生成分析结果

巴菲特 agent 典型的 tool_call 序列：

```
Round 1: yfinance → 获取 AAPL 10 年财务数据（营收、净利润、FCF、ROE、D/E）
Round 2: financial_datasets → 获取 ROIC、毛利率、营业利润率等标准化指标
Round 3: duckduckgo → 搜索 AAPL 近 12 个月竞争地位、管理层变动新闻
Round 4: tavily → 深入搜索资本配置事件（M&A、回购、分红）
... 最终生成分析报告
```

### 1.5 记忆层 — 跨会话持久化

**文件**: `fincept-qt/scripts/agents/finagent_core/modules/memory_module.py`

巴菲特 agent 启用了两种记忆：

1. **会话记忆** (`memory: true`) — 对话历史，SQLite 存储
   - 路径: `<data_dir>/personas/<user_id>/warren_buffett_agent/memory.db`
   - 跨会话保持上下文，可回顾之前的分析

2. **智能记忆** (`agentic_memory: true`) — Agent 自动从交互中学习用户偏好
   - 路径: `<data_dir>/personas/<user_id>/warren_buffett_agent/agentic_memory.db`
   - 自动提取和存储关键事实、用户偏好

3. **知识库** (`knowledge_base: "buffett_philosophy"`) — 巴菲特投资哲学 RAG
   - 路径: `<data_dir>/personas/<user_id>/warren_buffett_agent/knowledge/`

### 1.6 输出层 — 结果回传

**Agno Agent 返回** → `CoreAgent.get_response_content()` 提取文本 → `main.py` 封装为 JSON → stdout 输出 → C++ `PythonRunner` 读取 → `AgentService` 发射 Qt signal → UI 更新

```json
{
  "success": true,
  "response": "## Signal\nbullish, confidence 0.75\n\n## Moat\nbrand + switching cost. Evidence: ...\n\n## Numbers\nROE 10y median: 42%, ROIC: 35%, ...\n\n## Management Verdict\n...\n\n## Verdict\nWonderful business at fair price\n\n## Risks To The Thesis\n..."
}
```

---

## 二、市场指标路径：巴菲特指数

### 2.1 数据脚本

**文件**: `fincept-qt/scripts/akshare_analysis.py`（第 245-258 行）

```python
def get_buffett_indicator(self) -> Dict[str, Any]:
    """Get Buffett market valuation indicator"""
    if not AKSHARE_FEATURES_AVAILABLE:
        return {"success": False, "error": "AKShare features module not available", "data": []}
    try:
        df = stock_buffett_index_lg()   # AKShare API 调用
        return self._validate_and_format_dataframe(df)
    except Exception as e:
        return {"success": False, "error": str(e), "data": []}
```

数据源：AKShare 的 `stock_buffett_index_lg()` 函数，返回 A 股市场的巴菲特指标数据（总市值 / GDP）。

### 2.2 JSON 响应格式

```json
{
  "success": true,
  "data": [
    {
      "date": "2024-05-13",
      "total_market_cap": 1234567890,
      "gdp_value": 456789012,
      "buffett_ratio": 2.70
    }
  ],
  "count": 1,
  "timestamp": 1715625600,
  "data_quality": "high",
  "columns": ["date", "total_market_cap", "gdp_value", "buffett_ratio"]
}
```

### 2.3 C++ 集成

**文件**: `fincept-qt/src/services/akshare/AkShareService.cpp`

```
AkShareScreen → AkShareService → PythonRunner → akshare_analysis.py → stock_buffett_index_lg()
                                      ↓
                               JSON stdout → extract_json() → QJsonArray → DataHub publish
                                      ↓
                              AkShareScreen UI 表格展示
```

- 缓存 TTL：2 分钟
- 错误处理：3 次重试 + 指数退避

### 2.4 备用数据源

**文件**: `fincept-qt/scripts/akshare_stocks_margin.py`（第 224-226 行）

```python
# 备用的巴菲特指数实现
@safe_call
def get_buffett_index():
    return ak.stock_buffett_index_lg()
```

---

## 三、完整架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         C++ Layer (Qt)                          │
│                                                                 │
│  ┌──────────────────┐    ┌─────────────────────────────────┐   │
│  │ AgentChatPanel   │───▶│ AgentService (singleton)        │   │
│  │ AgentsViewPanel  │    │ - run_agent_streaming()         │   │
│  └──────────────────┘    │ - topic: agent:output:<id>      │   │
│                          │ - topic: agent:stream:<id>       │   │
│  ┌──────────────────┐    └────────────┬────────────────────┘   │
│  │ AkShareScreen    │                 │ QProcess / stdin         │
│  │ (巴菲特指数表格)  │───▶ AkShareService ──▶ PythonRunner      │
│  └──────────────────┘                                      │    │
└────────────────────────────────────────────────────────────┼────┘
                                                             │
                        JSON payload over stdin/stdout        │
                                                             │
┌────────────────────────────────────────────────────────────┼────┐
│                      Python Layer                            ▼    │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ main.py — dispatch_action("run")                         │   │
│  │   1. AgentLoader.registry.get("warren_buffett_agent")    │   │
│  │   2. CoreAgent(api_keys=..., user_id=...)                │   │
│  │   3. PersonaRegistry.get_or_create(user, agent)          │   │
│  │      → PersonaRuntime.build()                            │   │
│  │         → build_agno_agent()                             │   │
│  │            → Agno Agent(instructions, model, tools, ...)  │   │
│  │   4. runtime.run(query) → Agno Agent.run()               │   │
│  │      LLM 执行巴菲特分析框架 prompt:                        │   │
│  │      ┌─────────────────────────────────────────┐          │   │
│  │      │ Round 1: yfinance → 10 年财务数据        │          │   │
│  │      │ Round 2: financial_datasets → ROE/ROIC   │          │   │
│  │      │ Round 3: duckduckgo → 竞争地位/管理层    │          │   │
│  │      │ Round 4: tavily → 资本配置事件            │          │   │
│  │      │ ... → 生成 InvestmentSignal               │          │   │
│  │      └─────────────────────────────────────────┘          │   │
│  │   5. get_response_content(response) → JSON stdout        │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ akshare_analysis.py — get_buffett_indicator()            │   │
│  │   → stock_buffett_index_lg() → DataFrame → JSON          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 四、关键设计决策

### 4.1 为什么用 Prompt 而非量化代码？

巴菲特框架的本质是**定性判断**（护城河类型、管理层质量、能力圈）而非纯量化筛选。评分权重和阈值定义在 JSON 配置中作为参考，但实际分析由 LLM 基于框架 prompt 执行。这允许：

- 灵活处理非标准化数据（如"命名一个具体的护城河来源"）
- 自然语言输出比纯数值评分更具可读性
- 可通过修改 prompt 而非代码来调整分析逻辑

### 4.2 Persona 隔离

每个 (user_id, agent_id) 组合有独立的：
- Memory 数据库（对话历史）
- Storage 数据库（session 持久化）
- Knowledge 目录（RAG 知识库）
- Agentic Memory 数据库（自动学习）

确保不同用户的巴菲特 agent 分析不会互相干扰。

### 4.3 Agent 复用

`PersonaRegistry` 的 LRU 缓存（默认 8 个槽位）确保：
- 同一个用户的巴菲特 agent 跨调用复用状态
- 上下文记忆在 session 间保持连续
- 不活跃的 persona 会被 evict 并正确关闭资源

---

## 五、文件索引

| 文件 | 角色 | 行数范围 |
|------|------|---------|
| `scripts/agents/TraderInvestorsAgent/configs/agent_definitions.json` | Agent 配置定义 | 4-133 |
| `scripts/agents/finagent_core/main.py` | Python 入口 + action 分发 | 261-332 |
| `scripts/agents/finagent_core/core_agent.py` | 核心编排器 | 全文件 |
| `scripts/agents/finagent_core/persona_runtime.py` | Persona 状态管理 | 全文件 |
| `scripts/agents/finagent_core/persona_registry.py` | LRU 缓存管理 | 全文件 |
| `scripts/agents/finagent_core/agent_loader.py` | Agent 发现与加载 | 全文件 |
| `scripts/agents/finagent_core/registries/tools_registry.py` | 工具注册表 | - |
| `scripts/agents/finagent_core/registries/models_registry.py` | 模型注册表 | - |
| `scripts/agents/finagent_core/modules/memory_module.py` | 记忆模块 | - |
| `src/services/agents/AgentService.h` | C++ 服务接口 | 全文件 |
| `src/services/agents/AgentService.cpp` | C++ 服务实现 | - |
| `src/services/agents/AgentTypes.h` | C++ 数据类型 | - |
| `src/screens/agent_config/AgentChatPanel.h` | 聊天 UI | - |
| `src/screens/agent_config/AgentsViewPanel.h` | Agent 编辑 UI | - |
| `scripts/akshare_analysis.py` | 巴菲特指数数据 | 245-258 |
| `src/services/akshare/AkShareService.cpp` | AKShare 服务 | - |
