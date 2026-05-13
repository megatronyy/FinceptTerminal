# FinceptTerminal Python 模块架构分析

> 生成日期：2026-05-08
> 脚本总数：1,411 个 Python 文件
> Python 版本：3.11.9
> 包管理器：UV 0.7.12

---

## 一、总体架构

Python 模块在 FinceptTerminal 中承担三大角色：**数据获取**、**分析计算**、**AI Agent**。C++ 主程序通过桥接层（`src/python/`）启动和管理 Python 进程，采用 JSON 作为统一通信格式。

### 1.1 C++ ↔ Python 通信架构

```
┌─────────────────────────────────────────────────────────────┐
│                    C++ 主程序 (Qt6)                          │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │ PythonRunner  │  │ PythonWorker │  │OptionGreeksWorker │  │
│  │ (一次性脚本)  │  │ (持久守护进程)│  │ (期权计算守护进程) │  │
│  └──────┬───────┘  └──────┬───────┘  └────────┬──────────┘  │
│         │                 │                    │              │
│    JSON over stdout   二进制协议 over stdin/stdout           │
│    (命令行参数)       (4字节长度前缀 + JSON)                  │
└─────────┼─────────────────┼────────────────────┼────────────┘
          │                 │                    │
┌─────────▼─────────────────▼────────────────────▼────────────┐
│                    Python 运行时                              │
│                                                             │
│  ┌──────────┐  ┌────────────┐  ┌────────────┐  ┌────────┐  │
│  │ 数据获取  │  │  分析计算   │  │  AI Agent  │  │  交易  │  │
│  │ 60+数据源 │  │ 量化/估值   │  │ 30+ 代理   │  │ 交易所 │  │
│  └──────────┘  └────────────┘  └────────────┘  └────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 两种执行模式

| 模式 | 组件 | 通信协议 | 适用场景 | 启动开销 |
|------|------|----------|----------|----------|
| **一次性脚本** | `PythonRunner` | 命令行参数 → stdout JSON | 低频调用、复杂计算 | 每次启动新进程 |
| **持久守护进程** | `PythonWorker` / `OptionGreeksWorker` | 二进制帧协议 (4B len + JSON) | 高频调用（行情数据、期权计算） | 一次启动，复用进程 |

---

## 二、C++ 桥接层 (`src/python/`)

### 2.1 PythonRunner — 一次性脚本执行器

**职责：** 将 Python 脚本作为子进程执行，捕获 JSON 输出。

```cpp
class PythonRunner : public QObject {
    // 核心接口
    void run(const QString& script, const QStringList& args,
             Callback cb, StreamCallback on_line = {});
    void run_code(const QString& code, Callback cb);

    // 环境管理
    QString python_path() const;
    QString scripts_dir() const;
    QProcessEnvironment build_python_env() const;
    bool is_available() const;
    void set_max_concurrent(int n);  // 默认 3
};
```

**工作流程：**

```
C++ 调用 run("yfinance_data", ["get_quote", "AAPL"], callback)
  │
  ├─ 1. 检查并发槽位（默认最多 3 个并发进程）
  │     └─ 满了则入队等待
  ├─ 2. 构建进程环境
  │     ├─ 选择正确的 venv（numpy1 vs numpy2）
  │     ├─ 注入环境变量（API keys、路径）
  │     └─ 大参数 (>8KB) 溢出到临时文件
  ├─ 3. 启动 QProcess
  │     └─ python <script_path> <args...>
  ├─ 4. 捕获 stdout/stderr
  │     └─ 支持流式行回调 (StreamCallback)
  └─ 5. 进程结束后解析 JSON
        └─ extract_json() 从输出中提取 JSON 块
```

**关键特性：**
- 并发限制：默认 3 个，通过 `set_max_concurrent()` 可调
- 大参数处理：超过 8KB 的参数自动溢出到临时文件（避免 Windows 命令行长度限制）
- JSON 提取：`extract_json()` 从混合输出中智能提取 JSON 块
- 环境隔离：仅注入白名单凭证变量，不透传 shell 环境中的敏感信息

### 2.2 PythonWorker — 持久守护进程

**职责：** 维持一个长期运行的 Python 进程，避免每次请求的 2-3 秒 import 开销。主要用于 `yfinance_data.py`。

```cpp
class PythonWorker : public QObject {
    bool is_ready() const;
    void submit(const QString& action, const QJsonObject& payload, Callback cb);
    void stop();
};
```

**二进制帧协议：**

```
请求/响应帧格式：
┌───────────────────────────────────────┐
│  4 字节 (big-endian)  │  N 字节       │
│  JSON body 长度        │  UTF-8 JSON   │
└───────────────────────────────────────┘

握手：守护进程启动后发送 {"ready": true, "pid": 12345}
请求：{"id": 1, "action": "batch_all", "payload": {...}}
响应：{"id": 1, "ok": true, "result": {...}}
```

**请求 JSON 格式：**

```json
{
    "id": 1,
    "action": "batch_all",
    "payload": {
        "quotes": ["AAPL", "GOOGL"],
        "sparklines": ["TSLA"],
        "histories": [{"symbol": "MSFT", "period": "1y", "interval": "1d"}]
    }
}
```

**响应 JSON 格式：**

```json
{
    "id": 1,
    "ok": true,
    "result": {
        "quotes": [{"symbol": "AAPL", "price": 150.25, ...}],
        "sparklines": [...],
        "histories": [...]
    }
}
```

**进程管理：**
- 懒启动：首次 `submit()` 时才启动进程
- 自动重启：进程退出后自动重启，最多 5 次
- 就绪超时：15 秒 watchdog，超时则重启
- 请求队列：守护进程未就绪时请求排队等待
- 崩溃恢复：崩溃时 fail 所有 in-flight 请求，保留 queued 请求等待重启

**支持的动作 (`action`)：**
- `batch_all` — 批量获取行情、迷你图、历史数据
- `quote` / `sparkline` / `history` — 单独获取各类数据

### 2.3 OptionGreeksWorker — 期权计算守护进程

**职责：** 专门的持久守护进程，用于期权 Greeks 计算（py_vollib + scipy）。

```cpp
class OptionGreeksWorker : public QObject {
    void submit(const QString& action, const QJsonObject& payload, Callback cb);
};
```

**与 PythonWorker 的区别：**
- 使用 venv-numpy2（包含 py_vollib + scipy）
- 就绪超时更长：20 秒（因 scipy 导入较慢）
- 仅支持 `option_greeks_batch` 动作
- 运行 `option_greeks_daemon.py` 脚本

### 2.4 PythonSetupManager — 环境安装管理器

**职责：** 首次运行时自动安装 Python 环境和依赖。

```cpp
class PythonSetupManager : public QObject {
    SetupStatus check_status() const;
    void run_setup();
    QString python_path(const QString& venv_name = "venv-numpy2") const;
    QString uv_path() const;

signals:
    void progress_changed(const SetupProgress& progress);
    void setup_complete(bool success, const QString& error);
};
```

**安装流程：**

```
1. 下载 UV 独立二进制 (~13MB)
   └─ uv 0.7.12, GitHub Releases

2. 安装 Python 3.11.9
   └─ uv python install 3.11.9

3. 并行创建两个虚拟环境
   ├─ venv-numpy1 (NumPy 1.x 兼容库)
   └─ venv-numpy2 (NumPy 2.x + 通用库)

4. 并行安装依赖包
   ├─ venv-numpy1: requirements-numpy1.txt
   └─ venv-numpy2: requirements-numpy2.txt

5. Hash 校验 + 标记
   └─ .packages_installed 文件记录 requirements hash
```

**双虚拟环境路由：**

| 虚拟环境 | 用途 | 典型库 |
|----------|------|--------|
| `venv-numpy1` | NumPy 1.x 兼容的量化库 | vectorbt, backtesting, gluonts, functime, pyportfolioopt, financepy, ffn |
| `venv-numpy2` | NumPy 2.x + 通用库 | yfinance, pandas, scipy, py_vollib, 其余所有 |

### 2.5 环境变量体系

**固定变量（所有 Python 进程）：**

| 变量 | 值 | 说明 |
|------|-----|------|
| `PYTHONIOENCODING` | `utf-8` | 强制 UTF-8 输出 |
| `PYTHONDONTWRITEBYTECODE` | `1` | 不生成 .pyc |
| `PYTHONUNBUFFERED` | `1` | 禁用输出缓冲 |
| `PYTHONPATH` | `{scripts_dir}` | 脚本搜索路径 |

**应用变量：**

| 变量 | 说明 |
|------|------|
| `FINCEPT_DATA_DIR` | 应用数据目录 |
| `FINAGENT_DATA_DIR` | Agent 数据目录 |
| `FINAGENT_RUNTIME_CACHE_SIZE` | Agent 运行时缓存大小 |

**凭证变量（从 SecureStorage 注入）：**

| 变量 | 数据源 |
|------|--------|
| `ALPHA_VANTAGE_API_KEY` | Alpha Vantage |
| `FRED_API_KEY` | FRED 经济数据 |
| `OPENAI_API_KEY` | OpenAI LLM |
| `ANTHROPIC_API_KEY` | Anthropic Claude |
| 其他 API keys... | 各数据源 |

**UV 变量：**

| 变量 | 值 | 说明 |
|------|-----|------|
| `UV_PYTHON_INSTALL_DIR` | `{install_dir}/python` | Python 安装位置 |
| `UV_CACHE_DIR` | `{install_dir}/uv-cache` | 包缓存 |
| `UV_LINK_MODE` | `hardlink` | 硬链接节省空间 |
| `UV_COMPILE_BYTECODE` | `1` | 预编译字节码 |
| `UV_CONCURRENT_DOWNLOADS` | 自适应 | 并发下载数 |
| `UV_CONCURRENT_INSTALLS` | 自适应 | 并发安装数 |
| `UV_HTTP_TIMEOUT` | `120` | 网络超时 (秒) |

### 2.6 Python 路径解析优先级

```
1. UV 管理的 venv  (venv-numpy2 或 venv-numpy1)
   └─ {install_dir}/python/venv-{name}/bin/python
2. 可执行文件旁边的 venv (便携/开发模式)
   └─ {app_dir}/venv/bin/python
3. 系统 Python
   └─ python / python3
```

---

## 三、Python 脚本生态 (`scripts/`)

### 3.1 目录结构

```
scripts/
├── agents/                 # AI Agent 实现
│   ├── finagent_core/      # Agent 核心（Agno 框架）
│   ├── geopolitical/       # 地缘政治 Agent (19 个)
│   ├── hedge_fund/         # 对冲基金 Agent (8 个)
│   └── legendary_investors/# 传奇投资者 Agent (2 个)
├── Analytics/              # 分析模块
│   ├── alternateInvestment/# 另类投资分析 (28 模块)
│   ├── backtesting/        # 回测引擎 (5 框架)
│   ├── corporateFinance/   # 公司金融 (M&A, 估值)
│   ├── equity/             # 权益分析
│   ├── portfolio/          # 组合管理
│   └── derivatives/        # 衍生品分析
├── exchange/               # 交易所交易
│   ├── exchange_daemon.py  # 持久交易守护进程
│   ├── kraken.py           # Kraken
│   └── binance.py          # Binance
├── voice/                  # 语音处理
│   ├── tts.py / deepgram_tts.py
│   ├── speech_to_text.py / deepgram_stt.py
│   └── clap_detector.py
├── vision_quant/           # 视觉量化
├── technicals/             # 技术分析
├── agno_trading/           # Agno 交易系统
├── alpha_arena/            # Alpha 竞技场
├── ai_quant_lab/           # AI 量化实验室 (18+ 模块)
├── yfinance_data.py        # Yahoo Finance 数据 (守护进程)
├── option_greeks_daemon.py # 期权 Greeks (守护进程)
├── fincept_output_standard.py  # 统一输出规范
├── akshare_*.py            # 中国市场数据 (20+ 模块)
└── [60+ 数据源脚本]
```

### 3.2 统一输出规范 (`fincept_output_standard.py`)

所有 Python 脚本遵循统一的 JSON 输出格式：

```json
{
    "success": true,
    "data": {
        // 业务数据，结构因脚本而异
    },
    "metadata": {
        "script": "yfinance_data",
        "timestamp": "2026-05-08T12:00:00Z",
        "output_type": "table | dict | array | text | number | multi | error",
        "execution_time_ms": 234
    },
    "error": null
}
```

**错误格式：**

```json
{
    "success": false,
    "data": null,
    "metadata": {
        "script": "yfinance_data",
        "output_type": "error",
        "execution_time_ms": 50
    },
    "error": {
        "message": "Rate limit exceeded",
        "type": "APIError"
    }
}
```

### 3.3 AI Agent 体系 (`scripts/agents/`)

#### 3.3.1 FinAgent Core — Agent 框架

基于 **Agno**（LLM Agent 框架）实现，单可配置 Agent 系统。

| 组件 | 文件 | 职责 |
|------|------|------|
| 入口 | `finagent_core/main.py` | 统一 Agent 执行入口 |
| 核心 Agent | `finagent_core/core_agent.py` | 单 Agent 实例，配置变更时重建 |
| Agent 工厂 | `finagent_core/agent_factory.py` | 从配置创建 Agno Agent |
| 超级 Agent | `finagent_core/super_agent.py` | 多 Agent 协调 |
| 模拟交易 | `finagent_core/paper_trading_bridge.py` | Paper Trading 集成 |

**C++ 调用方式：**

```cpp
PythonRunner::instance().run("agents/finagent_core/main.py",
    {"--action", "run_agent", "--agent_id", "warren_buffett",
     "--query", "Should I buy AAPL?"},
    callback);
```

#### 3.3.2 地缘政治 Agent（19 个）

基于三大战略分析框架：

**大棋局框架 (Grand Chessboard, 5 agents)：**

| Agent | 分析维度 |
|-------|----------|
| `american_primacy_agent` | 美国全球领导力战略 |
| `eurasian_balkans_agent` | 中亚地缘政治 |
| `heartland_agent` | 麦金德心脏地带控制论 |
| `pivots_agent` | 关键地缘政治支点国家 |
| `players_agent` | 主要全球大国博弈 |

**地理囚徒框架 (Prisoners of Geography, 10 agents)：**

| Agent | 地区 |
|-------|------|
| `russia_geography_agent` | 俄罗斯地理约束 |
| `china_geography_agent` | 中国领土战略 |
| `usa_geography_agent` | 美国地理优势 |
| `europe_geography_agent` | 欧洲地理挑战 |
| `middle_east_geography_agent` | 中东地缘 |
| `africa_geography_agent` | 非洲发展约束 |
| `india_pakistan_geography_agent` | 南亚地理 |
| `japan_korea_geography_agent` | 东亚岛国 |
| `latin_america_geography_agent` | 拉美地理 |
| `arctic_geography_agent` | 北极战略意义 |

**世界秩序框架 (World Order, 4 agents)：**

| Agent | 秩序观 |
|-------|--------|
| `american_order_agent` | 自由国际秩序 |
| `chinese_order_agent` | 儒家和谐观 |
| `european_order_agent` | 权力平衡体系 |
| `islamic_order_agent` | 伊斯兰治理原则 |

#### 3.3.3 对冲基金 Agent（8 个）

| Agent | 策略风格 | AUM |
|-------|----------|-----|
| `bridgewater_associates` | 全球宏观、风险平价 | $124B |
| `citadel` | 多策略、量化 | $62B |
| `renaissance_technologies` | 量化模型 | $55B |
| `two_sigma` | AI/ML、系统化交易 | $60B |
| `de_shaw` | 计算金融 | $60B |
| `elliott_management` | 激进主义、困境债务 | $56B |
| `pershing_square` | 激进价值投资 | $16B |
| `arq_capital` | 因子投资、量化 | $90B |

#### 3.3.4 传奇投资者 Agent（2 个）

| Agent | 投资哲学 |
|-------|----------|
| `warren_buffett_agent` | 价值投资、护城河、长期持有 |
| `benjamin_graham_agent` | 深度价值、安全边际、市场先生 |

### 3.4 分析模块 (`scripts/Analytics/`)

#### 3.4.1 回测引擎（5 个框架）

| 框架 | 脚本目录 | 特点 |
|------|----------|------|
| **Backtesting.py** | `backtestingpy/` | 轻量级 Python 回测，策略 + 信号 + 优化 |
| **VectorBT** | `vectorbt/` | 向量化高性能回测 |
| **FastTrade** | `fasttrade/` | 轻量快速回测 |
| **LEAN Engine** | `lean/` | 机构级回测（QuantConnect） |
| **Zipline** | `zipline/` | Quantopian 风格回测 |

每个框架的文件结构：

```
backtestingpy/
├── backtestingpy_provider.py  # Provider 接口（C++ 调用入口）
├── btp_strategies.py          # 策略实现
├── btp_signals.py             # 信号生成
└── btp_optimize.py            # 参数优化
```

#### 3.4.2 公司金融分析

```
corporateFinance/
├── advanced_analytics/   # Monte Carlo 估值、回归分析
├── deal_comparison/      # M&A 交易比较
├── deal_database/        # 交易数据库（Schema/Parser/Scanner/Tracker）
└── deal_structure/       # 交易结构（Collar/Earnout/Exchange Ratio）
```

#### 3.4.3 另类投资分析（28 模块）

| 模块 | 分析内容 |
|------|----------|
| `asset_location` | 战略资产配置 |
| `convertible_bonds` | 可转债分析 |
| `covered_calls` | 备兑看涨策略 |
| `digital_assets` | 数字资产（加密）分析 |
| `emerging_market_bonds` | 新兴市场债券 |
| `hedge_funds` | 对冲基金绩效指标 |
| `real_estate` | 房地产投资分析 |
| `private_capital` | 私募股权/资本 |
| `structured_products` | 结构化产品分析 |

### 3.5 数据源（60+ 提供商）

#### 3.5.1 市场数据

| 脚本 | 数据源 | 数据类型 |
|------|--------|----------|
| `yfinance_data.py` ⚡ | Yahoo Finance | 实时行情、历史数据（守护进程） |
| `alphavantage_data.py` | Alpha Vantage | 股票/外汇/加密 |
| `tradingview_data.py` | TradingView | 市场数据和图表 |
| `databento_provider.py` | Databento | 高频市场数据 |
| `tiingo_data.py` | Tiingo | 高级市场数据 |
| `eodhd_data.py` | EODHD | 历史市场数据 |
| `polygon_io_data.py` | Polygon.io | 股票/外汇/加密 |
| `finnhub_data.py` | Finnhub | 实时金融数据 |

> ⚡ = 持久守护进程（通过 PythonWorker 调用）

#### 3.5.2 经济数据

| 脚本 | 数据源 | 覆盖 |
|------|--------|------|
| `fred_data.py` | FRED | 美联储经济数据 |
| `worldbank_data.py` | World Bank | 全球发展数据 |
| `imf_data.py` | IMF | 国际货币数据 |
| `oecd_data.py` | OECD | 38 国经济数据 |
| `ecb_data.py` | ECB | 欧洲央行数据 |
| `bea_data.py` | BEA | 美国经济数据 |
| `bls_data.py` | BLS | 美国劳工统计 |

#### 3.5.3 加密货币数据

| 脚本 | 数据源 |
|------|--------|
| `coingecko.py` | CoinGecko |
| `coinmarketcap_data.py` | CoinMarketCap |
| `exchange/kraken.py` | Kraken |
| `exchange/binance.py` | Binance |

#### 3.5.4 政府数据

| 脚本 | 数据源 |
|------|--------|
| `sec_data.py` | SEC 文件 |
| `treasury_data.py` | 美国财政部 |
| `federal_reserve_data.py` | 美联储 |
| `congress_gov_data.py` | 美国国会 |

#### 3.5.5 中国市场数据 (AkShare 生态)

| 脚本 | 数据 |
|------|------|
| `akshare_data.py` | 统一接口 |
| `akshare_stocks_*.py` | 多个股票模块 |
| `akshare_futures.py` | 期货交易数据 |
| `akshare_bonds.py` | 债券市场数据 |
| `akshare_macro.py` | 经济指标 |

#### 3.5.6 国际数据

| 脚本 | 国家/地区 |
|------|-----------|
| `eurostat_data.py` | 欧洲 |
| `estat_japan_api.py` | 日本 |
| `datagovuk_api.py` | 英国 |
| `datagov_au_api.py` | 澳大利亚 |

### 3.6 交易所交易 (`scripts/exchange/`)

#### Exchange Daemon — 持久交易守护进程

```python
# exchange_daemon.py — 消除 600-1200ms Python 启动开销
# 协议：JSON-RPC over stdin/stdout
```

**支持的操作：**

| 操作 | 说明 |
|------|------|
| 市场数据 | Tickers, Orderbooks, Trades |
| 订单管理 | 下单、撤单、修改订单 |
| 账户 | 余额、持仓查询 |
| 交易所信息 | 手续费、交易对列表 |

**支持的交易所：** Kraken, Binance, Coinbase Pro, Bitfinex, Huobi, OKX 等

### 3.7 语音模块 (`scripts/voice/`)

| 脚本 | 功能 | 提供商 |
|------|------|--------|
| `tts.py` | 文字转语音 | pyttsx3 |
| `deepgram_tts.py` | 文字转语音 | Deepgram |
| `speech_to_text.py` | 语音转文字 | Google STT |
| `deepgram_stt.py` | 语音转文字 | Deepgram |
| `clap_detector.py` | 拍手检测 | 本地音频分析 |

### 3.8 AI 量化实验室 (`scripts/ai_quant_lab/`)

18+ 专业量化模块，涵盖：

| 类别 | 模块 |
|------|------|
| **市场分析** | QLib 集成、GS Quant、Functime |
| **技术分析** | 技术指标、信号生成 |
| **统计分析** | 因子分析、风险模型 |
| **ML 模型** | 强化学习交易、在线学习、元学习 |
| **组合优化** | 均值方差、Black-Litterman |
| **HFT** | 高频交易策略 |
| **因子挖掘** | Alpha 因子发现和评估 |

---

## 四、数据流详解

### 4.1 一次性脚本执行流

```
C++ Service 调用
    │
    ▼
PythonRunner::run("fred_data", ["get_series", "GDP"], callback)
    │
    ├─ 1. 选择 venv（默认 venv-numpy2）
    ├─ 2. 构建环境变量（注入 API Key 等）
    ├─ 3. 启动 QProcess
    │      python {venv}/python scripts/fred_data.py get_series GDP
    │
    ├─ 4. Python 脚本执行
    │      ├─ 解析命令行参数
    │      ├─ 调用 FRED API
    │      ├─ 构建标准 JSON 输出
    │      └─ print(json.dumps(result))
    │
    ├─ 5. C++ 捕获 stdout
    └─ 6. extract_json() → callback(PythonResult{success, output, ...})
```

### 4.2 持久守护进程数据流

```
C++ Service 调用
    │
    ▼
PythonWorker::submit("batch_all", {quotes: ["AAPL"]}, callback)
    │
    ├─ 1. 检查守护进程是否就绪
    │      ├─ 未启动 → launch_process()
    │      └─ 未就绪 → 入队等待
    │
    ├─ 2. 构建请求帧
    │      {"id": 1, "action": "batch_all", "payload": {...}}
    │      → 4B 长度前缀 + UTF-8 JSON body
    │
    ├─ 3. 写入守护进程 stdin
    │
    ├─ 4. Python 守护进程处理
    │      ├─ 读取帧（4B 长度 + JSON）
    │      ├─ 分发到 action handler
    │      ├─ 执行 yfinance 查询
    │      └─ 写入响应帧到 stdout
    │
    ├─ 5. C++ 读取响应帧
    │      ├─ on_ready_read() → try_drain_frames()
    │      └─ 按 id 匹配到 Pending 请求
    │
    └─ 6. callback(ok, result, error)
```

### 4.3 DataHub 集成数据流

```
Python 脚本 → PythonWorker/Runner → Service (Producer) → DataHub → Screen (Subscriber)

具体示例（行情数据）：

yfinance_data.py (守护进程)
    │  batch_all 响应
    ▼
PythonWorker
    │  callback
    ▼
MarketDataService (Producer)
    │  DataHub::publish("market:quote:AAPL", quoteData)
    ▼
DataHub
    │  subscribe 回调
    ▼
WatchlistScreen / DashboardScreen / EquityTradingScreen
```

---

## 五、关键设计决策

### 5.1 为什么用两个虚拟环境？

| 问题 | 解决方案 |
|------|----------|
| 部分量化库（vectorbt, gluonts 等）依赖 NumPy 1.x | `venv-numpy1` — 安装 NumPy 1.x 兼容版本 |
| yfinance、scipy 等新库需要 NumPy 2.x | `venv-numpy2` — 安装 NumPy 2.x + 通用库 |
| 避免版本冲突 | C++ 侧根据脚本名自动路由到正确的 venv |

**路由规则（C++ 侧硬编码）：**

```cpp
static QStringList kNumpy1Scripts = {
    "vectorbt", "backtesting", "gluonts", "functime",
    "pyportfolioopt", "financepy", "ffn", "ffn_wrapper"
};
// 匹配 → venv-numpy1
// 不匹配 → venv-numpy2 (默认)
```

### 5.2 为什么用持久守护进程？

| 一次性启动开销 | 守护进程方案 |
|---------------|-------------|
| Python 解释器启动 ~200ms | 一次启动，长期复用 |
| yfinance 导入 ~1-2s | 守护进程内常驻 |
| pandas 导入 ~300ms | 守护进程内常驻 |
| **总计 ~2-3s/请求** | **~10ms/请求** |

守护进程适用于高频调用场景（行情刷新每 5-15 秒），一次性脚本适用于低频场景（Agent 执行、回测等）。

### 5.3 为什么用 UV 而非 pip？

| pip | UV |
|-----|-----|
| 串行下载/安装 | 并行下载/安装 |
| 无缓存复用 | 全局缓存 + 硬链接 |
| 无字节码预编译 | `UV_COMPILE_BYTECODE=1` |
| 慢 | 快 10-100x |

---

## 六、依赖管理

### 6.1 核心依赖

| 库 | 版本约束 | 用途 |
|-----|---------|------|
| `agno` | latest | LLM Agent 框架 |
| `yfinance` | >=0.2 | Yahoo Finance 数据 |
| `pandas` | >=2.0 | 数据处理 |
| `numpy` | 1.x / 2.x | 数值计算 |
| `scipy` | >=1.11 | 科学计算 |
| `py_vollib` | latest | 期权 Greeks |
| `ccxt` | >=4.0 | 加密交易所统一接口 |
| `langchain` | >=0.1 | AI 编排 |
| `akshare` | latest | 中国市场数据 |
| `vectorbt` | latest | 向量化回测 |
| `matplotlib` | >=3.7 | 可视化 |

### 6.2 需求文件

```
scripts/
├── requirements-numpy1.txt   # venv-numpy1 的依赖列表
└── requirements-numpy2.txt   # venv-numpy2 的依赖列表
```

每个 venv 安装完成后生成 `.packages_installed` 标记文件，内容为 requirements 文件的 hash 值，用于快速判断是否需要重新安装。

---

## 七、安全机制

### 7.1 凭证隔离

- API Key 存储在 `SecureStorage`（AES-256-GCM 加密）
- Python 进程仅注入白名单内的凭证变量
- 不透传 shell 环境中可能存在的敏感信息

### 7.2 进程沙箱

- Python 进程通过 QProcess 启动，权限继承 C++ 主进程
- 环境变量最小化注入
- 大参数通过临时文件传递，避免命令行注入

### 7.3 错误隔离

- Python 脚本崩溃不影响 C++ 主进程
- 守护进程自动重启，有最大重启次数限制
- 所有错误通过结构化 JSON 返回

---

## 八、性能优化策略

| 策略 | 实现方式 | 效果 |
|------|----------|------|
| **守护进程复用** | PythonWorker / OptionGreeksWorker | 避免每次 2-3s 启动开销 |
| **并发限制** | PythonRunner max_concurrent=3 | 防止资源耗尽 |
| **UV 快速安装** | 并行下载/安装 + 缓存 | 安装速度快 10-100x |
| **双 venv 隔离** | numpy1/numpy2 分离 | 避免 numpy 版本冲突 |
| **字节码预编译** | UV_COMPILE_BYTECODE=1 | 减少 import 时间 |
| **大参数溢出** | >8KB 写临时文件 | 避免 Windows 命令行限制 |
| **请求合并** | batch_all 动作 | 一次请求获取多种数据 |
| **Hash 标记** | .packages_installed | 快速跳过已安装的包 |

---

## 九、模块统计

| 类别 | 数量 | 说明 |
|------|------|------|
| Python 脚本总数 | 1,411 | 所有 .py 文件 |
| AI Agent | 30+ | 地缘政治 + 对冲基金 + 投资者 |
| 数据源提供商 | 60+ | 覆盖全球主要金融数据源 |
| 回测框架 | 5 | Backtesting.py, VectorBT, FastTrade, LEAN, Zipline |
| 另类投资模块 | 28 | 覆盖可转债/房地产/私募/结构化产品等 |
| 中国市场模块 | 20+ | AkShare 生态全覆盖 |
| 交易所支持 | 6+ | Kraken, Binance, Coinbase, Bitfinex, Huobi, OKX |
| 语音模块 | 5 | TTS + STT + 拍手检测 |
| 量化实验室模块 | 18+ | QLib/GS Quant/Functime/RL/因子挖掘等 |
| 持久守护进程 | 3 | yfinance_data, option_greeks, exchange_daemon |
