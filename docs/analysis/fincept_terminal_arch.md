# FinceptTerminal 总体架构分析

> 生成日期：2026-05-08
> 项目仓库：FinceptTerminal
> 技术栈：C++20 / Qt 6.8.3 / Python 3.x / CMake / Ninja

---

## 一、项目概览

FinceptTerminal 是一个原生桌面金融终端应用，定位对标 Bloomberg Terminal / Refinitiv Eikon，但以开源、可扩展为核心设计理念。项目采用 C++20/Qt6 作为主框架，嵌入 Python 运行时用于数据获取、分析和 AI 代理。编译产物为单一原生二进制文件，不依赖 Electron 或 Web 运行时。

### 核心特性

| 特性 | 说明 |
|------|------|
| 原生性能 | C++20 编译，无 Electron/Web 开销 |
| 多窗口停靠 | 基于 Advanced Docking System (ADS)，支持面板拖拽、撕裂、复制 |
| 实时数据 | DataHub 发布/订阅系统，支持 60+ 数据源 |
| AI 集成 | 内置 AI Agent 框架，支持多模型多代理协作 |
| Python 分析 | 嵌入 Python 运行时，集成 Qlib/QuantLib/Backtrader 等量化库 |
| 跨平台 | Windows / Linux / macOS 三平台支持 |
| 安全认证 | PBKDF2-SHA256 + AES-256-GCM + MFA 多层安全体系 |

---

## 二、整体架构

### 2.1 架构分层

```
┌──────────────────────────────────────────────────────────────────┐
│                        UI 层 (Screens)                          │
│  50+ 界面：Dashboard / Portfolio / Trading / Research / AI...   │
├──────────────────────────────────────────────────────────────────┤
│                     共享 UI 组件 (ui/)                           │
│  Widgets / Charts / Navigation / Theme / CommandPalette...      │
├──────────────────────────────────────────────────────────────────┤
│                      应用框架 (app/)                             │
│  TerminalShell → WindowFrame → ScreenRouter / DockScreenRouter  │
├──────────────────────────────────────────────────────────────────┤
│              DataHub — 发布/订阅数据中枢                          │
│  Topic 路由 / Producer-Subscriber / 缓存策略 / 调度引擎          │
├──────────────────────────────────────────────────────────────────┤
│                    业务服务层 (services/)                        │
│  60+ 服务模块：MarketData / News / Portfolio / Agent / Crypto... │
├──────────┬───────────┬──────────────┬───────────────────────────┤
│ Python   │   MCP     │    Auth      │      Storage              │
│ Bridge   │   Tools   │   Security   │   SQLite / Cache          │
│ Runner   │  237+工具  │  MFA/PIN    │   Migration               │
│ Worker   │  JSON-RPC │  AES-256     │   Workspace               │
└──────────┴───────────┴──────────────┴───────────────────────────┘
```

### 2.2 源码目录结构

```
fincept-qt/src/
├── app/          # 应用入口、窗口管理、路由
├── core/         # 核心工具：配置、日志、事件总线、动作系统
├── datahub/      # 发布/订阅数据中枢
├── services/     # 60+ 业务服务模块
├── screens/      # 50+ UI 界面
├── python/       # Python 桥接层
├── ui/           # 共享 UI 组件和主题系统
├── mcp/          # Model Context Protocol 工具集成
├── auth/         # 认证与安全
└── storage/      # 数据持久化
```

### 2.3 数据流架构

```
                    ┌─────────────┐
                    │ Python 脚本  │
                    │ 外部 API    │
                    └──────┬──────┘
                           │ JSON/HTTP
                    ┌──────▼──────┐
                    │   Service   │ (Producer)
                    │  业务服务层   │
                    └──────┬──────┘
                           │ publish()
                    ┌──────▼──────┐
                    │   DataHub   │ ← Topic 路由 / 缓存 / 调度
                    │  发布/订阅   │
                    └──────┬──────┘
                           │ subscribe() 回调
                    ┌──────▼──────┐
                    │   Screen    │ (Subscriber)
                    │   UI 界面    │
                    └─────────────┘
```

---

## 三、核心模块详解

### 3.1 应用框架层 (`app/`)

应用框架层负责整个应用的生命周期管理、窗口系统和界面路由。

| 类 | 职责 | 设计模式 |
|----|------|----------|
| `TerminalShell` | 进程级单例，协调所有子系统初始化和关闭 | Singleton |
| `WindowFrame` | 主窗口容器，管理停靠面板、主题、布局 | Facade |
| `ScreenRouter` | 基础界面路由 (QStackedWidget) | Factory |
| `DockScreenRouter` | ADS 感知的高级路由，支持面板撕裂/复制/平铺 | Factory + Command |
| `InstanceLock` | 单实例锁 + 跨实例 IPC 通信 | Singleton + IPC |

**应用启动序列：**

1. Profile 解析和路径设置
2. Crash handler 安装
3. TerminalShell 初始化
4. DataHub 元类型注册
5. Producer 注册（服务注册 Topic 模式）
6. 数据库设置和迁移
7. 认证引导
8. MCP 工具初始化
9. Python 环境检查/安装
10. 窗口创建

**窗口管理系统特性：**
- 多显示器支持（`move_to_screen`）
- 布局持久化（`capture_layout` / `apply_layout`）
- 专注模式（隐藏 Chrome）
- 面板撕裂到新窗口
- 面板复制（同一数据源，多个视图）
- 2x2 平铺排列
- 符号组联动（Symbol Group Linking）

### 3.2 DataHub — 数据中枢 (`datahub/`)

DataHub 是整个应用的数据核心，实现了发布/订阅模式，是所有数据流的单一真相源。

#### Topic 命名规范

```
格式：domain:subdomain:id[:modifier]
示例：market:quote:AAPL
      market:history:AAPL:daily
      news:symbol:TSLA
      option:chain:SPY
通配符：market:quote:* （仅订阅端使用）
```

#### 核心接口

**Producer（数据生产者）：**
```cpp
class Producer {
    QStringList topic_patterns() const;  // 声明拥有的 topic
    void refresh(const QStringList& topics);  // 获取并 publish()
};
```

**Subscriber（数据消费者）：**
```cpp
DataHub::instance().subscribe(widget, "market:quote:AAPL",
    [](const QVariant& v) { update_ui(v.value<QuoteData>()); });
```

#### TopicPolicy 配置

| 策略 | 说明 |
|------|------|
| TTL | 缓存数据存活时间 |
| min_interval | 最小刷新间隔（限流） |
| refresh_timeout | 刷新超时 |
| push_only | WebSocket 驱动的 topic，不主动拉取 |
| coalesce | 请求合并（避免重复请求） |
| drop_on_idle | 界面不可见时丢弃数据 |
| pause_inactive | 非活跃时暂停刷新 |

#### DataHub 纪律规则（CI 强制）

| 规则 | 说明 |
|------|------|
| D1 | Screen 禁止直接调用 PythonRunner，必须通过 DataHub Producer |
| D2 | 获取外部数据的服务必须实现 Producer 并通过 DataHub 发布 |
| D3 | 禁止在 Widget 中用 QTimer 做数据刷新，由 Hub 调度器控制 |
| D4 | Screen 禁止直接调用 service 的 fetch_* 回调，必须用 subscribe/peek |
| D5 | push_only topic 必须在 Producer 构造时声明策略 |

#### 已注册的元类型

| 类型 | 用途 |
|------|------|
| `QuoteData` | 实时行情数据 |
| `HistoryPoint` | 历史 OHLCV 数据点 |
| `InfoData` | 公司基本信息 |
| `NewsArticle` | 新闻文章 |
| `EconomicsResult` | 经济指标数据 |

### 3.3 业务服务层 (`services/`)

服务层包含 60+ 服务模块，按领域组织。

#### 3.3.1 AI 与机器学习

| 服务 | 说明 | Producer |
|------|------|----------|
| `AgentService` | AI Agent 中央服务，支持流式/结构化/多查询执行，内置股票分析、组合调仓、风险评估等工作流 | ✅ `agent:output:*` |
| `AIQuantLabService` | 24 模块量化研究平台（QLib, GS Quant, Functime 等），覆盖强化学习交易、组合优化、因子挖掘、HFT | ❌ |
| `QuantLibClient` | QuantLib REST API HTTP 客户端 | ❌ |

#### 3.3.2 市场数据

| 服务 | 说明 | Producer |
|------|------|----------|
| `MarketDataService` | 批量行情、公司信息、历史数据、迷你图 | ✅ `market:quote:*`, `market:history:*`, `market:sparkline:*` |
| `DatabentoService` | 机构级市场数据（波动率曲面、期货、期权） | ❌ |
| `AsiaMarketsService` | 亚洲市场数据浏览与查询 | ❌ |
| `AkShareService` | 中国市场数据 AkShare 连接器 | ❌ |
| `SectorResolver` | 代码→行业映射，缓存 + yfinance | ❌ |

#### 3.3.3 新闻与信息

| 服务 | 说明 | Producer |
|------|------|----------|
| `NewsService` | RSS 聚合 + AI 摘要 + 情感分析 + 市场影响评估 + WebSocket 实时推送 | ✅ `news:general`, `news:symbol:*`, `news:category:*` |
| `ForumService` | Fincept 社区论坛集成（帖子、投票、评论） | ❌ |
| `GeopoliticsService` | 地缘政治事件追踪、HDX 人道主义数据、贸易分析 | ✅ `geopolitics:*` |
| `MaritimeService` | 船舶追踪、海域监控、AIS 数据 | ✅ `maritime:*` |

#### 3.3.4 交易与组合

| 服务 | 说明 | Producer |
|------|------|----------|
| `PortfolioService` | 组合 CRUD、资产/交易管理、绩效指标、相关性分析、基准对比 | ❌ |
| `OptionChainService` | 期权链组装、Greeks 计算、Max Pain / PCR | ✅ `option:chain:*`, `option:atm_iv:*` |
| `AlgoTradingService` | 策略 CRUD、部署生命周期、回测执行、市场扫描 | ❌ |
| `BacktestingService` | 多提供商回测（LEAN, VectorBT, FastTrade 等） | ❌ |

#### 3.3.5 预测市场与加密

| 服务 | 说明 | Producer |
|------|------|----------|
| `PolymarketService` | Polymarket 三 API 集成（Gamma/CLOB/Data） | ❌ |
| `Prediction Exchange Registry` | 统一预测市场交易接口（Kalshi/Polymarket/内部） | ❌ |
| `WalletService` | 加密钱包连接、余额追踪、交易签名、PumpFun swap | ✅ balance/price/activity |
| `TotpService` | TOTP 时间验证码生成 | ❌ |

#### 3.3.6 经济与政府数据

| 服务 | 说明 | Producer |
|------|------|----------|
| `EconomicsService` | 经济数据调度器 | ✅ `econ:*` |
| `GovDataService` | 政府数据调度器 | ✅ `govdata:*` |
| `DBnomicsService` | DBnomics 经济数据库（提供者/数据集/序列/观测值） | ✅ `dbnomics:*` |

#### 3.3.7 分析与研究

| 服务 | 说明 | Producer |
|------|------|----------|
| `EquityResearchService` | 股票研究：搜索、财报、技术分析、同行对比、TALIpp | ❌ |
| `MAAnalyticsService` | 并购分析：DCF/LBO/Comps 估值、公平意见 | ✅ `ma:*` |
| `RelationshipMapService` | 企业关系图谱 | ✅ `geopolitics:relationship_graph:*` |

#### 3.3.8 基础设施与工具

| 服务 | 说明 |
|------|------|
| `NotificationService` | 通知分发（20+ 提供商：Discord/Email/Slack/Telegram 等） |
| `FileManagerService` | 文件存储和索引 |
| `PushpinService` | 持久化代码管理 |
| `WorkflowService` | 工作流持久化和执行 |
| `ReportBuilderService` | 报告文档管理（组件 CRUD、撤销/重做） |
| `UpdateService` | 应用自动更新（GitHub Releases） |
| `PythonCliService` | 通用 Python 脚本包装器 |

#### 3.3.9 语音与交互

| 服务 | 说明 |
|------|------|
| `SpeechService` | 语音转文字（Google STT / Deepgram） |
| `TtsService` | 文字转语音（pyttsx3 / Deepgram） |
| `ClapDetectorService` | 背景拍手检测 |

#### 3.3.10 计费与用户

| 服务 | 说明 | Producer |
|------|------|----------|
| `FeeDiscountService` | 手续费折扣计算 | ✅ `billing:fncpt_discount:*` |
| `TierService` | 用户等级管理 | ❌ |

### 3.4 UI 界面层 (`screens/`)

共 47 个子目录，50+ 界面类，按领域组织。

#### 3.4.1 核心交易与组合

| 界面 | 功能 |
|------|------|
| `DashboardScreen` | 主工作区：工具栏、Ticker Bar、可拖拽 Widget 网格、市场脉动、状态栏 |
| `PortfolioScreen` | 多组合管理：热力图、绩效图表、行业分析、交易面板 |
| `MarketsScreen` | 市场数据终端：三栏可调面板 |
| `EquityTradingScreen` | 股票交易：Watchlist + 图表 + 下单 + 订单簿 + Broker 集成 |
| `CryptoTradingScreen` | 加密交易：多交易所、模拟盘、杠杆设置 |

#### 3.4.2 研究与分析

| 界面 | 功能 |
|------|------|
| `AIQuantLabScreen` | 24 模块量化研究：市场/技术/统计/ML 四大类 |
| `EquityResearchScreen` | 股票深度研究：8 标签页（概览/财报/分析/技术/Talipp/同行/新闻/情感） |
| `BacktestingScreen` | 多引擎回测：6 提供商、50+ 策略、策略优化、Walk-Forward |
| `SurfaceAnalyticsScreen` | 35 种金融曲面可视化：权益衍生品/固收/FX/信用/商品/风险/宏观 |
| `TradeVizScreen` | 双边贸易流可视化（Chord Diagram） |

#### 3.4.3 算法交易

| 界面 | 功能 |
|------|------|
| `AlgoTradingScreen` | 策略构建器 + 策略列表 + 扫描器 + 部署仪表板 |
| `AlphaArenaScreen` | 算法竞赛平台：排行榜、实时模式、模型聊天 |
| `CryptoCenterScreen` | 加密钱包中心：Home/Trade/Activity/Settings/Stake/Markets |

#### 3.4.4 市场数据与新闻

| 界面 | 功能 |
|------|------|
| `WatchlistScreen` | 全屏 Watchlist：实时报价、搜索、符号管理、组联动 |
| `AkShareScreen` | 中国市场数据集成 |

#### 3.4.5 工具与开发

| 界面 | 功能 |
|------|------|
| `NotesScreen` | 金融笔记：三栏布局、分类、搜索、收藏、Markdown |
| `CodeEditorScreen` | Python/Jupyter 编辑器：Cell 编辑、Markdown 渲染 |
| `ReportBuilderScreen` | 金融报告创建：组件工具栏 + 文档画布 + 属性面板 |
| `ChatModeScreen` | 全屏 AI 聊天：会话面板 + 消息面板 + Agent 面板 + TerminalToolBridge |
| `NodeEditorScreen` | 可视化编程（占位） |

#### 3.4.6 配置与设置

| 界面 | 功能 |
|------|------|
| `SettingsScreen` | 12 个设置板块：凭证/外观/通知/存储/数据源/LLM/MCP/日志/安全/快捷键/Python/开发者 |
| `AgentConfigScreen` | AI Agent 管理：Agent/Teams/Workflows/Tools/Planner/System 6 视图 |

#### 3.4.7 用户与认证

| 界面 | 功能 |
|------|------|
| `LoginScreen` | 邮箱/密码登录 + MFA |
| `RegisterScreen` | 用户注册 |
| `PricingScreen` | 订阅计划 |
| `ProfileScreen` | 用户资料（5 区块：概览/使用量/安全/账单/支持） |
| `SetupScreen` | 首次运行 Python 环境安装向导 |

#### 3.4.8 开发者工具

| 界面 | 功能 |
|------|------|
| `DataHubInspector` | 实时 DataHub 统计查看器：Topic 监控、订阅者数量、策略查看 |
| `LaunchpadScreen` | 无窗口时的最小化门户：新建窗口/切换 Profile/打开布局 |

#### 3.4.9 关键 UI 接口

**IStatefulScreen（状态持久化）：**
- `restore_state()` / `save_state()` / `state_key()` / `state_version()`
- 实现者：Portfolio, Watchlist, Notes, CryptoTrading, AIQuantLab, Backtesting, CodeEditor, ReportBuilder, SurfaceAnalytics

**IGroupLinked（符号组联动）：**
- `set_group()` / `on_group_symbol_changed()` / `current_symbol()`
- 实现者：Portfolio, Watchlist, EquityTrading, CryptoTrading, EquityResearch, SurfaceAnalytics
- 功能：切换一个界面的符号，其他联动界面自动跟随

### 3.5 共享 UI 组件 (`ui/`)

| 子目录 | 组件 | 说明 |
|--------|------|------|
| `widgets/` | Card, SearchBar, SpeedSparkline, WorldMapWidget, StatusBadge, NotifToast, ConfettiOverlay 等 | 原子级 UI 组件 |
| `navigation/` | CommandBar, NavigationBar, FKeyBar, ToolBar, StatusBar, DockToolBar, DockStatusBar | 导航与工具栏 |
| `command/` | CommandPalette (Ctrl+K), CommandParser, QuickCommandBar, SuggestionIndex | 命令系统（模糊搜索） |
| `charts/` | ChartFactory | 图表工厂 |
| `markdown/` | MarkdownRenderer | Obsidian 风格 Markdown → HTML |
| `tables/` | DataTable | 可复用数据表格 |
| `pushpins/` | PushpinBar, SymbolChip | 快速 Pin 面板（拖拽） |
| `workspace/` | LayoutOpenDialog, LayoutSaveAsDialog | 工作区布局管理 |
| `components/` | ComponentBrowserDialog, ComponentCard | 组件浏览器（热度排序） |
| `theme/` | Theme, ThemeManager, ThemeTokens, StyleSheets | 主题系统（Design Tokens） |
| `error/` | ErrorPipeline | 错误路由和展示 |
| `debug/` | DebugOverlay | 调试信息叠加层 |
| `notifications/` | NotificationService | 外部通知分发 |

### 3.6 Python 桥接层 (`python/`)

| 组件 | 说明 | 设计模式 |
|------|------|----------|
| `PythonSetupManager` | 首次运行 Python 环境引导（UV 包管理、Hash 校验、进度报告） | State Machine |
| `PythonRunner` | 一次性脚本执行（并发限制 3 个、流式输出、JSON 结果提取） | Command + Pool |
| `PythonWorker` | 持久化守护进程（长连接 Python 进程、二进制协议通信、自动重启） | Worker |
| `OptionGreeksWorker` | 期权计算专用 Worker | Specialized Worker |

**通信机制：**
- PythonRunner: JSON over QProcess（一次性）
- PythonWorker: 二进制协议 over QProcess（持久化）
- 环境变量传递配置信息

### 3.7 MCP 工具集成 (`mcp/`)

Model Context Protocol 是 LLM 工具调用的标准化协议。

| 组件 | 说明 |
|------|------|
| `McpService` | 统一接口：内部工具 + 外部工具 |
| `McpProvider` | 内部工具注册表和执行器（237+ 内置工具） |
| `McpManager` | 外部 MCP 服务器生命周期管理 |
| `McpClient` | JSON-RPC 2.0 over stdio 与外部服务器通信 |
| `SchemaValidator` | JSON Schema 输入验证 |
| `ToolSchemaBuilder` | 流式 API 声明工具 Schema |
| `ToolRetriever` | BM25 语义搜索工具目录 |
| `TerminalMcpBridge` | HTTP 桥接，供 Python Agent 调用 |

**工具授权体系：5 级认证**

```
Public → Authenticated → Premium → Admin → System
```

### 3.8 认证与安全 (`auth/`)

| 组件 | 说明 |
|------|------|
| `AuthManager` | 认证状态机 + Session 管理 |
| `AuthApi` | 无状态 API 调用（登录/注册/刷新） |
| `AuthTypes` | 登录/Session 数据类型 |
| `PinManager` | 本地 PIN 存储（PBKDF2-SHA256，600k 迭代） |
| `InactivityGuard` | 空闲超时 + 屏幕锁定 |
| `SessionGuard` | Session 轮询验证（30s 间隔） |
| `SecurityAuditLog` | PIN/锁屏安全事件日志 |
| `UserApi` | 用户资料/订阅/支付处理 |

**安全机制：**
- PBKDF2-SHA256 哈希（本地 PIN）
- AES-256-GCM 加密存储（凭证）
- Session + PIN + MFA 多层认证
- 可配置空闲锁定超时

### 3.9 数据持久化 (`storage/`)

| 组件 | 说明 |
|------|------|
| `Database` | SQLite 封装（WAL 模式） |
| `StorageManager` | 中央数据管理和清理 |
| `SecureStorage` | AES-256-GCM 加密存储 |
| `CacheManager` | SQLite 缓存 + TTL |
| `TabSessionStore` | Tab 会话持久化 |
| `BaseRepository` | 模板基类（Repository 模式） |
| 20+ Repository | AccountRepository, ChatRepository, PortfolioRepository, SettingsRepository... |
| `WorkspaceDb` | 工作区数据库 |
| `CrashRecovery` | 应用状态恢复 |
| `WorkspaceSnapshotRing` | 工作区快照环形缓冲 |
| `MigrationRunner` | 数据库 Schema 迁移系统 |

---

## 四、Python 脚本生态 (`scripts/`)

### 4.1 数据源（60+ 提供商）

| 领域 | 提供商 |
|------|--------|
| 市场数据 | Yahoo Finance, Alpha Vantage, TradingView, Databento |
| 经济数据 | FRED, World Bank, IMF, OECD, BEA, BLS |
| 加密货币 | CoinGecko, Kraken, Binance |
| 政府数据 | SEC, Congress.gov, Federal Reserve |
| 国际数据 | AkShare (中国), Eurostat, 各国 data.gov |

### 4.2 分析模块

| 模块 | 功能 |
|------|------|
| 权益估值 | DCF, DDM, 乘数估值 |
| 组合管理 | 优化、风险管理 |
| 衍生品 | 期权定价、Greeks |
| 经济分析 | 增长模型、政策分析 |
| 量化分析 | CFA 模型、技术分析 |

### 4.3 AI Agent

| Agent | 说明 |
|-------|------|
| Agno Trading | 多代理交易系统 |
| Investor Personas | 巴菲特/格雷厄姆投资策略 |
| Hedge Fund Agents | Bridgewater/Citadel 框架 |
| Geopolitical | 大棋局分析 |

### 4.4 专用系统

| 系统 | 功能 |
|------|------|
| AI Quant Lab | Qlib 集成 + RDAgent |
| Backtesting | LEAN Engine, VectorBT, FastTrade |
| Satellite Data | NASA, ESA 地球观测 |

---

## 五、构建系统

### 5.1 CMake 配置

| 配置 | 说明 |
|------|------|
| 标准 | C++20 |
| 最低编译器 | MSVC 19.38+ / GCC 12.3+ / Clang 15.0+ |
| Qt 版本 | 6.8.3（固定） |
| 构建系统 | Ninja |
| PCH | 启用 |
| CCache | 支持 |

### 5.2 预设

| 预设 | 平台 |
|------|------|
| `win-debug` / `win-release` / `win-fast` / `win-release-lto` | Windows |
| `linux-debug` / `linux-release` | Linux |
| `macos-debug` / `macos-release` | macOS |

### 5.3 Qt 模块依赖

| 模块 | 用途 |
|------|------|
| Core / Widgets | 基础 UI 框架 |
| Charts | 图表渲染 |
| Network | HTTP / WebSocket |
| Sql | SQLite 数据库 |
| Concurrent | 多线程 |
| Multimedia | 音频输入/输出 |
| PrintSupport | 打印支持 |
| WebSockets（可选） | WebSocket 通信 |
| TextToSpeech（可选） | TTS |
| WaylandClient（可选） | Linux Wayland |

### 5.4 代码质量工具

| 工具 | 配置 |
|------|------|
| clang-format | LLVM 风格，K&R 大括号，4 空格缩进，120 列宽 |
| clang-tidy | bugprone-*, modernize-*, performance-*, readability-* |
| cppcheck | warning, performance, portability |

---

## 六、CI/CD 工作流

| 工作流 | 触发 | 功能 |
|--------|------|------|
| `lint.yml` | 手动触发 | clang-format / clang-tidy / cppcheck / DataHub 纪律检查 |
| `build-cpp.yml` | Push/PR | 多平台构建（Windows x64/arm64, Linux, macOS） |
| `release.yml` | Tag | 安装包生成和发布 |
| `pr-gate.yml` | PR | 代码质量门禁 |

---

## 七、设计模式总结

| 模式 | 应用场景 |
|------|----------|
| **Singleton** | DataHub, TerminalShell, Logger 等全局服务 |
| **Publisher-Subscriber** | DataHub Topic 系统 |
| **Observer** | Qt Signal/Slot, Symbol Group 变更通知 |
| **Factory** | 延迟加载界面、图表创建 |
| **Strategy** | 主题切换、遥测后端、TTS/STT 提供商 |
| **Command** | Action 系统、键盘快捷键 |
| **Registry** | Action/Panel/Window/Symbol 注册表 |
| **Repository** | 数据访问层（20+ Repository） |
| **Facade** | WindowFrame 封装 ADS 复杂性 |
| **State Machine** | PythonSetupManager 安装阶段 |
| **Worker** | PythonWorker 持久化进程 |
| **Policy** | TopicPolicy 配置 topic 行为 |

---

## 八、架构亮点与评估

### 优势

1. **DataHub 解耦**：所有数据流经 DataHub，界面与服务完全解耦，新增数据源或界面零耦合改动
2. **原生性能**：C++20 编译，无 Electron 开销，适合高频数据刷新场景
3. **Python 生态融合**：通过 PythonRunner/Worker 桥接，可复用全部 Python 数据科学生态
4. **可扩展的 MCP 工具系统**：237+ 内置工具 + 外部服务器支持，BM25 语义搜索发现工具
5. **多窗口停靠系统**：ADS 提供专业级面板管理，支持撕裂/复制/平铺/跨显示器
6. **企业级安全**：PBKDF2 + AES-256-GCM + MFA + 空闲锁定 + 安全审计日志
7. **状态持久化**：IStatefulScreen 接口保证界面状态跨会话保留
8. **符号组联动**：IGroupLinked 实现跨界面符号同步，专业终端核心功能

### 潜在关注点

1. **服务数量庞大**：60+ 服务模块的维护成本和测试覆盖需要持续关注
2. **Python 依赖管理**：大量 Python 脚本的依赖版本管理需要规范化
3. **占位界面较多**：47 个 screen 目录中约 20 个仍为占位状态（ComingSoonScreen）

---

## 九、功能点汇总

| 领域 | 功能 |
|------|------|
| **行情数据** | 实时报价、历史数据、迷你图、市场分类、60+ 数据源 |
| **股票交易** | 多账户、Watchlist、下单、订单簿、Broker 集成 |
| **加密交易** | 多交易所、模拟盘、杠杆、钱包管理、Swap |
| **期权交易** | 期权链、Greeks、IV 曲面、Max Pain、PCR |
| **组合管理** | 多组合、热力图、绩效归因、相关性分析、基准对比 |
| **算法交易** | 策略构建、部署、扫描、回测 |
| **AI Agent** | 多代理协作、流式推理、结构化输出、团队/工作流 |
| **量化研究** | 24 模块 AI Quant Lab、强化学习、因子挖掘、HFT |
| **回测** | 6 引擎、50+ 策略、Walk-Forward、策略优化 |
| **新闻** | RSS 聚合、AI 摘要、情感分析、市场影响评估、实时推送 |
| **地缘政治** | 冲突监控、人道主义数据、贸易分析、关系图谱 |
| **海事追踪** | 船舶搜索、位置追踪、历史轨迹 |
| **经济数据** | FRED/IMF/World Bank/OECD 等 20+ 经济数据库 |
| **政府数据** | SEC/Congress/Fed 等 10+ 政府数据源 |
| **预测市场** | Polymarket/Kalshi/内部市场、订单簿、价格历史 |
| **分析工具** | DCF/DDM/Comps 估值、并购分析、技术分析 |
| **可视化** | 35 种金融曲面、贸易流 Chord Diagram、世界地图 |
| **协作** | 社区论坛、报告构建器、AI 聊天 |
| **语音交互** | STT/TTS、拍手检测、多提供商 |
| **通知系统** | 20+ 通知渠道（Discord/Slack/Telegram/Email 等） |
| **安全** | MFA/PIN/AES-256 加密/空闲锁定/安全审计 |
| **工作区** | 多窗口、可停靠面板、布局模板、跨显示器、状态持久化 |
| **命令系统** | Ctrl+K 命令面板、Slash 命令、F-Key 快捷键 |
| **开发工具** | DataHub Inspector、代码编辑器、Node Editor |
| **自动更新** | GitHub Releases 检测和安装 |
