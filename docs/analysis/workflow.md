# Workflow 系统完整分析

## 概述

FinceptTerminal 的 Workflow 系统由三个独立的子系统组成，分属不同的抽象层级：

| 子系统 | 语言 | 执行引擎 | 用途 |
|--------|------|---------|------|
| **Agno Workflow** | Python | Agno 框架原生 Workflow | 预定义金融分析流水线（股票分析、组合再平衡、风险评估） |
| **RenTech Workflow** | Python | Agno Workflow + 自研 Agent 协作 | 量化对冲基金完整交易周期（信号发现→IC决策→执行→复盘） |
| **Node Editor Workflow** | C++ | 自研 DAG 执行器 | 可视化拖拽式工作流编辑与执行 |

三个子系统共享 AgentService 的 Python 桥接通道，但各自有独立的执行逻辑和数据模型。

---

## 目录

- [核心作用总结](#核心作用总结)
- [一、Agno Workflow（Python 预定义流水线）](#一agno-workflowpython-预定义流水线)
  - [1.1 架构概览](#11-架构概览)
  - [1.2 StockAnalysisWorkflow](#12-stockanalysisworkflow)
  - [1.3 PortfolioRebalancingWorkflow](#13-portfoliorebalancingworkflow)
  - [1.4 RiskAssessmentWorkflow](#14-riskassessmentworkflow)
  - [1.5 工厂类与调用链](#15-工厂类与调用链)
- [二、RenTech Workflow（量化对冲基金流水线）](#二rentech-workflow量化对冲基金流水线)
  - [2.1 架构概览](#21-架构概览)
  - [2.2 数据模型](#22-数据模型)
  - [2.3 SignalDiscoveryWorkflow](#23-signaldiscoveryworkflow)
  - [2.4 DailyTradingCycleWorkflow](#24-dailytradingcycleworkflow)
  - [2.5 完整子工作流清单](#25-完整子工作流清单)
- [三、Node Editor Workflow（C++ 可视化工作流）](#三node-editor-workflowc-可视化工作流)
  - [3.1 架构概览](#31-架构概览)
  - [3.2 核心组件](#32-核心组件)
  - [3.3 节点类型体系](#33-节点类型体系)
  - [3.4 执行引擎](#34-执行引擎)
- [四、Execution Planner（动态计划生成）](#四execution-planner动态计划生成)
- [五、C++ 服务层与 Python 桥接](#五c-服务层与-python-桥接)
- [六、完整文件索引](#六完整文件索引)

---

## 核心作用总结

### 解决的核心问题

单个 AI Agent 一次调用只能做一件事。但金融分析天然是多步骤、有依赖、需并行的复杂任务。Workflow 把多个 Agent 调用编排成有序流水线，将"人指挥 AI 做一步"升级为"人设定目标，AI 自动编排多步执行"。

### 六大核心能力

#### 1. 多步骤编排（Sequential Orchestration）

将复杂金融分析拆解为有序步骤，前序结果自动传递给后续步骤。

不使用 Workflow 时需要用户手动执行 5 次查询：
```
用户：获取 AAPL 行情数据
用户：基于上面的数据做技术分析
用户：再做基本面分析
用户：给出买卖建议
用户：生成完整报告
```

Workflow 自动完成，用户只需输入一个 ticker：
```
fetch_market_data → technical_analysis + fundamental_analysis（并行） → generate_recommendation → create_report
```

实现方式：
- Python Agno Workflow：`Step(name="xxx", executor=fn)` 链式组合
- RenTech Workflow：4 个 Agent 按顺序执行，每步结果写入 `WorkflowContext`
- C++ Node Editor：DAG 边定义依赖关系，拓扑排序后顺序执行

#### 2. 并行执行（Parallel Execution）

互不依赖的步骤同时运行，减少总耗时。

| 场景 | 串行耗时 | 并行耗时 |
|------|---------|---------|
| 技术分析 + 基本面分析 | ~20s + ~20s = 40s | ~20s |
| 多 ticker 信号发现 | N × ~15s | ~15s |

实现方式：
- Python Agno Workflow：`Parallel(Step(...), Step(...))` 原语
- C++ Node Editor：DAG 拓扑排序后同层节点并行执行
- RenTech DailyCycle：多 ticker × 多 signal_type 的信号发现循环

#### 3. 条件分支与路由（Conditional Branching）

根据中间结果动态选择执行路径，避免无效计算。

**组合再平衡的 Condition**：
```
analyze_portfolio → 偏离度检查
  → 有偏离？  是 → rebalance_plan → generate_orders
  → 无偏离？  否 → no_action（跳过后续步骤）
```
判断逻辑：检查 LLM 输出中是否包含 "drift"/"rebalance"/"overweight" 等关键词。

**风险评估的 Router**：
```
identify_risk_type → 判断风险类型
  → "credit"    → credit_risk_step
  → "liquidity" → liquidity_risk_step
  → "general"   → general_risk_step
  → 默认         → market_risk_step
```

实现方式：
- Python Agno Workflow：`Condition(predicate, branches)` 和 `Router(selector, choices)`
- C++ Node Editor：`ControlFlowNodes` 提供条件分支节点

#### 4. 多 Agent 协作（Multi-Agent Collaboration）

不同专业领域的 Agent 在同一流水线中各司其职，Workflow 负责串联和传递上下文。

RenTech 的例子最典型——10 个专业化 Agent 按角色分工：
```
Data Scientist      → 数据清洗，保证输入质量
Signal Scientist    → 统计信号发现，要求 p < 0.01
Quant Researcher    → 回测验证，排除过拟合
Research Lead       → 质量审查，决定是否提交 IC
Risk Quant          → VaR / 回撤 / 压力测试
Compliance Officer  → 合规检查
Portfolio Manager   → 仓位建议
IC Chair            → 最终审批（0.5075 胜率阈值）
Execution Trader    → TWAP / VWAP 执行
Market Maker        → 流动性提供与价差捕获
```

每个 Agent 有独立的 system prompt、工具集和评判标准，Workflow 通过 `WorkflowContext` 在步骤间传递累积结果。

#### 5. 可复用模板（Reusable Templates）

将成熟的分析流程固化为模板，用户选一个参数就能跑完整个分析。

预定义模板（10 种）：

| 模板 | 输入 | 输出 | 底层 Workflow |
|------|------|------|--------------|
| Stock Analysis | ticker | 综合投资报告 | StockAnalysisWorkflow |
| Portfolio Rebalancing | 持仓数据 | 再平衡交易指令 | PortfolioRebalancingWorkflow |
| Risk Assessment | 组合数据 | 风险评估报告 | RiskAssessmentWorkflow |
| Macro Scan | — | 宏观经济扫描 | LLM 单次调用 |
| Earnings Brief | ticker | 财报摘要 | LLM 单次调用 |
| Sector Rotation | — | 板块轮动分析 | LLM 单次调用 |
| Options Flow Scan | — | 期权异动扫描 | LLM 单次调用 |
| Sentiment Scan | — | 市场情绪扫描 | LLM 单次调用 |
| Custom Query | 自定义 prompt | 自定义分析 | LLM 单次调用 |
| Daily Trading Cycle | tickers + signal_types | 完整交易漏斗 | DailyTradingCycleWorkflow |

#### 6. 动态计划生成（Dynamic Planning）

用户输入自然语言，LLM 自动生成多步执行计划，支持依赖关系和并行。

```
用户输入："分析 AAPL 和 MSFT 的投资机会，比较后推荐一个"

Execution Planner 生成：
  Step 1 (parallel): 获取 AAPL 数据 | 获取 MSFT 数据
  Step 2: AAPL 技术分析（依赖 Step 1 AAPL）
  Step 3: MSFT 技术分析（依赖 Step 1 MSFT）
  Step 4: 基本面对比（依赖 Step 2, 3）
  Step 5: 生成推荐报告（依赖 Step 4）
```

两种生成方式：
- **LLM 动态生成**：将用户查询 + JSON schema 约束发给 LLM，返回结构化 PlanStep 列表
- **模板回退**：当 LLM 生成失败时，使用预定义的 `stock_analysis_plan()` 或 `portfolio_rebalance_plan()`

### 三个子系统能力对比

| 能力 | Agno Workflow | RenTech Workflow | Node Editor (C++) |
|------|:---:|:---:|:---:|
| 多步骤编排 | Step 原语 | Agent 链式调用 | DAG 边依赖 |
| 并行执行 | Parallel 原语 | 阶段内循环 | DAG 分层并行 |
| 条件分支 | Condition | 信号关键词过滤 | ControlFlowNodes |
| 动态路由 | Router | — | — |
| 循环执行 | Loop | ticker × signal 循环 | Loop 节点 |
| 多 Agent 协作 | 单 Agent 多步调用 | 10 Agent 专业分工 | AgentNodes 桥接 Python |
| 可视化编辑 | 不支持 | 不支持 | 拖拽式画布 |
| 用户自定义 | 改 Python 代码 | 改 Python 代码 | 拖拽连线，无需写代码 |
| 执行计划生成 | 不支持 | 不支持 | — |
| 状态持久化 | Session 存储 | WorkflowContext | WorkflowService 持久化 |

---

## 一、Agno Workflow（Python 预定义流水线）

### 1.1 架构概览

**文件**: `scripts/agents/finagent_core/modules/workflow_module.py`

基于 Agno 框架原生 Workflow 子类模式，使用 5 种流程控制原语：

| 原语 | Agno 类 | 功能 |
|------|---------|------|
| 顺序执行 | `Step` | 单步 LLM 调用 |
| 并行执行 | `Parallel` | 多步同时执行，结果合并 |
| 条件分支 | `Condition` | 布尔判断，选择执行路径 |
| 路由选择 | `Router` | 多路分支，动态选择一条执行 |
| 循环执行 | `Loop` | 重复执行直到条件满足或达到上限 |

每个 Step 内部由一个 LLM Agent（`_make_workflow_agent`）执行，Agent 可携带工具（yfinance、duckduckgo、TerminalToolkit）。

### 1.2 StockAnalysisWorkflow

**文件**: `workflow_module.py:128-215`

股票分析流水线，5 个步骤：

```
┌─────────────────────┐
│ fetch_market_data    │  获取价格、成交量、新闻
└─────────┬───────────┘
          │
    ┌─────┴──────┐
    │  Parallel   │
    ┌┴──────────┐┌┴───────────────────┐
    │ technical ││ fundamental_analysis│  并行：技术+基本面
    │ _analysis ││                    │
    └────┬──────┘└──────┬─────────────┘
         └──────┬───────┘
                │
┌───────────────┴──────────────┐
│ generate_recommendation      │  买入/卖出/持有建议
└───────────────┬──────────────┘
                │
┌───────────────┴──────────────┐
│ create_report                │  综合投资报告
└──────────────────────────────┘
```

**关键实现**：
- 每个 step 是一个 Python 闭包，接收 `StepInput`，返回 `StepOutput`
- `fetch_step` → `tech_step` + `fund_step`（Parallel）→ `rec_step` → `report_step`
- LLM Agent 通过 `agent.run(prompt)` 执行，prompt 中注入前序步骤的 `previous_step_content`
- 工具：yfinance（行情数据）+ duckduckgo（新闻搜索）+ TerminalToolkit（内部 MCP）

### 1.3 PortfolioRebalancingWorkflow

**文件**: `workflow_module.py:219-283`

组合再平衡流水线，使用 `Condition` 原语实现条件分支：

```
┌─────────────────────┐
│ analyze_portfolio   │  计算配置偏离度
└─────────┬───────────┘
          │
┌─────────┴───────────┐
│ Condition(check_drift)│  关键词判断：drift/rebalance/overweight...
└───┬─────────┬───────┘
    │ YES     │ NO (implicit)
┌───┴──────┐  │
│rebalance │  │  计算再平衡方案
│ _plan    │  │
└───┬──────┘  │
    └────┬────┘
         │
┌────────┴───────────┐
│ generate_orders    │  生成具体交易指令
└────────────────────┘
```

**条件判断逻辑**（`needs_rebal` 函数，第 263-266 行）：
```python
def needs_rebal(step_input):
    prev = str(getattr(step_input, "previous_step_content", ""))
    keywords = ["drift", "rebalance", "exceed", "above", "below", "overweight", "underweight"]
    return any(k in prev.lower() for k in keywords)
```

### 1.4 RiskAssessmentWorkflow

**文件**: `workflow_module.py:287-374`

风险评估流水线，使用 `Router` 原语实现多路分支：

```
┌─────────────────────┐
│ identify_risk_type  │  识别风险类型
└─────────┬───────────┘
          │
┌─────────┴───────────┐
│ Router(risk_router)  │  根据 LLM 输出关键词路由
└──┬──────┬──────┬────┬┘
   │      │      │    │
┌──┴──┐┌──┴───┐┌┴───┐┌┴──────┐
│market││credit││liq ││general│  4 种风险分析
│_risk ││_risk ││_uid││_risk  │
└──┬───┘└──┬───┘└─┬──┘└──┬────┘
   └───────┴────────┴──────┘
           │
┌──────────┴──────────┐
│ generate_risk_report│  综合风险报告
└─────────────────────┘
```

**路由逻辑**（`selector` 函数，第 356-361 行）：
```python
def selector(step_input):
    prev = str(getattr(step_input, "previous_step_content", "")).lower()
    if "credit"    in prev: return [credit_s]
    if "liquidity" in prev: return [liquid_s]
    if "general"   in prev: return [general_s]
    return [market_s]  # 默认路由到市场风险
```

### 1.5 工厂类与调用链

**工厂类**: `FinancialWorkflowTemplates`（第 378-422 行）

静态方法统一创建 workflow 实例，加载工具，注入 TerminalToolkit：

```python
class FinancialWorkflowTemplates:
    @staticmethod
    def stock_analysis_pipeline(api_keys, model_config, tools,
                                 terminal_endpoint, terminal_tool_defs, ...) -> StockAnalysisWorkflow:
        _tools = tools or _load_workflow_tools(api_keys, terminal_endpoint, ...)
        return StockAnalysisWorkflow(api_keys, model_config, tools=_tools)
```

**调用链**：
```
C++ AgentService::run_workflow("stock_analysis", params)
  → Python main.py dispatch_action("stock_analysis")
    → CoreAgent(api_keys).run_stock_analysis(symbol, config)
      → FinancialWorkflowTemplates.stock_analysis_pipeline(api_keys, ...)
        → StockAnalysisWorkflow(api_keys, model_config, tools)
          → workflow.run(input={"symbol": "AAPL"})
            → Agno Workflow 引擎执行 Steps
```

**main.py 中的 workflow actions**（第 621-648 行）：
| Action | 说明 |
|--------|------|
| `stock_analysis` | 股票分析流水线 |
| `portfolio_rebal` | 组合再平衡 |
| `risk_assessment` | 风险评估 |
| `macro_scan` | 宏观扫描 |
| `earnings_brief` | 财报简报 |
| `sector_rotation` | 板块轮动 |
| `options_scan` | 期权扫描 |
| `sentiment_scan` | 情绪扫描 |
| `custom_query` | 自定义查询 |

---

## 二、RenTech Workflow（量化对冲基金流水线）

### 2.1 架构概览

**目录**: `scripts/agents/hedgeFundAgents/renaissance_technologies_hedge_fund_agent/`

仿照文艺复兴科技（Renaissance Technologies）Medallion 基金的运作模式，实现完整的量化交易闭环。包含 10 个专业化 Agent 和 6 个子工作流。

**Agent 团队**：

| 团队 | Agent | 职责 |
|------|-------|------|
| Research Team | Signal Scientist | 统计信号发现（p<0.01） |
| | Data Scientist | 数据质量与特征工程 |
| | Quant Researcher | 策略开发与回测 |
| | Research Lead | 研究验证与协调 |
| Trading Team | Execution Trader | 最优执行（TWAP/VWAP） |
| | Market Maker | 流动性提供与价差捕获 |
| Risk Team | Risk Quant | VaR、回撤、压力测试 |
| | Compliance Officer | 合规检查 |
| Investment Committee | Portfolio Manager | 仓位管理与组合构建 |
| | IC Chair | 最终审批（0.5075 胜率阈值） |

### 2.2 数据模型

**文件**: `workflows/base.py`

基于 Pydantic BaseModel 的强类型数据模型，贯穿整个交易流水线：

```
SignalData          → 信号数据（ticker, direction, strength, p_value, confidence, expected_return_bps）
RiskAssessment      → 风险评估（marginal_var, position_size, stress_loss, risk_score, approved）
ExecutionPlan       → 执行计划（target_quantity, algorithm, urgency, cost_estimates）
ExecutionResult     → 执行结果（filled_quantity, avg_price, vs_vwap, implementation_shortfall）
TradeDecision       → 交易决策（APPROVED/REJECTED, approved_quantity, stop_loss）
WorkflowContext     → 工作流上下文（累积所有上述对象 + step_history）
```

辅助类型：
- `WorkflowStatus`: PENDING → RUNNING → COMPLETED / FAILED / CANCELLED
- `StepStatus`: PENDING → RUNNING → COMPLETED / FAILED / SKIPPED
- `StepResult`: 单步结果（step_name, agent, duration, output）
- `WorkflowResult`: 完整工作流结果（所有 StepResult 汇总）

### 2.3 SignalDiscoveryWorkflow

**文件**: `workflows/signal_discovery.py`

4 步顺序流水线，每个步骤由不同的 Agent 执行：

```
┌─────────────────────┐
│ Data Scientist      │  数据准备：获取 OHLCV、清洗、特征工程
└─────────┬───────────┘
          │
┌─────────┴───────────┐
│ Signal Scientist    │  模式检测：统计测试、信号强度、p 值
└─────────┬───────────┘
          │
┌─────────┴───────────┐
│ Quant Researcher    │  统计验证：回测、交叉验证、过拟合检查
└─────────┬───────────┘
          │
┌─────────┴───────────┐
│ Research Lead       │  质量审查：ADVANCE_TO_IC / REVISE / REJECT
└─────────────────────┘
```

关键约束：信号必须满足 `p_value < 0.01` 才能进入下一阶段。

### 2.4 DailyTradingCycleWorkflow

**文件**: `workflows/daily_cycle.py`

主工作流 — 模拟完整交易日。编排 5 个子工作流为 6 个阶段：

```
┌─────────────────────────────────────────────────────────────────┐
│                    DAILY TRADING CYCLE                           │
│                                                                 │
│  Phase 1: PRE-MARKET ─── Signal Discovery                       │
│    对每个 ticker × signal_type 运行 SignalDiscoveryWorkflow      │
│    输出: discovered_signals[]                                    │
│              │                                                  │
│  Phase 2: VALIDATION ─── Signal Validation                      │
│    对每个 discovered_signal 运行 SignalValidationWorkflow        │
│    过滤: recommendation == "SUBMIT_TO_IC"                       │
│    输出: validated_signals[]                                     │
│              │                                                  │
│  Phase 3: RISK REVIEW ─── Risk Assessment                       │
│    对每个 validated_signal 运行 RiskAssessmentWorkflow           │
│    过滤: decision == "APPROVED"                                  │
│    输出: risk_approved_signals[]                                 │
│              │                                                  │
│  Phase 4: IC DECISION ─── Investment Committee                   │
│    Portfolio Manager: 仓位建议                                    │
│    IC Chair: 最终审批（APPROVED / REJECTED）                     │
│    输出: approved_trades[] (TradeDecision[])                     │
│              │                                                  │
│  Phase 5: EXECUTION ─── Trade Execution                          │
│    对每个 approved_trade 运行 ExecutionPipelineWorkflow          │
│    输出: executions[] (ExecutionResult[])                        │
│              │                                                  │
│  Phase 6: POST-MARKET ─── Post-Trade Analysis                   │
│    对每个 execution 运行 PostTradeWorkflow                       │
│    分析与学习反馈                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**漏斗统计**：
```
N discovered → M validated (M/N%) → K approved (K/M%) → J executed (J/K%)
```

### 2.5 完整子工作流清单

| 文件 | 工作流 | 步骤数 | 功能 |
|------|--------|--------|------|
| `signal_discovery.py` | SignalDiscoveryWorkflow | 4 | 数据准备→模式检测→统计验证→研究审查 |
| `signal_validation.py` | SignalValidationWorkflow | - | 信号质量验证 |
| `risk_assessment.py` | RiskAssessmentWorkflow | - | VaR/回撤/压力测试 |
| `execution_pipeline.py` | ExecutionPipelineWorkflow | - | TWAP/VWAP 执行 |
| `post_trade_analysis.py` | PostTradeWorkflow | - | 盘后分析与学习 |
| `daily_cycle.py` | DailyTradingCycleWorkflow | 6 phases | 编排上述全部子工作流 |

---

## 三、Node Editor Workflow（C++ 可视化工作流）

### 3.1 架构概览

C++ 端的可视化工作流系统，基于 DAG（有向无环图）执行引擎。用户通过拖拽节点、连线的方式构建工作流，系统自动解析依赖关系并并行执行。

**核心文件**（`src/services/workflow/`）：

| 文件 | 职责 |
|------|------|
| `WorkflowService.h/cpp` | 工作流 CRUD、持久化、执行调度 |
| `WorkflowExecutor.h/cpp` | DAG 执行引擎（拓扑排序、并行执行、环检测） |
| `WorkflowCache.h/cpp` | 执行缓存 |
| `NodeRegistry.h/cpp` | 节点类型注册表 |
| `ExpressionEngine.h/cpp` | 参数表达式引擎 |
| `ParameterProcessor.h/cpp` | 参数处理 |
| `RiskManager.h/cpp` | 工作流级风险管理 |
| `ExecutionHooks.h/cpp` | 执行前后钩子 |
| `AuditLogger.h/cpp` | 审计日志 |
| `ConfirmationService.h/cpp` | 人工确认服务 |
| `Extensions.h/cpp` | 扩展点 |

### 3.2 核心组件

**数据结构**（`src/screens/node_editor/NodeEditorTypes.h`）：

```cpp
struct WorkflowDef {
    QString id;
    QString name;
    QString description;
    QVector<NodeDef> nodes;    // 节点定义
    QVector<EdgeDef> edges;    // 连线定义
    QJsonObject metadata;
};
```

**UI 层**：

| UI 组件 | 文件 | 功能 |
|---------|------|------|
| NodeEditorScreen | `src/screens/node_editor/NodeEditorScreen.h` | 可视化画布编辑器 |
| WorkflowsViewPanel | `src/screens/agent_config/WorkflowsViewPanel.h` | 预定义工作流面板 |
| PlannerViewPanel | `src/screens/agent_config/PlannerViewPanel.h` | 计划生成与执行面板 |

### 3.3 节点类型体系

**目录**: `src/services/workflow/nodes/`

| 节点类型 | 文件 | 功能 |
|---------|------|------|
| **TriggerNodes** | `TriggerNodes.h/cpp` | 工作流启动触发器 |
| **ControlFlowNodes** | `ControlFlowNodes.h/cpp` | 条件、循环、分支 |
| **DataFormatNodes** | `DataFormatNodes.h/cpp` | 数据格式转换 |
| **MarketDataNodes** | `MarketDataNodes.h/cpp` | 行情数据获取 |
| **TradingNodes** | `TradingNodes.h/cpp` | 交易执行 |
| **AnalyticsNodes** | `AnalyticsNodes.h/cpp` | 分析计算 |
| **AgentNodes** | `AgentNodes.h/cpp` | Python Agent 调用（桥接 Python 工作流） |
| **IntegrationNodes** | `IntegrationNodes.h/cpp` | 外部系统集成 |
| **NotificationNodes** | `NotificationNodes.h/cpp` | 通知告警 |
| **SafetyNodes** | `SafetyNodes.h/cpp` | 风险检查与验证 |
| **FileNodes** | `FileNodes.h/cpp` | 文件操作 |
| **UtilityNodes** | `UtilityNodes.h/cpp` | 通用工具 |

`NodeRegistry` 管理所有节点类型的注册和实例化。`AgentNodes` 是 C++ 与 Python 的桥接点 — 允许在可视化工作流中调用 Python Agent 执行。

### 3.4 执行引擎

**文件**: `src/services/workflow/WorkflowExecutor.h/cpp`

```
WorkflowDef (DAG)
     │
     ▼
拓扑排序 (Topological Sort)
     │
     ▼
分层并行执行
┌───────────────────────────────────────────┐
│ Level 0: [Node A] [Node B]  ← 无依赖，并行 │
│ Level 1: [Node C]            ← 依赖 A, B   │
│ Level 2: [Node D] [Node E]  ← 依赖 C，并行  │
│ Level 3: [Node F]            ← 依赖 D, E   │
└───────────────────────────────────────────┘
     │
     ▼
结果聚合 → 信号发射 → UI 更新
```

特性：
- DAG 拓扑排序，自动检测循环依赖
- 按层级并行执行（同层无依赖的节点并行运行）
- 信号驱动的进度报告（Qt signals）
- 通过 `AgentNodes` 可调用 Python Agent/Workflow

---

## 四、Execution Planner（动态计划生成）

**文件**: `scripts/agents/finagent_core/execution_planner.py`

动态计划系统，可从自然语言查询生成多步执行计划。

**数据模型**：

```python
class StepType(Enum):
    AGENT = "agent"           # LLM Agent 执行
    TOOL = "tool"             # 工具直接调用
    CONDITION = "condition"   # 条件分支
    PARALLEL = "parallel"     # 并行执行
    LOOP = "loop"             # 循环执行
    WAIT = "wait"             # 等待用户输入
    CHECKPOINT = "checkpoint" # 状态检查点

@dataclass
class PlanStep:
    id: str
    name: str
    step_type: StepType
    config: Dict[str, Any]
    dependencies: List[str]   # 依赖的 step id 列表
    status: StepStatus
    result: Any
    error: Optional[str]

@dataclass
class ExecutionPlan:
    id: str
    name: str
    description: str
    steps: List[PlanStep]
    context: Dict[str, Any]
    status: StepStatus
    current_step_index: int
```

**计划模板**：
- `stock_analysis_plan()` — 并行数据获取 + 分析
- `portfolio_rebalance_plan()` — 持仓获取 → 风险分析 → 再平衡建议

**调用链**：
```
C++ AgentService::create_plan(query, config)
  → Python main.py dispatch_action("create_plan")
    → ExecutionPlanner.generate_plan(query)
      → 方式 1: LLM 动态生成（JSON schema 约束）
      → 方式 2: 模板回退（预定义 PlanStep 序列）
  ← ExecutionPlan (JSON)
  → C++ emit plan_created(ExecutionPlan)

C++ AgentService::execute_plan(plan, config)
  → Python dispatch_action("execute_plan")
    → 按 dependencies 顺序执行每个 PlanStep
  ← ExecutionPlan (updated with results)
  → C++ emit plan_executed(ExecutionPlan)
```

**C++ 侧的 ExecutionPlan**（`src/services/agents/AgentTypes.h`）：

```cpp
struct PlanStep {
    QString id;
    QString name;
    QString step_type;
    QJsonObject config;
    QStringList dependencies;
    QString status;  // "pending", "running", "completed", "failed"
    QString result;
    QString error;
};

struct ExecutionPlan {
    QString id;
    QString name;
    QString description;
    QVector<PlanStep> steps;
    QString status;
    bool is_complete = false;
    bool has_failed = false;
    QString request_id;
};
```

---

## 五、C++ 服务层与 Python 桥接

### AgentService 中的 Workflow 方法

**文件**: `src/services/agents/AgentService.h`

```cpp
// 预定义工作流
QString run_workflow(const QString& workflow_type, const QJsonObject& params);

// 动态计划
QString create_plan(const QString& query, const QJsonObject& config);
QString execute_plan(const QJsonObject& plan, const QJsonObject& config);

// 金融工作流快捷方式
void run_stock_analysis(const QString& symbol, const QJsonObject& config);
void run_portfolio_rebalancing(const QJsonObject& portfolio_data);
void run_risk_assessment(const QJsonObject& portfolio_data);
void run_portfolio_analysis(const QString& analysis_type, const QJsonObject& portfolio_summary);

// 特定计划模板
QString create_stock_analysis_plan(const QString& symbol, const QJsonObject& config);
QString create_portfolio_plan(const QJsonObject& goals, const QJsonObject& config);
```

### 信号（Signals）

```cpp
signals:
    void agent_result(AgentExecutionResult result);          // 工作流最终结果
    void agent_stream_token(const QString& request_id, ...); // 流式 token
    void agent_stream_thinking(const QString& request_id, ..); // 思考状态
    void agent_stream_done(AgentExecutionResult result);     // 流式完成
    void plan_created(ExecutionPlan plan);                   // 计划生成完成
    void plan_executed(ExecutionPlan plan);                  // 计划执行完成
```

### DataHub Topics

AgentService 作为 push-only Producer，发布到以下 DataHub topics：

| Topic Pattern | 用途 |
|---------------|------|
| `agent:output:<request_id>` | 工作流最终结果 |
| `agent:stream:<request_id>` | 流式 token（50ms 合并） |
| `agent:status:<request_id>` | 思考/工具调用状态 |
| `agent:routing:<request_id>` | 路由决策 |
| `agent:error:*` | 错误信息 |

---

## 六、完整文件索引

### Python — Agno Workflow

| 文件 | 行数 | 职责 |
|------|------|------|
| `scripts/agents/finagent_core/modules/workflow_module.py` | 575 | 全部：3 个 Workflow 类 + 工厂 + Builder |
| `scripts/agents/finagent_core/core_agent.py` | 993 | CoreAgent.run_stock_analysis / run_portfolio_rebalancing / run_risk_assessment |
| `scripts/agents/finagent_core/main.py` | - | dispatch_action: run_workflow, create_plan, execute_plan, stock_analysis 等 |

### Python — RenTech Workflow

| 文件 | 行数 | 职责 |
|------|------|------|
| `scripts/agents/hedgeFundAgents/renaissance_technologies_hedge_fund_agent/workflows/base.py` | 294 | 数据模型 + WorkflowContext |
| `scripts/agents/hedgeFundAgents/renaissance_technologies_hedge_fund_agent/workflows/signal_discovery.py` | 347 | 信号发现 4 步流水线 |
| `scripts/agents/hedgeFundAgents/renaissance_technologies_hedge_fund_agent/workflows/signal_validation.py` | - | 信号验证 |
| `scripts/agents/hedgeFundAgents/renaissance_technologies_hedge_fund_agent/workflows/risk_assessment.py` | - | 风险评估 |
| `scripts/agents/hedgeFundAgents/renaissance_technologies_hedge_fund_agent/workflows/execution_pipeline.py` | - | 交易执行 |
| `scripts/agents/hedgeFundAgents/renaissance_technologies_hedge_fund_agent/workflows/post_trade_analysis.py` | - | 盘后分析 |
| `scripts/agents/hedgeFundAgents/renaissance_technologies_hedge_fund_agent/workflows/daily_cycle.py` | 438 | 主编排器：6 阶段完整交易日 |
| `scripts/agents/hedgeFundAgents/configs/renaissance_technologies_agent.json` | - | RenTech Agent 配置 |

### Python — Execution Planner

| 文件 | 职责 |
|------|------|
| `scripts/agents/finagent_core/execution_planner.py` | PlanStep, ExecutionPlan, 动态计划生成 |

### C++ — Workflow Service

| 文件 | 职责 |
|------|------|
| `src/services/workflow/WorkflowService.h/cpp` | 工作流管理与调度 |
| `src/services/workflow/WorkflowExecutor.h/cpp` | DAG 执行引擎 |
| `src/services/workflow/NodeRegistry.h/cpp` | 节点类型注册 |
| `src/services/workflow/ExpressionEngine.h/cpp` | 参数表达式 |
| `src/services/workflow/ParameterProcessor.h/cpp` | 参数处理 |
| `src/services/workflow/RiskManager.h/cpp` | 风险管理 |
| `src/services/workflow/ExecutionHooks.h/cpp` | 执行钩子 |
| `src/services/workflow/AuditLogger.h/cpp` | 审计日志 |
| `src/services/workflow/ConfirmationService.h/cpp` | 确认服务 |
| `src/services/workflow/WorkflowCache.h/cpp` | 执行缓存 |
| `src/services/workflow/Extensions.h/cpp` | 扩展点 |
| `src/services/workflow/adapters/ServiceBridges.h/cpp` | 服务桥接 |

### C++ — Workflow Nodes

| 文件 | 职责 |
|------|------|
| `src/services/workflow/nodes/TriggerNodes.h/cpp` | 触发器节点 |
| `src/services/workflow/nodes/ControlFlowNodes.h/cpp` | 控制流节点 |
| `src/services/workflow/nodes/DataFormatNodes.h/cpp` | 数据格式节点 |
| `src/services/workflow/nodes/MarketDataNodes.h/cpp` | 行情数据节点 |
| `src/services/workflow/nodes/TradingNodes.h/cpp` | 交易节点 |
| `src/services/workflow/nodes/AnalyticsNodes.h/cpp` | 分析节点 |
| `src/services/workflow/nodes/AgentNodes.h/cpp` | Python Agent 桥接节点 |
| `src/services/workflow/nodes/IntegrationNodes.h/cpp` | 集成节点 |
| `src/services/workflow/nodes/NotificationNodes.h/cpp` | 通知节点 |
| `src/services/workflow/nodes/SafetyNodes.h/cpp` | 安全节点 |
| `src/services/workflow/nodes/FileNodes.h/cpp` | 文件节点 |
| `src/services/workflow/nodes/UtilityNodes.h/cpp` | 工具节点 |

### C++ — Agent Service（Workflow 相关）

| 文件 | 职责 |
|------|------|
| `src/services/agents/AgentService.h` | run_workflow, create_plan, execute_plan 等 |
| `src/services/agents/AgentService.cpp` | 实现细节 |
| `src/services/agents/AgentTypes.h` | PlanStep, ExecutionPlan 数据结构 |

### C++ — UI

| 文件 | 职责 |
|------|------|
| `src/screens/node_editor/NodeEditorScreen.h/cpp` | 可视化工作流编辑器 |
| `src/screens/node_editor/NodeEditorTypes.h` | WorkflowDef, NodeDef, EdgeDef |
| `src/screens/agent_config/WorkflowsViewPanel.h/cpp` | 预定义工作流面板 |
| `src/screens/agent_config/PlannerViewPanel.h/cpp` | 计划生成与执行面板 |
