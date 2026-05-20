# `python main.py --serve` 执行流程详解

## 一、整体架构概览

`--serve` 模式 = **FastAPI 后端服务** + **后台启动即时分析任务** + **可选 Bot Stream 客户端** + **主线程阻塞保持运行**。

```
python main.py --serve
  │
  ├─ 1. 环境初始化 & 配置加载
  ├─ 2. 日志系统初始化
  ├─ 3. 模式判定 (--serve → start_serve = True)
  ├─ 4. 前端静态资源准备
  ├─ 5. 启动 FastAPI 服务（后台守护线程）
  ├─ 6. 启动 Bot Stream 客户端（钉钉/飞书）
  ├─ 7. 执行完整分析流程（run_full_analysis）
  │      ├─ 7.1 交易日过滤
  │      ├─ 7.2 创建 Pipeline & 并发分析个股
  │      ├─ 7.3 大盘复盘
  │      ├─ 7.4 合并推送通知
  │      ├─ 7.5 飞书云文档生成
  │      └─ 7.6 自动回测
  └─ 8. 主线程阻塞（while True + sleep），等待 Ctrl+C
```

---

## 二、启动链路（逐行解析）

### Step 1: 入口 & 参数解析

**文件**: [main.py:756](main.py#L756) `main()`

```
sys.exit(main())
  → parse_arguments()       # 解析 --serve, --port, --host 等
  → _setup_bootstrap_logging()  # 最早期的 stderr 日志
```

`--serve` 触发 `args.serve = True`，`args.serve_only = False`（区别于 `--serve-only`）。

### Step 2: 配置加载

**文件**: [main.py:778-782](main.py#L778-L782)

```python
config = get_config()  # 读取 .env → Config 单例
warnings = config.validate()  # 验证配置完整性
```

### Step 3: 运行日志切换

**文件**: [main.py:785-788](main.py#L785-L788)

```python
_setup_runtime_logging(config.log_dir, debug=args.debug)
# 从纯 stderr 切换到 "控制台 + 文件" 双输出
```

### Step 4: --webui 兼容映射

**文件**: [main.py:818-826](main.py#L818-L826)

```
--webui     → args.serve = True
--webui-only → args.serve_only = True
WEBUI_ENABLED=true (旧环境变量) → args.serve = True
```

### Step 5: 端口/地址解析

**文件**: [main.py:830-835](main.py#L830-L835)

兼容旧版 `WEBUI_HOST` / `WEBUI_PORT` 环境变量：如果用户未通过 `--host` / `--port` 显式指定，则从环境变量读取。

### Step 6: 前端资源准备 & API 启动

**文件**: [main.py:838-848](main.py#L838-L848)

```python
prepare_webui_frontend_assets()   # 构建前端静态资源
start_api_server(host, port, config)  # → 后台线程启动 uvicorn
```

**关键**: `start_api_server` 在 **后台守护线程** 运行 uvicorn，不阻塞主流程：

```python
# main.py:659-671
def run_server():
    uvicorn.run("api.app:app", host=host, port=port, log_level=..., log_config=None)

thread = threading.Thread(target=run_server, daemon=True)
thread.start()
```

### Step 7: Bot Stream 客户端启动

**文件**: [main.py:847-848](main.py#L847-L848), [main.py:680-710](main.py#L680-L710)

API 启动成功后，根据配置启动：
- **钉钉 Stream** (`dingtalk_stream_enabled`) → `start_dingtalk_stream_background()`
- **飞书 Stream** (`feishu_stream_enabled`) → `start_feishu_stream_background()`

均在后台线程运行。

---

## 三、分析任务执行（`run_full_analysis`）

### Step 8: 进入分析流程

**文件**: [main.py:864-985](main.py#L864-L985)

因为 `args.serve_only = False`，所以**不会**进入"仅服务"的 while-true 阻塞，而是继续执行分析。

### 8.1 交易日过滤（Issue #373）

**文件**: [main.py:470-483](main.py#L470-L483)

```python
_compute_trading_day_filter(config, args, effective_codes)
```

- 按市场判断每只股票今天是否交易
- 非交易日的股票被跳过
- 如果所有股票 + 大盘复盘都无可执行市场 → 整个 run 跳过

### 8.2 创建 Pipeline & 执行个股分析

**文件**: [main.py:502-516](main.py#L502-L516)

```python
pipeline = StockAnalysisPipeline(config=config, max_workers=args.workers, ...)
results = pipeline.run(stock_codes=stock_codes, dry_run=False, send_notification=True, ...)
```

**Pipeline 内部执行流程**（[pipeline.py:1704-1869](src/core/pipeline.py#L1704-L1869)）：

```
pipeline.run()
  │
  ├─ prefetch_realtime_quotes()    # 批量预取实时行情（股票数 ≥ 5 时）
  ├─ prefetch_stock_names()        # 预取股票名称
  │
  └─ ThreadPoolExecutor(max_workers)  # 低并发线程池（默认 3）
       │
       └─ process_single_stock(code)   # 每只股票完整流程：
            │
            ├─ Step 1: fetch_and_save_stock_data()
            │     ├─ 断点续传检查（has_today_data）
            │     └─ 从数据源拉取 → 保存到 SQLite
            │
            ├─ Step 2: analyze_stock(code)
            │     ├─ 获取实时行情（量比、换手率）
            │     ├─ 获取筹码分布
            │     ├─ 基本面聚合 (get_fundamental_context)
            │     ├─ 保存基本面快照
            │     ├─ 趋势分析 (trend_analyzer.analyze)
            │     │
            │     ├─ 分支 A: Agent 模式（配置了 agent_mode 或 agent_skills）
            │     │     → _analyze_with_agent()
            │     │        → build_agent_executor() → executor.run()
            │     │        → AgentResult → AnalysisResult 转换
            │     │
            │     └─ 分支 B: 传统模式（默认）
            │           ├─ 多维度情报搜索（新闻、风险、业绩预期）
            │           ├─ 社交舆情（美股专用）
            │           ├─ 获取分析上下文（技术面数据）
            │           ├─ 增强上下文（实时行情 + 筹码 + 趋势 + 基本面）
            │           ├─ LLM 流式分析 (analyzer.analyze)
            │           └─ 保存分析历史到 DB
            │
            └─ Step 3: 单股推送（single_notify 模式）
```

### 8.3 分析间隔延迟

**文件**: [main.py:519-527](main.py#L519-L527)

```python
time.sleep(analysis_delay)  # 个股分析 & 大盘复盘之间的间隔，避免 API 限流
```

### 8.4 大盘复盘

**文件**: [main.py:530-548](main.py#L530-L548)

```python
_run_market_review_with_shared_lock(
    config, run_market_review,
    notifier=pipeline.notifier,
    analyzer=pipeline.analyzer,
    search_service=pipeline.search_service,
    ...
)
```

- 使用 **分布式锁**（文件锁）防止重复执行
- 复用 Pipeline 中的分析器和搜索服务
- 按市场过滤（`effective_region`）确定复盘范围

### 8.5 合并推送通知

**文件**: [main.py:551-567](main.py#L551-L567)

当 `merge_notification = True` 时：

```
合并内容 = 大盘复盘报告 + 个股决策仪表盘
  → pipeline.notifier.send(combined_content, email_send_to_all=True, route_type="report")
```

### 8.6 飞书云文档生成

**文件**: [main.py:582-621](main.py#L582-L621)

```python
FeishuDocManager().create_daily_doc(doc_title, full_content)
```

- 标题格式: `"YYYY-MM-DD HH:MM 大盘复盘"`
- 内容 = 大盘复盘 + 个股决策仪表盘
- 创建成功后将文档链接推送通知

### 8.7 自动回测

**文件**: [main.py:624-641](main.py#L624-L641)

```python
if backtest_enabled:
    BacktestService().run_backtest(force=False, ...)
```

对历史分析结果进行评估（fail-open，失败不影响主流程）。

---

## 四、FastAPI 服务架构

### 4.1 应用创建

**文件**: [api/app.py:141-340](api/app.py#L141-L340)

```
create_app()
  │
  ├─ FastAPI 实例创建（title, description, lifespan）
  ├─ CORS 中间件（允许 localhost:5173, localhost:3000 等）
  ├─ 认证中间件（可选运行时认证）
  ├─ 注册 API v1 路由
  ├─ 错误处理器
  │
  └─ 路由注册：
       │
       ├─ GET  /          → 前端 index.html 或 "Frontend Not Built" 引导页
       ├─ GET  /api/health → 健康检查
       ├─ GET  /assets/{path} → 静态资源（带路径安全校验）
       └─ GET  /{full_path} → SPA 路由回退（非 API 请求返回 index.html）
```

### 4.2 API 路由树

**文件**: [api/v1/router.py](api/v1/router.py)

```
/api/v1
  ├── /auth        → 认证（登录、验证）
  ├── /agent       → Agent 管理
  ├── /analysis    → 股票分析（核心）
  │     ├── POST   /analyze          → 触发分析（同步/异步）
  │     ├── POST   /market-review    → 触发大盘复盘
  │     ├── GET    /tasks            → 任务列表
  │     ├── GET    /tasks/stream     → SSE 实时推送
  │     └── GET    /status/{task_id} → 查询任务状态
  ├── /history     → 历史记录查询
  ├── /stocks      → 股票行情数据
  ├── /backtest    → 回测接口
  ├── /system      → 系统配置管理
  ├── /usage       → 使用情况统计
  ├── /portfolio   → 组合管理
  └── /alerts      → 告警管理
```

### 4.3 异步分析任务队列

**文件**: [api/v1/endpoints/analysis.py:197-305](api/v1/endpoints/analysis.py#L197-L305)

```
POST /api/v1/analysis/analyze
  │
  ├─ 校验参数（stock_code / stock_codes）
  ├─ 归一化股票代码（名称→代码、大小写统一）
  ├─ 去重 & 批量限制（最多 50 只）
  │
  ├─ 同步模式（async_mode=false）
  │     → _handle_sync_analysis()
  │        → AnalysisService.analyze_stock()  直接执行
  │        → 返回 200 AnalysisResultResponse
  │
  └─ 异步模式（async_mode=true）
        → _handle_async_analysis_batch()
           → task_queue.submit_tasks_batch()
           → 返回 202 TaskAccepted
           → 后台线程执行实际分析
```

**任务队列** (`src/services/task_queue.py`) 核心能力：
- 防重复提交（相同股票正在分析中返回 409）
- 任务状态机：`pending → processing → completed / failed`
- SSE 事件推送（`task_created`, `task_progress`, `task_completed`, `task_failed`）
- 后台任务支持（大盘复盘等长时间任务）

---

## 五、数据流总览

```
┌─────────────┐     ┌──────────────┐     ┌───────────────┐
│  .env 配置   │ ──→ │  Config 单例  │ ──→ │  Pipeline 初始化 │
└─────────────┘     └──────────────┘     └───────┬───────┘
                                                  │
                    ┌─────────────────────────────┤
                    ▼                             ▼
          ┌─────────────────┐         ┌──────────────────────┐
          │ 数据获取层        │         │ 搜索服务              │
          │ DataFetcherManager│         │ (新闻/舆情/情报)      │
          │ ├─ AKShare       │         └──────────┬───────────┘
          │ ├─ yFinance      │                    │
          │ ├─ Tushare       │                    │
          │ ├─ Longbridge    │                    │
          │ └─ ...           │                    │
          └────────┬─────────┘                    │
                   │                              │
                   ▼                              ▼
          ┌─────────────────────────────────────────────┐
          │           分析引擎                           │
          │  ┌──────────────┐   ┌─────────────────────┐ │
          │  │ 技术分析器     │   │ LLM 分析器           │ │
          │  │ StockTrend    │   │ GeminiAnalyzer      │ │
          │  │ Analyzer      │   │ 或 Agent Executor   │ │
          │  └──────────────┘   └─────────────────────┘ │
          └─────────────────────┬───────────────────────┘
                                ▼
                    ┌───────────────────────┐
                    │  SQLite 持久化         │
                    │  ├─ 日线行情           │
                    │  ├─ 分析历史           │
                    │  ├─ 基本面快照         │
                    │  └─ 新闻情报           │
                    └───────────┬───────────┘
                                ▼
                    ┌───────────────────────┐
                    │  通知服务              │
                    │  ├─ 企业微信            │
                    │  ├─ 钉钉/飞书           │
                    │  ├─ Telegram/Discord   │
                    │  ├─ Email              │
                    │  └─ ...               │
                    └───────────────────────┘
```

---

## 六、`--serve` vs `--serve-only` 对比

| 维度 | `--serve` | `--serve-only` |
|------|-----------|----------------|
| API 服务 | 启动 | 启动 |
| Bot Stream | 启动（如果配置启用） | 启动（如果配置启用） |
| 立即执行分析 | 是（启动后立即 run_full_analysis） | 否 |
| 后续定时任务 | 否（不进入 schedule 模式） | 否 |
| 主线程行为 | 执行完分析后进入 while-true 阻塞 | 直接进入 while-true 阻塞 |
| 触发分析方式 | 自动 + API 接口 | 仅通过 API 接口手动触发 |
| 适用场景 | 定时执行 + 提供 API 查询 | 纯 API 服务，按需触发 |

---

## 七、线程模型

```
主线程 (main thread)
  │
  ├─ 1. 环境初始化 & 配置加载
  ├─ 2. 日志初始化
  ├─ 3. 前端资源准备
  ├─ 4. 启动 FastAPI（生成守护线程）
  ├─ 5. 启动 Bot Stream（生成守护线程）
  ├─ 6. run_full_analysis()
  │     │
  │     └─ ThreadPoolExecutor(max_workers=3)
  │           ├─ 线程1: process_single_stock(stock_A)
  │           ├─ 线程2: process_single_stock(stock_B)
  │           └─ 线程3: process_single_stock(stock_C)
  │
  └─ 7. while True: time.sleep(1)  ← 主线程阻塞，保持进程存活

后台守护线程 (daemon threads)
  ├─ uvicorn ASGI 服务线程（FastAPI）
  ├─ 钉钉 Stream 客户端线程（如果启用）
  └─ 飞书 Stream 客户端线程（如果启用）
```

---

## 八、关键文件清单

| 文件 | 职责 |
|------|------|
| [main.py](main.py) | 主调度入口、CLI 参数解析、模式路由 |
| [server.py](server.py) | FastAPI 独立启动入口（uvicorn server:app） |
| [api/app.py](api/app.py) | FastAPI 应用工厂、CORS、路由注册、SPA 托管 |
| [api/v1/router.py](api/v1/router.py) | API v1 路由聚合 |
| [api/v1/endpoints/analysis.py](api/v1/endpoints/analysis.py) | 分析接口（同步/异步/SSE） |
| [src/core/pipeline.py](src/core/pipeline.py) | 核心分析流水线编排 |
| [src/scheduler.py](src/scheduler.py) | 定时调度模块 |
| [src/config.py](src/config.py) | 配置管理单例 |
| [data_provider/](data_provider/) | 多数据源适配层 |
| [src/analyzer/](src/analyzer/) | LLM 分析器 |
| [src/agent/](src/agent/) | Agent 模式（多 Agent 协作） |
| [src/notification.py](src/notification.py) | 多通道通知服务 |
| [src/storage.py](src/storage.py) | SQLite 数据访问 |
| [src/services/task_queue.py](src/services/task_queue.py) | 异步任务队列 |

---

## 九、完整执行时序图

```
时间 →

main()
  │
  ├── parse_arguments() ─────────────────────────────── args.serve = True
  ├── get_config() ──────────────────────────────────── 加载 .env
  ├── setup_logging() ───────────────────────────────── 日志就绪
  │
  ├── prepare_webui_frontend_assets() ───────────────── 前端资源
  │
  ├── start_api_server() ────────────────────────────── [后台线程] uvicorn 启动
  │     │
  │     └── create_app()
  │           ├── CORS 中间件
  │           ├── 认证中间件
  │           ├── 注册 /api/v1/* 路由
  │           └── 注册 SPA 回退路由
  │
  ├── start_bot_stream_clients() ────────────────────── [后台线程] 钉钉/飞书
  │
  ├── run_full_analysis() ───────────────────────────── 主流程分析
  │     │
  │     ├── _compute_trading_day_filter() ───────────── 交易日过滤
  │     ├── StockAnalysisPipeline() ─────────────────── 初始化各模块
  │     │     ├── DataFetcherManager
  │     │     ├── StockTrendAnalyzer
  │     │     ├── GeminiAnalyzer
  │     │     ├── NotificationService
  │     │     ├── SearchService
  │     │     └── SocialSentimentService
  │     │
  │     ├── pipeline.run() ──────────────────────────── 并发分析
  │     │     ├── prefetch_realtime_quotes()
  │     │     ├── prefetch_stock_names()
  │     │     └── ThreadPoolExecutor(max_workers=3)
  │     │           ├── process_single_stock(A) ──→ fetch → analyze → save
  │     │           ├── process_single_stock(B) ──→ fetch → analyze → save
  │     │           └── process_single_stock(C) ──→ fetch → analyze → save
  │     │
  │     ├── time.sleep(analysis_delay) ──────────────── 间隔延迟
  │     │
  │     ├── _run_market_review_with_shared_lock() ───── 大盘复盘（带锁）
  │     │
  │     ├── merge_notification 合并推送 ─────────────── 个股 + 大盘
  │     │
  │     ├── FeishuDocManager.create_daily_doc() ─────── 飞书文档
  │     │
  │     └── BacktestService.run_backtest() ──────────── 自动回测
  │
  └── while True: sleep(1) ──────────────────────────── 主线程阻塞
        （API 服务继续在后台线程响应请求）
        │
        ├── POST /api/v1/analysis/analyze ─────────── 用户手动触发分析
        │     └── task_queue.submit_tasks_batch() ──→ 后台线程执行
        │
        ├── GET /api/v1/tasks/stream ──────────────── SSE 长连接
        │     └── 每 30s 心跳 + 任务事件推送
        │
        └── GET /api/v1/status/{task_id} ──────────── 查询任务状态
```

---

## 十、配置项影响速查

| 配置项 | 影响 |
|--------|------|
| `STOCK_LIST` | 分析的股票列表 |
| `MAX_WORKERS` | 并发线程数（默认 3） |
| `SCHEDULE_TIME` | 定时任务执行时间（`--serve` 不进入 schedule 模式） |
| `SCHEDULE_ENABLED` | 是否启用定时任务（`--serve` 不进入 schedule 模式） |
| `MARKET_REVIEW_ENABLED` | 是否启用大盘复盘 |
| `MERGE_EMAIL_NOTIFICATION` | 个股+大盘合并推送 |
| `SINGLE_STOCK_NOTIFY` | 单股推送模式 |
| `ANALYSIS_DELAY` | 个股与大盘分析之间的延迟秒数 |
| `BACKTEST_ENABLED` | 自动回测开关 |
| `DINGTALK_STREAM_ENABLED` | 钉钉 Stream 客户端 |
| `FEISHU_STREAM_ENABLED` | 飞书 Stream 客户端 |
| `AGENT_MODE` | 启用 Agent 分析模式 |
| `AGENT_SKILLS` | Agent 分析技能列表 |
| `ENABLE_REALTIME_QUOTE` | 实时行情开关 |
| `ENABLE_CHIP_DISTRIBUTION` | 筹码分布开关 |
| `REPORT_TYPE` | 报告类型（simple/brief/full） |
| `REPORT_LANGUAGE` | 报告语言（zh/en） |
| `WEBUI_ENABLED` | 兼容旧版 WebUI 开关 |
| `CORS_ORIGINS` | 额外允许的跨域来源 |
| `TRADING_DAY_CHECK_ENABLED` | 交易日检查开关 |
