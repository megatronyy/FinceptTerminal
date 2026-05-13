# FinceptTerminal Nodes 模块架构分析

> 生成日期：2026-05-08
> 节点类型：70+
> 节点分类：12 大类
> 执行引擎：DAG + 拓扑排序 + 并行执行

---

## 一、系统总览

Nodes 模块是一个**可视化金融工作流编排系统**，用户通过拖拽节点、连线构建自动化金融工作流。系统包含 70+ 内置节点类型、DAG 执行引擎、可视化画布编辑器和 SQLite 持久化。

### 1.1 整体架构

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Node Editor UI                               │
│                                                                      │
│  ┌──────────┐  ┌──────────────────────────┐  ┌──────────────────┐   │
│  │  Palette  │  │       Canvas (QGraphics)  │  │ Properties Panel │   │
│  │  节点面板  │  │  节点拖拽 / 连线 / 缩放   │  │  参数编辑        │   │
│  │  搜索/分类 │  │  小地图 / 橡皮筋选择      │  │                  │   │
│  └────┬─────┘  └────────────┬─────────────┘  └────────┬─────────┘   │
│       └────────────────────┼──────────────────────────┘              │
│                              │                                       │
│  ┌──────────────────────────▼────────────────────────────────────┐  │
│  │                    ExecutionResultsPanel                       │  │
│  │              执行结果面板（底部抽屉，可折叠）                     │  │
│  └───────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────┬───────────────────────────────────┘
                                   │
┌──────────────────────────────────▼───────────────────────────────────┐
│                     Workflow Service Layer                            │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────────┐ │
│  │ NodeRegistry  │  │ Workflow     │  │ ExpressionEngine            │ │
│  │ 节点类型注册表 │  │ Executor     │  │ 模板表达式引擎               │ │
│  │ (Singleton)   │  │ DAG 执行引擎  │  │ ={{...}} 变量替换           │ │
│  └──────┬───────┘  └──────┬───────┘  └────────────────────────────┘ │
│         │                 │                                          │
│  ┌──────▼─────────────────▼──────────────────────────────────────┐  │
│  │                    ServiceBridges                              │  │
│  │  将节点执行器连接到实际服务                                      │  │
│  │  MarketDataService / AgentService / NotificationService / ... │  │
│  └───────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────┬───────────────────────────────────┘
                                   │
┌──────────────────────────────────▼───────────────────────────────────┐
│                     Persistence Layer                                 │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────────┐ │
│  │ WorkflowRepo  │  │ SQLite       │  │ Auto-Save                  │ │
│  │ CRUD 操作     │  │ 3 表结构      │  │ 30 秒自动保存               │ │
│  └──────────────┘  └──────────────┘  └────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

### 1.2 源码布局

```
src/
├── services/workflow/
│   ├── WorkflowExecutor.h/cpp      # DAG 执行引擎
│   ├── WorkflowService.h/cpp       # 工作流 CRUD + 执行调度
│   ├── NodeRegistry.h/cpp          # 节点类型注册表 (Singleton)
│   ├── nodes/                      # 70+ 节点实现
│   │   ├── TriggerNodes.cpp        # 触发器节点 (7 种)
│   │   ├── ControlFlowNodes.cpp    # 控制流节点 (6 种)
│   │   ├── MarketDataNodes.cpp     # 市场数据节点 (9 种)
│   │   ├── TradingNodes.cpp        # 交易节点 (8 种)
│   │   ├── AnalyticsNodes.cpp      # 分析节点 (7 种)
│   │   ├── SafetyNodes.cpp         # 安全节点 (4 种)
│   │   ├── AgentNodes.cpp          # AI Agent 节点 (2 种)
│   │   ├── NotificationNodes.cpp   # 通知节点 (6 种)
│   │   ├── FileNodes.cpp           # 文件节点 (3 种)
│   │   ├── DataFormatNodes.cpp     # 数据格式节点 (3 种)
│   │   ├── UtilityNodes.cpp        # 工具节点 (15 种)
│   │   ├── IntegrationNodes.cpp    # 集成节点 (3 种)
│   │   └── ServiceBridges.cpp      # 服务桥接器
│   └── ExpressionEngine.cpp        # 表达式引擎
│
├── screens/node_editor/
│   ├── NodeEditorScreen.h/cpp      # 主界面 (IStatefulScreen)
│   ├── NodeEditorTypes.h           # 核心类型定义
│   ├── canvas/
│   │   ├── NodeCanvas.h/cpp        # QGraphicsView 画布
│   │   ├── NodeScene.h/cpp         # QGraphicsScene 场景
│   │   └── NodeItem.h/cpp          # 节点视觉元素
│   ├── palette/
│   │   └── NodePalette.h/cpp       # 节点面板（分类+搜索）
│   ├── properties/
│   │   ├── NodePropertiesPanel.h/cpp       # 属性编辑面板
│   │   └── ExecutionResultsPanel.h/cpp     # 执行结果面板
│   └── toolbar/
│       └── NodeEditorToolbar.h/cpp # 工具栏
│
└── storage/repositories/
    └── WorkflowRepository.h/cpp    # SQLite 持久化
```

---

## 二、核心类型定义

### 2.1 NodeTypeDef — 节点类型定义

```cpp
struct NodeTypeDef {
    QString type_id;        // 唯一标识，如 "trigger.manual"
    QString display_name;   // 显示名，如 "Manual Trigger"
    QString category;       // 分类，如 "Triggers"
    QString description;    // 描述
    QString icon_text;      // 1-2 字符图标
    QString accent_color;   // 分类主题色
    int version = 1;

    QVector<PortDef> inputs;     // 输入端口定义
    QVector<PortDef> outputs;    // 输出端口定义
    QVector<ParamDef> parameters; // 可配置参数

    // 异步执行函数
    using ExecuteFn = std::function<void(
        const QJsonObject& params,
        const QVector<QJsonValue>& inputs,
        std::function<void(bool success, QJsonValue output, QString error)> callback
    )>;
    ExecuteFn execute;
};
```

### 2.2 PortDef — 端口定义

```cpp
struct PortDef {
    QString id;          // 端口ID，如 "output_true"
    QString label;       // 显示名，如 "True"
    QString data_type;   // 数据类型：Main / MarketData / PriceData / SignalData
    bool is_input;       // 输入或输出
};
```

### 2.3 ParamDef — 参数定义

```cpp
struct ParamDef {
    QString id;            // 参数ID
    QString label;         // 显示名
    QString type;          // 类型：text / number / select / boolean / expression
    QVariant default_value; // 默认值
    QStringList options;    // select 类型的选项列表
    bool required;         // 是否必填
};
```

---

## 三、执行引擎

### 3.1 WorkflowExecutor

**位置：** `src/services/workflow/WorkflowExecutor.h/cpp`

DAG 执行引擎，负责工作流拓扑排序、环检测和并行执行。

#### 核心方法

```cpp
class WorkflowExecutor : public QObject {
    // 执行整个工作流
    void execute(const WorkflowDef& workflow);

    // 从指定节点开始执行（子图执行）
    void execute_from(const WorkflowDef& workflow, const QString& start_node_id);

    // 内部方法
    void launch_ready_nodes();   // 启动入度为 0 的节点
    void collect_inputs(node);   // 收集上游节点输出
    void on_node_done(...);      // 节点完成回调

signals:
    void node_started(QString node_id);
    void node_finished(QString node_id, bool success, QJsonValue result);
    void workflow_finished(bool success);
};
```

#### 执行流程

```
1. 构建邻接表 (Adjacency List)
   从 edges 构建 source → target 映射

2. 环检测 (Cycle Detection)
   DFS 三色标记法 (White → Gray → Black)
   检测到环则中止执行

3. 拓扑排序 (Topological Sort)
   Kahn's Algorithm（BFS 入度法）
   产生分层执行序列

4. 分层并行执行
   Level 0: [A]           ← 入度为 0，立即启动
   Level 1: [B, C]        ← A 完成后，B/C 并行启动
   Level 2: [D]           ← B/C 都完成后启动
   Level 3: [E]           ← D 完成后启动

5. 数据传播
   节点输出 → 写入 result_map → 下游节点 collect_inputs() 读取
   注解标记：_branch (true/false), _route (index)
```

#### 数据流注解

| 注解 | 含义 | 使用场景 |
|------|------|----------|
| `_branch: true/false` | If/Else 分支标记 | 下游节点判断是否接收数据 |
| `_route: 0/1/2` | Switch 路由索引 | 多路分支选择 |
| `_loop_item` | Loop 当前迭代项 | 循环体内使用 |
| `_loop_done` | Loop 完成信号 | 循环结束触发 |

### 3.2 ExpressionEngine — 模板表达式引擎

**位置：** `src/services/workflow/ExpressionEngine.cpp`

```cpp
// 支持的语法
={{$input.price}}          // 引用上游输入
={{$params.symbol}}        // 引用节点参数
={{$input.field.nested}}   // 点号路径解析

// 上下文解析
context = {
    "input": { upstream output },
    "params": { node parameters },
    "env": { environment variables }
}
```

### 3.3 NodeRegistry — 节点注册表

```cpp
class NodeRegistry {  // Singleton
    void register_type(NodeTypeDef def);   // 注册节点类型
    NodeTypeDef* find(const QString& type_id);  // 按 ID 查找
    QVector<NodeTypeDef> all();            // 所有节点
    QVector<NodeTypeDef> by_category(const QString& cat);  // 按分类
    void register_builtin_nodes();         // 注册内置节点
};
```

---

## 四、全部节点类型详解

### 4.1 Trigger Nodes（触发器，7 种）

**文件：** `src/services/workflow/nodes/TriggerNodes.cpp`
**分类：** "Triggers"
**特点：** 无输入端口，仅输出，作为工作流起点。

| 节点 | ID | 参数 | 输出 | 说明 |
|------|----|------|------|------|
| Manual Trigger | `trigger.manual` | 无 | `output_main` | 手动触发，返回 `{triggered: true}` |
| Schedule | `trigger.schedule` | `cron` (Cron 表达式) | `output_main` | 定时触发 |
| Price Alert | `trigger.price_alert` | `symbol`, `condition` (above/below/crosses), `price` | `output_main` | 价格触警 |
| News Event | `trigger.news_event` | `keywords` (逗号分隔) | `output_main` | 新闻关键词触发 |
| Webhook | `trigger.webhook` | `path`, `method` (GET/POST/PUT) | `output_main` | HTTP Webhook 触发 |
| Market Hours Cron | `trigger.cron_market` | `cron`, `exchange`, `include_premarket` | `output_main` | 市场时段定时触发 |
| Portfolio Drift | `trigger.portfolio_drift` | `drift_threshold_pct`, `check_interval_min` | `output_main` | 组合偏离触发 |

### 4.2 Control Flow Nodes（控制流，6 种）

**文件：** `src/services/workflow/nodes/ControlFlowNodes.cpp`
**分类：** "Control Flow"

| 节点 | ID | 输入 | 输出 | 参数 | 说明 |
|------|----|------|------|------|------|
| If/Else | `control.if_else` | `input_0` | `output_true`, `output_false` | `condition` (表达式) | 条件分支，注解 `_branch` |
| Switch | `control.switch` | `input_0` | `output_0`, `output_1`, `output_2` | `field`, `case1`, `case2` | 多路路由，注解 `_route` |
| Loop | `control.loop` | `input_0` | `output_item`, `output_done` | `max_iterations` | 数组迭代 |
| Split | `control.split` | `input_0` | `output_0`, `output_1` | 无 | 数据复制到两路 |
| Merge | `control.merge` | `input_0`~`input_3` | `output_main` | `mode` (append/merge_by_key/keep_first) | 合并多路输入 |
| Wait | `control.wait` | `input_0` | `output_main` | `seconds` | 延时（QTimer） |

**Merge 模式详解：**

| 模式 | 行为 |
|------|------|
| `append` | 将所有输入合并为数组 |
| `merge_by_key` | 按 join_key 字段合并 |
| `keep_first` | 仅保留最先到达的输入 |

### 4.3 Market Data Nodes（市场数据，9 种）

**文件：** `src/services/workflow/nodes/MarketDataNodes.cpp`
**分类：** "Market Data"

| 节点 | ID | 参数 | 后端服务 |
|------|----|------|----------|
| Get Quote | `market.get_quote` | `symbol`, `source` (yahoo/alpha_vantage/polygon/databento) | MarketDataService → yfinance_data.py |
| Historical Data | `market.get_historical` | `symbol`, `period`, `interval` | MarketDataService → yfinance_data.py |
| Market Depth | `market.get_depth` | `symbol`, `depth` | Level 2 订单簿 |
| Ticker Stats | `market.get_stats` | `symbol` | 52 周高低、成交量、市值 |
| Fundamentals | `market.get_fundamentals` | `symbol`, `type` (overview/income/balance/cashflow/earnings) | 公司财务数据 |
| FRED Economics | `market.get_economics` | `preset/series_id`, `start_date`, `end_date`, `frequency`, `transform` | fred_data.py |
| Yield Curve | `market.get_yield_curve` | `start_date`, `end_date` | 美国国债收益率 |
| Options Chain | `market.get_options_chain` | `symbol`, `expiry`, `type` (calls/puts/both) | 期权链 + Greeks |
| Crypto Price | `market.get_crypto_price` | `symbol`, `quote`, `exchange` | 加密货币价格 |

### 4.4 Trading Nodes（交易，8 种）

**文件：** `src/services/workflow/nodes/TradingNodes.cpp`
**分类：** "Trading"
**特点：** 交易操作需要用户确认。

| 节点 | ID | 参数 | 说明 |
|------|----|------|------|
| Place Order | `trading.place_order` | `symbol`, `side`, `order_type`, `quantity`, `price`, `broker` | 下单（需确认） |
| Get Orders | `trading.get_orders` | `broker` | 查询订单 |
| Get Positions | `trading.get_positions` | `broker` | 查询持仓 |
| Get Holdings | `trading.get_holdings` | `broker` | 查询资产 |
| Get Balance | `trading.get_balance` | `broker` | 查询余额 |
| Bracket Order | `trading.bracket_order` | `symbol`, `side`, `quantity`, `entry_price`, `stop_loss`, `take_profit` | OCO 括号单 |
| Trailing Stop | `trading.trailing_stop` | `symbol`, `trail_type` (percent/fixed), `trail_value` | 追踪止损 |
| Scale In | `trading.scale_in` | `symbol`, `side`, `total_quantity`, `tranches`, `interval_sec` | 分批建仓 |

### 4.5 Analytics Nodes（分析，7 种）

**文件：** `src/services/workflow/nodes/AnalyticsNodes.cpp`
**分类：** "Analytics"

| 节点 | ID | 参数 | 后端 |
|------|----|------|------|
| Technical Indicators | `analytics.technical_indicators` | `indicator` (SMA/EMA/RSI/MACD/BBANDS/...), `period`, `symbol` | compute_technicals.py |
| Backtest | `analytics.backtest` | `start_date`, `end_date`, `initial_capital`, `commission` | 回测引擎 |
| Portfolio Optimization | `analytics.portfolio_optimization` | `method` (mean_variance/black_litterman/...), `risk_free_rate` | Python 优化 |
| Performance Metrics | `analytics.performance_metrics` | `benchmark`, `risk_free_rate` | Sharpe/Sortino/MaxDD |
| Correlation Matrix | `analytics.correlation_matrix` | `method` (pearson/spearman/kendall), `period` | 相关性矩阵 |
| Risk Ratios | `analytics.sharpe_ratio` | `ratio` type, `risk_free_rate`, `benchmark` | Sharpe/Sortino/Calmar |
| Monte Carlo | `analytics.monte_carlo` | `simulations`, `horizon_days`, `confidence_levels` | 蒙特卡洛模拟 |

### 4.6 Safety Nodes（安全，4 种）

**文件：** `src/services/workflow/nodes/SafetyNodes.cpp`
**分类：** "Safety"
**特点：** 作为安全闸门，检查通过才传递数据到下游。

| 节点 | ID | 参数 | 检查内容 |
|------|----|------|----------|
| Risk Check | `safety.risk_check` | `max_position_pct`, `max_volatility` | 仓位大小 + 波动率检查 |
| Loss Limit | `safety.loss_limit` | `daily_limit`, `weekly_limit` | 日/周亏损限额 |
| Trading Hours | `safety.trading_hours` | `exchange`, `allow_premarket` | 交易所开市时间 |
| Max Drawdown | `safety.max_drawdown_check` | `max_drawdown_pct`, `window` | 最大回撤熔断 |

**Safety Node 工作模式：**

```
Input → [Safety Check] → Pass → Output (数据透传)
                    └─ Fail → Output (空 + 错误信息)
                              → 下游节点收到失败通知
```

### 4.7 Agent Nodes（AI Agent，2 种）

**文件：** `src/services/workflow/nodes/AgentNodes.cpp`
**分类：** "Agents"

| 节点 | ID | 参数 | 说明 |
|------|----|------|------|
| AI Agent Runner | `agent.run` | `agent_id`, `llm_profile_id`, `extra_instructions`, `reasoning`, `memory`, `guardrails` | 执行配置好的 AI Agent |
| Tool Picker | `agent.tool_picker` | `query` | LLM 智能选择 MCP 工具，返回 `{tool, args}` |

**AI Agent Runner 执行流：**

```
1. 从 AgentConfigRepository 加载 Agent 配置
2. 解析 LLM Profile
3. 注入上游输入作为 context
4. 调用 AgentService::run_agent()
5. 异步回调 → 桥接到工作流回调
```

### 4.8 Notification Nodes（通知，6 种）

**文件：** `src/services/workflow/nodes/NotificationNodes.cpp`
**分类：** "Notifications"

| 节点 | ID | 参数 | 通道 |
|------|----|------|------|
| Email | `notify.email` | `message`, `title` | 邮件 |
| Slack | `notify.slack` | `message`, `title` | Slack Webhook |
| Discord | `notify.discord` | `message`, `title` | Discord Webhook |
| Telegram | `notify.telegram` | `message`, `title` | Telegram Bot |
| SMS | `notify.sms` | `message`, `title` | 短信 |
| Webhook | `notify.webhook` | `message`, `title`, `url` | 自定义 HTTP |

所有通知节点通过 `NotificationService` 分发。

### 4.9 File Nodes（文件，3 种）

**文件：** `src/services/workflow/nodes/FileNodes.cpp`
**分类：** "Files"

| 节点 | ID | 参数 | 说明 |
|------|----|------|------|
| File Operations | `file.operations` | `operation` (read/write/append/delete/exists), `path`, `encoding` | 文件读写 |
| Binary File | `file.binary` | `operation`, `path` | 二进制文件 |
| Convert to File | `file.convert` | `format` (json/csv/xlsx/xml/pdf), `path` | 格式转换导出 |

### 4.10 Data Format Nodes（数据格式，3 种）

**文件：** `src/services/workflow/nodes/DataFormatNodes.cpp`
**分类：** "Data Transform"

| 节点 | ID | 参数 | 说明 |
|------|----|------|------|
| JSON | `format.json` | `operation` (parse/stringify/query), JSONPath | JSON 操作 |
| CSV Parse | `format.csv_parse` | `delimiter`, `has_header`, `encoding` | CSV 解析（支持引号字段） |
| Regex Extract | `format.regex_extract` | `pattern`, `field`, `group`, `all_matches` | 正则提取 |

### 4.11 Utility Nodes（工具，15 种）

**文件：** `src/services/workflow/nodes/UtilityNodes.cpp`
**分类：** "Utilities" / "Data Transform"

| 节点 | ID | 参数 | 说明 |
|------|----|------|------|
| Date/Time | `utility.datetime` | `format`, `timezone` | 当前日期时间 |
| Filter | `transform.filter` | `field`, `operator`, `value` | 数组过滤 |
| Map | `transform.map` | `expression` | 数组变换 |
| Aggregate | `transform.aggregate` | `field`, `operation` (sum/avg/min/max/count) | 聚合计算 |
| Sort | `transform.sort` | `field`, `direction` (asc/desc) | 排序 |
| Join | `transform.join` | `join_key`, `join_type` (inner/left/right/outer) | SQL 风格连接 |
| GroupBy | `transform.group_by` | `field` | 分组 |
| Deduplicate | `transform.deduplicate` | `field` | 去重 |
| Limit | `utility.limit` | `count` | 截断数组 |
| Item Lists | `utility.item_lists` | `operation` (split/concat/flatten/unique/reverse/shuffle) | 列表操作 |
| RSS Read | `utility.rss_read` | `url`, `limit` | RSS/Atom 读取 |
| Crypto/Hash | `utility.crypto` | `operation` (sha256/md5/base64_encode/base64_decode/hmac_sha256) | 加密哈希 |
| Database | `utility.database` | `operation`, `query` | SQLite 查询 |
| Cache | `utility.cache_node` | `ttl_seconds`, `cache_key` | 输出缓存 |
| Template Render | `utility.template_render` | `template`, `output_key` | `{{var}}` 模板渲染 |

### 4.12 Integration Nodes（集成，3 种）

**文件：** `src/services/workflow/nodes/IntegrationNodes.cpp`
**分类：** "MCP" / "Utilities"

| 节点 | ID | 参数 | 说明 |
|------|----|------|------|
| MCP Tool Call | `mcp.tool_call` | `tool` (名称或 `serverId__toolName`) | 调用任意 MCP 工具 (237+) |
| Google Sheets | `utility.google_sheets` | `source`, `operation`, `file_id`, `sheet_name`, `range` | Google Sheets API |
| API Call | `utility.api_call` | `url`, `method`, `auth_type`, `auth_value`, `headers`, `body` | 通用 HTTP 客户端 |

---

## 五、ServiceBridges — 服务桥接

**文件：** `src/services/workflow/nodes/ServiceBridges.cpp`

ServiceBridges 将节点的 `execute` 函数连接到实际的后端服务：

```
Node execute callback
    │
    ▼
ServiceBridges (分发)
    │
    ├─ Market Data Nodes → MarketDataService → PythonWorker → yfinance_data.py
    │
    ├─ Trading Nodes → UnifiedTrading / PaperTrading (需用户确认)
    │
    ├─ Agent Nodes → AgentService → PythonRunner → finagent_core/main.py
    │
    ├─ Analytics Nodes → PythonRunner → compute_technicals.py / 优化脚本
    │
    ├─ Safety Nodes → 本地计算（仓位/波动率/时间检查）
    │
    ├─ Notification Nodes → NotificationService → 外部通知渠道
    │
    ├─ MCP Nodes → McpService → 本地 237+ 工具
    │
    └─ Utility Nodes → 本地计算（过滤/排序/哈希/模板等）
```

---

## 六、可视化编辑器

### 6.1 NodeEditorScreen — 主界面

**位置：** `src/screens/node_editor/NodeEditorScreen.h/cpp`
**接口：** 实现 `IStatefulScreen`（状态持久化）

```
┌─ NodeEditorToolbar ────────────────────────────────────────────────┐
│ [←] [Undo] [Redo] [Save] [Load] [Clear] [Import] [Export]        │
│ [▶ Execute]  [Deploy]  [Templates ▾]   工作流名称: ____________    │
├────────────┬──────────────────────────────────┬───────────────────┤
│  Palette   │         Canvas                    │  Properties       │
│            │                                   │                   │
│ ▼ Triggers │   [Manual]──→[Get Quote]──→[If]  │  Name: Get Quote  │
│  Manual    │                │         ╱     ╲  │  symbol: AAPL     │
│  Schedule  │                ▼       True   False│ source: yahoo    │
│  Price     │           [Alert Me]   [Log It]   │                   │
│  ...       │                                   │  [Delete Node]    │
│ ▼ Market   │                                   │                   │
│  Quote     │                                   │                   │
│  History   │                                   │                   │
│  ...       │                          [MiniMap]│                   │
├────────────┴──────────────────────────────────┴───────────────────┤
│  Execution Results (collapsible)                                    │
│  ✓ Manual Trigger    2ms    {triggered: true}                      │
│  ✓ Get Quote (AAPL)  234ms  {price: 150.25, change: 2.5}          │
│  ✓ If/Else           0ms    → True branch                          │
│  ✓ Alert Me          150ms  Sent via Slack                          │
└────────────────────────────────────────────────────────────────────┘
```

**交互功能：**

| 功能 | 操作 |
|------|------|
| 添加节点 | 从 Palette 拖拽到 Canvas |
| 连接端口 | 从输出端口拖到输入端口 |
| 删除节点 | 选中后按 Delete |
| 全选 | Ctrl+A |
| 复制/粘贴 | Ctrl+C / Ctrl+V |
| 撤销/重做 | Ctrl+Z / Ctrl+Y |
| 缩放 | 滚轮 (0.25x - 3x) |
| 平移 | 中键拖拽 / Ctrl+左键 |
| 框选 | 左键框选 |
| 右键菜单 | Canvas 右键快速添加节点 |
| 自动保存 | 每 30 秒 |

### 6.2 预置模板（15 个）

通过工具栏 Templates 下拉菜单加载：

| 类别 | 模板 |
|------|------|
| 价格监控 | 价格触警 → 技术分析 → 通知 |
| 新闻工作流 | 新闻触发 → AI 摘要 → 通知 |
| 交易策略 | 定时触发 → 数据获取 → 分析 → 安全检查 → 下单 |
| 组合再平衡 | 偏离触发 → 优化 → 风险检查 → 下单 → 通知 |
| ... | 共 15 个预置模板 |

### 6.3 执行可视化

- 执行中：边（Edge）动画显示数据流向
- 节点状态：等待（灰色）→ 执行中（蓝色脉冲）→ 成功（绿色）→ 失败（红色）
- 结果面板：可折叠，显示每个节点的耗时和输出

---

## 七、持久化层

### 7.1 数据库 Schema

**迁移：** `v008_workflows`

```sql
-- 工作流元数据
CREATE TABLE workflows (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL DEFAULT 'Untitled Workflow',
    description TEXT DEFAULT '',
    status TEXT NOT NULL DEFAULT 'draft',
    static_data TEXT DEFAULT '{}',
    created_at TEXT DEFAULT (datetime('now')),
    updated_at TEXT DEFAULT (datetime('now'))
);

-- 工作流节点
CREATE TABLE workflow_nodes (
    id TEXT PRIMARY KEY,
    workflow_id TEXT NOT NULL REFERENCES workflows(id) ON DELETE CASCADE,
    type TEXT NOT NULL,           -- 节点类型 ID
    name TEXT NOT NULL,           -- 用户自定义名称
    type_version INTEGER NOT NULL DEFAULT 1,
    pos_x REAL NOT NULL DEFAULT 0,
    pos_y REAL NOT NULL DEFAULT 0,
    parameters TEXT DEFAULT '{}',  -- JSON 参数
    credentials TEXT DEFAULT '{}', -- JSON 凭证
    disabled INTEGER NOT NULL DEFAULT 0,
    continue_on_fail INTEGER NOT NULL DEFAULT 0,
    retry_on_fail INTEGER NOT NULL DEFAULT 0,
    max_tries INTEGER NOT NULL DEFAULT 1,
    sort_order INTEGER NOT NULL DEFAULT 0
);

-- 工作流连线
CREATE TABLE workflow_edges (
    id TEXT PRIMARY KEY,
    workflow_id TEXT NOT NULL REFERENCES workflows(id) ON DELETE CASCADE,
    source_node TEXT NOT NULL,
    target_node TEXT NOT NULL,
    source_port TEXT NOT NULL,
    target_port TEXT NOT NULL,
    animated INTEGER NOT NULL DEFAULT 0
);
```

### 7.2 节点容错配置

每个节点可独立配置容错行为：

| 参数 | 类型 | 说明 |
|------|------|------|
| `disabled` | bool | 禁用节点（跳过执行） |
| `continue_on_fail` | bool | 失败后继续执行下游 |
| `retry_on_fail` | bool | 失败后自动重试 |
| `max_tries` | int | 最大重试次数 |

---

## 八、数据流详解

### 8.1 端到端执行示例：价格触警工作流

```
[Price Alert: AAPL > 150]
    │ {symbol: "AAPL", condition: "above", price: 150}
    ▼
[Get Quote: AAPL]
    │ {price: 152.30, change: 2.5}
    ▼
[If/Else: price > 150]
    │ condition = "={{$input.price}} > 150"
    │ → True: {_branch: true, price: 152.30, ...}
    ▼
[Safety: Risk Check]
    │ max_position_pct: 10%, max_volatility: 50
    │ → Pass
    ▼
[Trading: Place Order]
    │ symbol: AAPL, side: buy, quantity: 100
    │ → Order placed (需用户确认)
    ▼
[Notify: Slack]
    │ message: "Bought 100 AAPL @ $152.30"
    │ → Slack notification sent
```

### 8.2 并行执行示例

```
[Manual Trigger]
    │
    ├──→ [Get Quote: AAPL] ──→ [Analytics: RSI] ──┐
    │                                               │
    ├──→ [Get Quote: GOOGL] ──→ [Analytics: RSI] ──┤
    │                                               ▼
    └──→ [Get Quote: MSFT] ──→ [Analytics: RSI] ──[Merge]
                                                        │
                                                        ▼
                                                   [Portfolio Optimize]
                                                        │
                                                        ▼
                                                   [Notify: Email]
```

执行时序：
```
t=0ms    [Manual Trigger] ✓
t=0ms    [Get Quote: AAPL] [Get Quote: GOOGL] [Get Quote: MSFT]  ← 并行
t=250ms  三个 Quote 全部完成
t=250ms  [RSI: AAPL] [RSI: GOOGL] [RSI: MSFT]  ← 并行
t=500ms  三个 RSI 全部完成
t=500ms  [Merge] ✓
t=500ms  [Portfolio Optimize]
t=1200ms [Notify: Email] ✓
```

---

## 九、节点统计

| 分类 | 节点数 | 文件 |
|------|--------|------|
| Triggers（触发器） | 7 | TriggerNodes.cpp |
| Control Flow（控制流） | 6 | ControlFlowNodes.cpp |
| Market Data（市场数据） | 9 | MarketDataNodes.cpp |
| Trading（交易） | 8 | TradingNodes.cpp |
| Analytics（分析） | 7 | AnalyticsNodes.cpp |
| Safety（安全） | 4 | SafetyNodes.cpp |
| Agents（AI Agent） | 2 | AgentNodes.cpp |
| Notifications（通知） | 6 | NotificationNodes.cpp |
| Files（文件） | 3 | FileNodes.cpp |
| Data Format（数据格式） | 3 | DataFormatNodes.cpp |
| Utilities（工具） | 15 | UtilityNodes.cpp |
| Integration（集成） | 3 | IntegrationNodes.cpp |
| **合计** | **73** | 12 个文件 |

---

## 十、设计亮点

### 10.1 DAG 执行引擎

- **拓扑排序** 保证依赖顺序正确
- **环检测** 防止无限循环
- **分层并行** 最大化吞吐量
- **异步回调** 不阻塞 UI 线程

### 10.2 金融安全体系

- 4 种 Safety Node 组成安全闸门链
- 交易操作需用户显式确认
- 每个节点独立配置容错策略
- 最大回撤熔断机制

### 10.3 表达式引擎

- `={{...}}` 语法实现参数动态化
- 支持引用上游输出和节点参数
- 点号路径解析支持嵌套对象

### 10.4 可视化编程

- 拖拽式节点编辑器
- 15 个预置模板快速起步
- 执行过程实时可视化（边动画、节点状态）
- 30 秒自动保存 + 状态持久化

### 10.5 可扩展性

- 新节点类型只需注册 `NodeTypeDef`
- 每个 Node 的 execute 是独立的 `std::function`
- MCP Tool Call 节点直接访问 237+ 工具
- Agent 节点可调用任意配置的 AI Agent
