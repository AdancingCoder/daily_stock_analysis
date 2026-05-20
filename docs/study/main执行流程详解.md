# `python main.py` 完整执行流程与数据流

> 本文档覆盖 `python main.py`（无参数）的全部执行路径和所有可选模式，逐阶段追踪从启动到退出的完整生命周期。

## 一、项目定位与总览

**股票智能分析系统（DSA）** 的核心主程序入口是 `main.py`，它既是 CLI 工具，也是 Web 服务启动器，还是定时调度控制器。

### 一条命令，六种运行模式

| 模式 | 触发方式 | 说明 |
|------|----------|------|
| **单次分析** | `python main.py`（默认） | 读取 `STOCK_LIST`，执行一次完整分析后退出 |
| **定时任务** | `python main.py --schedule` | 每日 `18:00`（可配）自动分析，永久运行 |
| **Web 服务** | `python main.py --webui` | 启动 Web UI + API + 同时执行分析 |
| **仅 Web** | `python main.py --webui-only` | 仅启动 Web UI + API，不自动分析 |
| **大盘复盘** | `python main.py --market-review` | 仅运行大盘分析，不分析个股 |
| **回测** | `python main.py --backtest` | 对历史分析结果做回测评估 |

### 所有 CLI 参数速查

```
--debug              启用调试模式，输出详细日志
--dry-run            仅获取数据，不调用 LLM 分析
--stocks CODE,CODE   指定分析目标（覆盖 .env 中的 STOCK_LIST）
--no-notify          不发送推送通知
--check-notify       检查通知渠道配置，不发送通知
--single-notify      每分析完一只股票立即推送（非汇总）
--workers N          并发线程数
--schedule           定时任务模式（每日执行）
--no-run-immediately 定时模式启动时不立即执行一次
--market-review      仅运行大盘复盘
--no-market-review   跳过大盘复盘
--force-run          跳过交易日检查，强制执行
--webui              启动 Web 管理界面（同时分析）
--webui-only         仅启动 Web 服务
--serve              启动 FastAPI 后端（同时分析）
--serve-only         仅启动 FastAPI 后端
--port PORT          服务端口（默认 8000）
--host HOST          服务监听地址（默认 0.0.0.0）
--no-context-snapshot 不保存分析上下文快照
--backtest           运行回测
--backtest-code CODE 仅回测指定股票
--backtest-days N    回测评估窗口（交易日数）
--backtest-force     强制回测（忽略已有结果）
```

## 二、启动阶段（所有模式共用）

### Phase 1: 模块级初始化（import 时执行）

```python
# main.py:1-34
setup_env()          # 加载 .env 文件到环境变量
```

`setup_env()` 在模块导入时**立即执行**，读取 `.env` 文件并写入 `os.environ`。后续所有配置读取都依赖此时的环境变量。

**代理配置**：如果 `USE_PROXY=true` 且非 GitHub Actions 环境，自动配置 `http_proxy` 和 `https_proxy`。

### Phase 2: `main()` 函数入口

```python
# main.py:756-997
def main() -> int:
```

#### Step 2.1: 解析命令行参数

```python
# main.py:764
args = parse_arguments()
```

#### Step 2.2: 初始化 Bootstrap 日志

```python
# main.py:768
_setup_bootstrap_logging(debug=args.debug)
```

在配置加载前先初始化 `stderr` 日志输出，确保早期失败能落盘。此时还不知道 `log_dir`，所以**不写文件日志**。

#### Step 2.3: 加载全局配置

```python
# main.py:779
config = get_config()
```

`Config` 单例从环境变量读取所有配置项，核心包括：

| 配置类别 | 关键变量 | 默认值 |
|----------|----------|--------|
| AI 模型 | `LITELLM_MODEL`, `GEMINI_API_KEY`, `OPENAI_API_KEY` | 无 |
| 自选股 | `STOCK_LIST` | 无 |
| 并发数 | `MAX_WORKERS` | 3 |
| 定时调度 | `SCHEDULE_TIME`, `SCHEDULE_ENABLED` | `"18:00"`, `false` |
| 实时行情 | `ENABLE_REALTIME_QUOTE`, `REALTIME_SOURCE_PRIORITY` | `true`, `"efinance"` |
| 筹码分布 | `ENABLE_CHIP_DISTRIBUTION` | `true` |
| 基本面 | `ENABLE_FUNDAMENTAL_PIPELINE` | `true` |
| 新闻搜索 | `SERPAPI_API_KEYS`, `TAVILY_API_KEYS` 等 | 无 |
| 通知渠道 | `WECHAT_WEBHOOK_URL`, `FEISHU_WEBHOOK_URL` 等 | 无 |
| 报告类型 | `REPORT_TYPE` | `"simple"` |
| 语言 | `REPORT_LANGUAGE` | `"zh"` |
| 大盘复盘 | `MARKET_REVIEW_ENABLED` | `true` |
| 交易日检查 | `TRADING_DAY_CHECK_ENABLED` | `true` |
| Agent 模式 | `AGENT_MODE`, `AGENT_SKILLS` | `false`, `[]` |
| 自动回测 | `BACKTEST_ENABLED` | `false` |

#### Step 2.4: 切换运行时日志

```python
# main.py:786
_setup_runtime_logging(config.log_dir, debug=args.debug)
```

切换到配置指定的日志目录，开始**同时输出到控制台和文件**。

#### Step 2.5: 验证配置

```python
# main.py:797-799
warnings = config.validate()
for warning in warnings:
    logger.warning(warning)
```

检查关键配置是否完整（如 API Key、自选股列表等），打印警告但不阻塞。

### Phase 3: 模式分流

```python
# main.py:801-816
```

在正式进入模式前，先处理两个特殊分支：

#### 3.1: `--check-notify` 通知诊断

```python
if args.check_notify:
    result = run_notification_diagnostics(config)
    print(format_notification_diagnostics(result))
    return 0
```

运行通知渠道诊断：检查每个渠道的配置是否有效、能否发送消息。打印诊断报告后退出。

#### 3.2: 解析股票列表

```python
if args.stocks:
    stock_codes = [canonical_stock_code(c) for c in args.stocks.split(',')]
else:
    stock_codes = None  # 后续从 config.stock_list 读取
```

如果指定了 `--stocks`，将代码标准化（转大写、去空格）。否则为 `None`，由 Pipeline 运行时从配置读取。

#### 3.3: `--webui` 映射到 `--serve`

```python
# main.py:818-821
if args.webui:
    args.serve = True
if args.webui_only:
    args.serve_only = True
```

`--webui` 和 `--webui-only` 是旧版别名，分别映射到 `--serve` 和 `--serve-only`。

兼容旧版 `WEBUI_ENABLED` 环境变量：

```python
if config.webui_enabled and not (args.serve or args.serve_only):
    args.serve = True
```

## 三、模式 A: Web 服务启动（serve / serve-only）

### 触发条件

`args.serve` 或 `args.serve_only` 为 `True`，且非 GitHub Actions 环境。

### Step A1: 准备前端静态资源

```python
# main.py:839
if not prepare_webui_frontend_assets():
    logger.warning("前端静态资源未就绪，Web 页面可能不可用")
```

检查 `apps/dsa-web/` 构建产物是否存在于正确位置。如果不存在，尝试自动构建（开发模式）或跳过。

### Step A2: 在后台线程启动 FastAPI

```python
# main.py:842
start_api_server(host=args.host, port=args.port, config=config)
```

`start_api_server()` 内部逻辑 ([main.py:647](main.py#L647))：

```python
def start_api_server(host, port, config):
    def run_server():
        uvicorn.run("api.app:app", host=host, port=port, ...)
    thread = threading.Thread(target=run_server, daemon=True)
    thread.start()
```

关键点：
- 使用 `daemon=True` 守护线程，主线程退出时自动终止
- uvicorn 加载 `api.app:app`（FastAPI 应用工厂）
- `log_config=None` 使用系统日志而非 uvicorn 自带日志

### Step A3: FastAPI 应用初始化

`api/app.py` 的 `create_app()` 函数执行：

1. 创建 FastAPI 实例，配置生命周期钩子
2. 注册 CORS 中间件（允许跨域访问）
3. 注册路由：
   - `/api/v1/analysis/*` - 分析触发、任务进度 SSE
   - `/api/v1/agent/*` - Agent 问股对话
   - `/api/v1/history/*` - 历史报告查询
   - `/api/v1/backtest/*` - 回测管理
   - `/api/v1/portfolio/*` - 持仓管理
   - `/api/v1/alerts/*` - 告警管理
   - `/api/v1/stocks/*` - 股票搜索与补全
   - `/api/v1/config/*` - 系统配置（读写 `.env`）
   - `/api/v1/auth/*` - 认证登录
   - `/api/v1/health` - 健康检查
   - `/api/v1/usage` - LLM 用量统计
   - `/docs` - Swagger API 文档
4. 挂载前端静态文件（SPA 路由兜底到 `index.html`）
5. 启动时自检：验证 `index.html` 引用的 JS/CSS 资源是否存在

### Step A4: 启动机器人 Stream 客户端

```python
# main.py:848
start_bot_stream_clients(config)
```

如果启用了钉钉/飞书 Stream 模式，在后台启动对应客户端：

```python
if config.dingtalk_stream_enabled:
    start_dingtalk_stream_background()
if config.feishu_stream_enabled:
    start_feishu_stream_background()
```

### Step A5: `serve-only` 阻塞等待

```python
# main.py:851-862
if args.serve_only:
    while True:
        time.sleep(1)
```

无限循环阻塞，仅保持服务运行。用户可以通过 Web UI (`/api/v1/analysis/analyze`) 手动触发分析。

## 四、模式 B: 单次分析（默认模式）

这是 `python main.py` **最核心的路径**。

### Step B1: 回测模式检查

```python
# main.py:866-880
if args.backtest:
    service = BacktestService()
    stats = service.run_backtest(code=..., force=..., eval_window_days=...)
    return 0
```

`BacktestService` 从数据库读取历史分析记录，评估预测准确率，将回测结果写回 `backtest_results` 表。

### Step B2: 大盘复盘模式检查

```python
# main.py:883-914
if args.market_review:
    notifier, analyzer, search_service = build_market_review_runtime(config)
    _run_market_review_with_shared_lock(config, run_market_review, ...)
    return 0
```

大盘复盘独立于个股分析：
- 先检查相关市场是否开盘（A股/港股/美股）
- 使用文件锁防止并发执行（`market_review_lock.py`）
- 获取市场统计（指数、涨跌家数、板块排行）
- 调用 LLM 生成大盘分析报告
- 推送到通知渠道

### Step B3: 定时任务模式检查

```python
# main.py:917-965
if args.schedule or config.schedule_enabled:
    run_with_schedule(
        task=scheduled_task,
        schedule_time=config.schedule_time,
        run_immediately=should_run_immediately,
        background_tasks=background_tasks,
        schedule_time_provider=schedule_time_provider,
    )
    return 0
```

详见下方「模式 C: 定时任务」。

### Step B4: 单次执行分析（默认路径）

```python
# main.py:968-985
if config.run_immediately:
    run_full_analysis(config, args, stock_codes)
```

**`run_full_analysis()` 完整链路** ([main.py:450](main.py#L450))：

#### B4.1: 交易日过滤（Issue #373）

```python
# main.py:471-483
filtered_codes, effective_region, should_skip = _compute_trading_day_filter(...)
```

按市场判断今日是否开盘：
- A 股：检查上证指数是否开盘
- 港股：检查恒生指数是否开盘
- 美股：检查纳斯达克是否开盘

如果股票所属市场今日休市，该股票被自动跳过。

```
输入: ["600519", "HK00700", "AAPL"]
假设今天是 A 股节假日:
输出: filtered_codes = ["HK00700", "AAPL"]
     skipped = {"600519"} → "今日休市股票已跳过: {'600519'}"
```

#### B4.2: 创建 Pipeline 实例

```python
# main.py:502-508
pipeline = StockAnalysisPipeline(
    config=config,
    max_workers=args.workers,
    query_id=uuid.uuid4().hex,
    query_source="cli",
    save_context_snapshot=save_context_snapshot
)
```

Pipeline 初始化各子模块：
- `db` → SQLite 数据库（单例）
- `fetcher_manager` → 多数据源管理器（10+ 数据源）
- `trend_analyzer` → 技术分析器（均线/MACD/量价）
- `analyzer` → GeminiAnalyzer（LLM 分析）
- `notifier` → NotificationService（12+ 通知渠道）
- `search_service` → SearchService（5+ 搜索引擎）
- `social_sentiment_service` → 社交舆情（仅美股）

#### B4.3: 执行个股分析

```python
# main.py:511-516
results = pipeline.run(
    stock_codes=stock_codes,
    dry_run=args.dry_run,
    send_notification=not args.no_notify,
    merge_notification=merge_notification
)
```

`pipeline.run()` 内部：

```
1. 获取股票列表（如果未指定，从 config.stock_list 读取）
2. 批量预取股票名称（避免并发时显示「股票xxxxx」）
3. 批量预取实时行情（≥5 只时触发全市场拉取）
4. 冻结本轮统一参考时间（断点续传用）
5. ThreadPoolExecutor(max_workers=N) 并发执行
   ├── process_single_stock(code_1)
   ├── process_single_stock(code_2)
   └── process_single_stock(code_3)
6. 收集所有结果到 results 列表
7. 统计: 成功 N, 失败 M, 耗时 X.XX 秒
8. 保存本地报告（Markdown 文件）
9. 发送通知推送（见下方通知推送详细链路）
```

`process_single_stock()` 内部链路（每只股票独立执行）：

```
1. set_frozen_target_date(frozen_td)  # 冻结目标交易日
2. fetch_and_save_stock_data(code)
   ├── get_stock_name()  →  "贵州茅台"
   ├── db.has_today_data()  →  断点续传检查
   ├── fetcher_manager.get_daily_data()  →  日线数据获取
   │   数据源故障切换: Efinance(P0) → AkShare(P1) → Pytdx(P2) → Baostock(P3) → YFinance(P4)
   │   输出: DataFrame[date, open, high, low, close, volume, amount, pct_chg, ma5, ma10, ma20]
   └── db.save_daily_data(df, code, source_name)  →  写入 SQLite
3. analyze_stock(code, report_type, query_id)
   ├── get_realtime_quote()  →  实时行情（价格、量比、换手率、PE、PB）
   ├── get_chip_distribution()  →  筹码分布（获利比例、集中度，带熔断器）
   ├── get_fundamental_context()  →  基本面聚合（7 模块块，fail-open）
   ├── trend_analyzer.analyze(df)  →  趋势分析（MA 状态、买卖信号、评分）
   ├── search_comprehensive_intel()  →  多维度新闻搜索（最多 5 次）
   ├── _enhance_context()  →  合并所有数据为一个字典
   ├── analyzer.analyze(context, news_context)  →  LLM 分析
   │   ├── 构建 System Prompt
   │   ├── 构建 User Prompt（结构化数据 + 新闻）
   │   ├── LiteLLM Router 调用 AI 模型（流式）
   │   ├── JSON 解析（json_repair 修复）
   │   ├── 内容完整性校验（可选重试）
   │   └── → AnalysisResult
   ├── 后处理（筹码补全 + 价格位置 + 决策稳定化）
   └── db.save_analysis_history(result, ...)  →  写入 SQLite
4. reset_frozen_target_date(token)  # 释放冻结
```

#### B4.4: 分析间隔延迟（可选）

```python
# main.py:519-527
if analysis_delay > 0:
    time.sleep(analysis_delay)
```

在个股分析和大盘分析之间添加延迟，避免 API 限流。默认 0（无延迟）。

#### B4.5: 大盘复盘（如果启用）

```python
# main.py:530-548
if config.market_review_enabled and not args.no_market_review:
    _run_market_review_with_shared_lock(config, run_market_review, ...)
```

大盘复盘与个股分析**共享同一套运行时组件**（notifier、analyzer、search_service），但使用独立的文件锁防止并发执行。

#### B4.6: 合并推送（Issue #190）

```python
# main.py:551-567
if merge_notification and (results or market_report):
    parts = []
    parts.append(f"# 📈 大盘复盘\n\n{market_report}")
    parts.append(f"# 🚀 个股决策仪表盘\n\n{dashboard_content}")
    combined_content = "\n\n---\n\n".join(parts)
    pipeline.notifier.send(combined_content, ...)
```

如果启用了合并推送（`merge_email_notification=true`），将大盘复盘和个股决策仪表盘合并为**一份报告**推送。

#### B4.7: 飞书云文档

```python
# main.py:582-621
feishu_doc = FeishuDocManager()
doc_url = feishu_doc.create_daily_doc(doc_title, full_content)
```

自动创建飞书每日复盘文档，标题格式 `"2026-05-19 18:00 大盘复盘"`。

#### B4.8: 自动回测（Issue #xxx）

```python
# main.py:624-641
if config.backtest_enabled:
    stats = BacktestService().run_backtest(...)
```

在每次分析完成后自动对历史数据进行回测。

## 五、模式 C: 定时任务（`--schedule`）

### 触发条件

`args.schedule=True` 或 `config.schedule_enabled=True`。

### Step C1: 构建调度参数

```python
# main.py:930-936
scheduled_stock_codes = _resolve_scheduled_stock_codes(stock_codes)  # → None
schedule_time_provider = _build_schedule_time_provider(config.schedule_time)
```

定时模式**不缓存**启动时的股票列表，每次执行前重新读取最新 `.env`。

### Step C2: 定义定时任务函数

```python
def scheduled_task():
    runtime_config = _reload_runtime_config()  # 重新加载 .env
    run_full_analysis(runtime_config, args, scheduled_stock_codes)
```

每次执行前都会：
1. 重新读取 `.env` 文件（支持 WebUI 动态修改配置）
2. 重置 Config 单例
3. 使用最新配置运行完整分析

### Step C3: 注册后台任务（可选）

如果启用了 Agent 事件监控（`agent_event_monitor_enabled=true`）：

```python
alert_worker = AlertWorker(config_provider=_reload_runtime_config)
background_tasks.append({
    "task": event_monitor_task,
    "interval_seconds": interval_minutes * 60,
    "run_immediately": True,
    "name": "agent_event_monitor",
})
```

`AlertWorker` 定期检查预设的告警规则（如价格突破、涨跌幅阈值），触发时推送通知。

### Step C4: 启动调度器

```python
# main.py:958-964
run_with_schedule(
    task=scheduled_task,
    schedule_time=config.schedule_time,    # 默认 "18:00"
    run_immediately=should_run_immediately,  # 默认 true
    background_tasks=background_tasks,
    schedule_time_provider=schedule_time_provider,
)
```

`run_with_schedule()` 内部 ([src/scheduler.py:308](src/scheduler.py#L308))：

```
1. 创建 Scheduler 实例
2. 注册后台任务（如果有）
3. 注册每日定时任务: schedule.every().day.at("18:00").do(scheduled_task)
4. 如果 run_immediately=true，立即执行一次
5. 进入主循环:
   while not shutdown:
       schedule.run_pending()     # 检查是否有任务到期
       _run_background_tasks()    # 执行后台任务
       time.sleep(30)             # 每 30 秒轮询一次
       _refresh_daily_schedule()  # 检查 SCHEDULE_TIME 是否有变化
```

**动态调度时间**：如果用户通过 WebUI 修改了 `SCHEDULE_TIME`，调度器会自动感知变更并重新注册定时任务。

**优雅退出**：收到 `SIGINT` 或 `SIGTERM` 信号后，等待当前任务完成再退出。

## 六、通知推送详细链路

### 推送入口

```python
# pipeline.py:1857-1867
if results and send_notification and not dry_run:
    self._send_notifications(results, report_type)
```

### Step N1: 生成聚合报告

```python
report = self._generate_aggregate_report(results, report_type)
```

根据 `report_type` 选择报告格式：
- `simple`（默认）：精简版决策仪表盘
- `brief`：简洁版（适合企业微信等长度受限渠道）
- `full`：完整版（包含所有分析细节）

### Step N2: 噪声控制检查

```python
noise_decision = notifier.evaluate_noise_control(
    report, route_type="report", dedup_key=noise_key, ...
)
if not noise_decision.should_send:
    return  # 跳过推送（相同内容在冷却期内）
```

去重键格式：`report:aggregate:{report_type}:{stock_codes}`

### Step N3: Markdown 转图片

```python
image_bytes = markdown_to_image(report, max_chars=...)
```

将 Markdown 报告渲染为图片（使用 `wkhtmltoimage` 或 `markdown-to-file`），部分渠道只发图片。

### Step N4: 逐个渠道推送

```python
channels = notifier.get_available_channels()
channels = notifier.get_channels_for_route("report", channels=channels)
```

各渠道推送策略：

| 渠道 | 内容 | 格式 | 特殊处理 |
|------|------|------|----------|
| 企业微信 | 精简版仪表盘 | 文本或图片 | 长度限制，必须精简 |
| 飞书 | 完整报告 | Markdown 文本 | 支持长文 |
| Telegram | 完整报告 | 文本或图片 | 可发大图 |
| Discord | 完整报告 | Markdown 文本 | 分块发送 |
| Slack | 完整报告 | 文本或图片 | Bot Token 认证 |
| 邮件 | 按股票分组 | HTML/图片 | 不同股票发到不同收件人 |
| PushPlus | 完整报告 | Markdown | 微信/短信推送 |
| Server 酱 | 完整报告 | Markdown | 微信推送 |
| ntfy | 完整报告 | 文本 | 自建推送服务 |
| Gotify | 完整报告 | 文本 | 自建推送服务 |
| Pushover | 完整报告 | 文本 | 手机推送 |
| AstrBot | 完整报告 | 文本 | QQ 机器人 |
| 自定义 Webhook | 完整报告 | JSON/文本 | 用户自定义 |
| 会话上下文 | 完整报告 | Markdown | Web UI / Bot 场景 |

**关键设计**：
- 每个渠道独立 `try/except`，单一失败不影响其他
- 企业微信**单独**处理（因为要截断为精简版）
- 邮件按股票分组发送到不同收件人（`stock_email_groups` 配置）

### Step N5: 推送后清理

```python
if success:
    notifier.record_noise_control(noise_decision)  # 记录已推送，设置冷却
```

## 七、FastAPI API 端点详解

当 Web 服务启动后，提供以下 REST API：

### 分析相关

| 端点 | 方法 | 功能 |
|------|------|------|
| `/api/v1/analysis/analyze` | POST | 触发分析任务 |
| `/api/v1/analysis/progress` | GET (SSE) | 实时任务进度流 |
| `/api/v1/analysis/status` | GET | 当前任务状态 |

### Agent 问股

| 端点 | 方法 | 功能 |
|------|------|------|
| `/api/v1/agent/chat` | POST | 发送 Agent 对话 |
| `/api/v1/agent/sessions` | GET | 获取会话列表 |
| `/api/v1/agent/strategies` | GET | 列出可用策略 |

### 历史与回测

| 端点 | 方法 | 功能 |
|------|------|------|
| `/api/v1/history/list` | GET | 历史报告列表 |
| `/api/v1/history/{id}` | GET | 单份报告详情 |
| `/api/v1/backtest/list` | GET | 回测结果列表 |
| `/api/v1/backtest/{id}` | GET | 回测详情 |

### 持仓与告警

| 端点 | 方法 | 功能 |
|------|------|------|
| `/api/v1/portfolio/list` | GET | 持仓列表 |
| `/api/v1/portfolio/add` | POST | 添加持仓 |
| `/api/v1/portfolio/remove` | POST | 移除持仓 |
| `/api/v1/alerts/list` | GET | 告警规则列表 |
| `/api/v1/alerts/create` | POST | 创建告警规则 |
| `/api/v1/alerts/delete` | POST | 删除告警规则 |

### 系统管理

| 端点 | 方法 | 功能 |
|------|------|------|
| `/api/v1/config/get` | GET | 读取当前配置 |
| `/api/v1/config/update` | POST | 更新配置（写入 `.env`） |
| `/api/v1/stocks/search` | GET | 股票搜索（代码/名称/拼音） |
| `/api/v1/health` | GET | 健康检查 |
| `/api/v1/usage/stats` | GET | LLM 用量统计 |

## 八、完整数据流全景图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     python main.py（默认单次分析模式）                       │
└──────────────────────────┬──────────────────────────────────────────────────┘
                           │
              ┌────────────▼────────────┐
              │  Phase 1: 模块初始化    │
              │  setup_env()            │
              │  加载 .env → os.environ │
              │  配置代理（可选）        │
              └────────────┬────────────┘
                           │
              ┌────────────▼────────────┐
              │  Phase 2: main() 入口   │
              │  1. parse_arguments()   │
              │  2. bootstrap 日志      │
              │  3. get_config()        │
              │  4. setup_logging()     │
              │  5. config.validate()   │
              │  6. 解析 STOCK_LIST     │
              └────────────┬────────────┘
                           │
              ┌────────────▼────────────┐
              │  Phase 3: 模式分流      │
              │                         │
              │  --check-notify → 诊断  │
              │  --backtest   → 回测    │
              │  --market-review → 复盘 │
              │  --schedule   → 定时    │
              │  --webui/--serve → Web  │
              │  默认         → 单次分析│
              └────────────┬────────────┘
                           │  ← 默认路径
              ┌────────────▼────────────┐
              │  Phase 4: run_full_analysis │
              │                         │
              │  ┌─── 交易日过滤 ───┐   │
              │  │ 600519 → A股 → 开盘?│ │
              │  │ 是 → 继续         │ │
              │  │ 否 → 跳过此股票   │ │
              │  └───────────────────┘   │
              │                         │
              │  ┌─── Pipeline 初始化 ─┐│
              │  │ db (SQLite)         ││
              │  │ DataFetcherManager  ││
              │  │ StockTrendAnalyzer  ││
              │  │ GeminiAnalyzer      ││
              │  │ NotificationService ││
              │  │ SearchService       ││
              │  │ SocialSentimentSvc  ││
              │  └─────────────────────┘│
              │                         │
              │  ┌─── 预取阶段 ────────┐│
              │  │ prefetch_names()    ││
              │  │ prefetch_quotes()   ││
              │  └─────────────────────┘│
              └────────────┬────────────┘
                           │ ThreadPoolExecutor(max_workers=3)
              ┌────────────▼────────────┐
              │  Phase 5: process_single_stock │
              │                         │
              │  ┌─ 1. 数据获取与保存 ─┐│
              │  │ get_stock_name()    ││
              │  │   → "贵州茅台"       ││
              │  │                     ││
              │  │ has_today_data()?   ││
              │  │   → True: 跳过获取   ││
              │  │   → False: 获取     ││
              │  │                     ││
              │  │ get_daily_data()    ││
              │  │   故障切换:          ││
              │  │   Efinance → AkShare││
              │  │   → Pytdx → Baostock││
              │  │   → YFinance        ││
              │  │                     ││
              │  │ 输出: DataFrame     ││
              │  │ [date, OHLCV, MA]   ││
              │  │                     ││
              │  │ save_daily_data()   ││
              │  │   → 写入 SQLite     ││
              │  └─────────────────────┘│
              │                         │
              │  ┌─ 2. AI 分析 ───────┐│
              │  │                     ││
              │  │ 2.1 实时行情        ││
              │  │   get_realtime_quote││
              │  │   → price, volume_  ││
              │  │     ratio, turnover ││
              │  │     _rate, PE, PB   ││
              │  │                     ││
              │  │ 2.2 筹码分布        ││
              │  │   get_chip_dist()   ││
              │  │   → profit_ratio,   ││
              │  │     concentration   ││
              │  │   (熔断器保护)       ││
              │  │                     ││
              │  │ 2.3 基本面聚合      ││
              │  │   get_fundamental() ││
              │  │   7 模块块:          ││
              │  │   valuation         ││
              │  │   growth            ││
              │  │   earnings          ││
              │  │   institution       ││
              │  │   capital_flow      ││
              │  │   dragon_tiger      ││
              │  │   boards            ││
              │  │   (fail-open)       ││
              │  │                     ││
              │  │ 2.4 趋势分析        ││
              │  │   trend_analyzer    ││
              │  │   .analyze(df)      ││
              │  │   → 趋势状态        ││
              │  │   → 买卖信号        ││
              │  │   → 信号评分        ││
              │  │   → 支撑/压力位     ││
              │  │                     ││
              │  │ 2.5 新闻搜索        ││
              │  │   search_intel()    ││
              │  │   最多 5 次搜索:     ││
              │  │   最新消息           ││
              │  │   风险排查           ││
              │  │   业绩预期           ││
              │  │   行业动态           ││
              │  │   政策影响           ││
              │  │                     ││
              │  │ 2.6 增强上下文      ││
              │  │   _enhance_context()││
              │  │   合并所有数据       ││
              │  │                     ││
              │  │ 2.7 LLM 分析        ││
              │  │   analyzer.analyze()││
              │  │   ├── System Prompt ││
              │  │   ├── User Prompt   ││
              │  │   ├── LiteLLM       ││
              │  │   │   Router        ││
              │  │   │   (流式)        ││
              │  │   ├── JSON 解析     ││
              │  │   ├── 完整性校验    ││
              │  │   └── AnalysisResult││
              │  │                     ││
              │  │ 2.8 后处理          ││
              │  │   筹码补全           ││
              │  │   价格位置           ││
              │  │   决策稳定化         ││
              │  │                     ││
              │  │ 2.9 保存历史        ││
              │  │   save_analysis_    ││
              │  │   history()         ││
              │  └─────────────────────┘│
              └────────────┬────────────┘
                           │ results = [AnalysisResult]
              ┌────────────▼────────────┐
              │  Phase 6: 结果处理       │
              │                         │
              │  6.1 统计输出           │
              │  6.2 保存本地报告        │
              │  6.3 噪声控制检查        │
              │  6.4 Markdown→图片     │
              │  6.5 多渠道推送         │
              │  6.6 飞书文档           │
              │  6.7 自动回测           │
              └────────────┬────────────┘
                           │
              ┌────────────▼────────────┐
              │  Phase 7: 输出摘要       │
              │  ===== 分析结果摘要 =====│
              │  ⚪ 贵州茅台(600519):    │
              │     观望 | 评分 72      │
              │  程序执行完成            │
              └─────────────────────────┘
```

## 九、七种核心设计模式

### 9.1 策略模式 + 故障切换（数据源）

```
DataFetcherManager 按优先级排序数据源列表
  ↓
逐个尝试，成功即返回
  ↓
失败记录原因，自动切换到下一个
  ↓
全部失败则抛出详细异常
```

不同市场有**专用路由**：
- 美股：Finnhub → AlphaVantage → YFinance → Longbridge
- 港股：Longbridge → YFinance → AkShare
- A 股：Efinance → AkShare → Pytdx → Baostock → YFinance

### 9.2 断点续传

```
db.has_today_data(code, target_date)
  ↓ True   → 跳过网络请求，使用数据库已有数据
  ↓ False  → 从数据源获取 → 保存到数据库
```

跨市场使用 `frozen_target_date` 统一目标交易日，避免同批股票使用不同日期。

### 9.3 Fail-Open 语义

以下模块采用 fail-open（失败不阻塞整体流程）：

| 模块 | 失败行为 |
|------|----------|
| 实时行情 | 降级为历史收盘价 |
| 筹码分布 | 跳过筹码分析 |
| 基本面聚合 | 标记为 `failed`，继续其他模块 |
| 新闻搜索 | 仅基于技术面分析 |
| 社交舆情 | 跳过（仅美股） |
| 飞书文档 | 忽略错误 |
| 自动回测 | 忽略错误 |

### 9.4 线程池并发 + 故障隔离

```
ThreadPoolExecutor(max_workers=N)
  ┌─────────┐ ┌─────────┐ ┌─────────┐
  │ 股票A    │ │ 股票B    │ │ 股票C    │
  │ 独立线程 │ │ 独立线程 │ │ 独立线程 │
  └────┬─────┘ └────┬─────┘ └────┬─────┘
       │             │             │
    成功/失败      成功/失败      成功/失败
       │             │             │
       └──────┬──────┘──────┬──────┘
              ▼             ▼
         results 列表    继续下一个
```

每个 `process_single_stock()` 调用被 `try/except` 包裹，异常不会传播到其他线程。

### 9.5 多模型 Fallback（LLM 层）

```
LiteLLM Router 配置多个模型
  ↓
优先使用主模型（如 Gemini）
  ↓ 失败
切换到备用模型（如 Claude）
  ↓ 失败
使用兜底模型（如 OpenAI）
  ↓ 全部失败
返回默认 AnalysisResult（评分 50，建议"观望"）
```

### 9.6 熔断器模式

筹码分布每个数据源独立熔断器：

```
连续失败 N 次 → OPEN（熔断）
  ↓
等待 M 分钟后 → HALF_OPEN（试探）
  ↓
试探成功 → CLOSED（恢复）
试探失败 → OPEN（继续熔断）
```

### 9.7 通知去重与噪声控制

```
去重键: report:aggregate:{type}:{codes}
  ↓
检查冷却期（默认 30 分钟）
  ↓
冷却期内 → 跳过推送
冷却期外 → 推送 → 记录时间 → 设置新冷却期
```

## 十、数据库表结构（SQLite）

| 表名 | 用途 | 关键字段 |
|------|------|----------|
| `daily_data` | 日线行情数据 | code, date, open, high, low, close, volume, amount, pct_chg, ma5, ma10, ma20 |
| `analysis_history` | 分析历史记录 | code, name, query_id, sentiment_score, trend_prediction, operation_advice, report, created_at |
| `news_intel` | 新闻情报 | code, dimension, query, results, created_at |
| `fundamental_snapshot` | 基本面快照 | query_id, code, payload, source_chain, coverage, created_at |
| `backtest_results` | 回测结果 | code, accuracy, total_predictions, correct_predictions |
| `usage_records` | LLM 用量 | model, call_type, stock_code, input_tokens, output_tokens, cost |
| `query_context` | 查询上下文 | query_id, query_source, platform, user_id |

## 十一、关键时间节点（典型耗时）

以分析 1 只 A 股（如 `600519`）为例：

| 阶段 | 耗时 | 备注 |
|------|------|------|
| 启动 + 配置加载 | < 1s | 内存操作 |
| 交易日检查 | < 100ms | 本地查询 |
| 名称预取 | ~200ms | 轻量查询 |
| 日线数据获取 | 1-5s | 取决于数据源 |
| 实时行情 | 1-3s | 网络请求 |
| 筹码分布 | 1-3s | 可能被熔断跳过 |
| 基本面聚合 | 5-15s | 受预算超时控制 |
| 趋势分析 | < 100ms | 本地计算 |
| 新闻搜索 | 5-15s | 最多 5 次搜索 |
| LLM 分析 | 10-60s | 取决于模型 |
| 结果解析 + 保存 | < 500ms | 本地操作 |
| 通知推送 | 1-5s | 取决于渠道数 |
| **总计（单股）** | **30-120s** | 因配置和网络而异 |

分析多只股票时，并发执行（默认 3 线程），总耗时 ≈ 最慢那只股票的时间。

## 十二、常见场景速查

### 场景 1: 只想分析特定股票

```bash
python main.py --stocks 600519,hk00700,AAPL
```

### 场景 2: 测试数据获取是否正常

```bash
python main.py --dry-run
```

不消耗 LLM 配额，仅测试数据源链路。

### 场景 3: 检查通知配置

```bash
python main.py --check-notify
```

打印所有通知渠道的诊断结果。

### 场景 4: 不发送推送，仅保存报告

```bash
python main.py --no-notify
```

### 场景 5: 每天 18:00 自动分析

```bash
python main.py --schedule
```

或使用 Docker 后台运行：
```bash
docker run -d daily_stock_analysis --schedule
```

### 场景 6: 启动 Web 工作台

```bash
python main.py --webui
# 访问 http://127.0.0.1:8000
```

### 场景 7: 节假日强制执行

```bash
python main.py --force-run
```

### 场景 8: 仅大盘复盘

```bash
python main.py --market-review
```

---

> 本文档覆盖了 `python main.py` 的全部执行路径。如需理解某个具体环节的数据流，可参考对应的模块代码：
> - 数据源层: `data_provider/`
> - 分析器: `src/analyzer.py`
> - Pipeline: `src/core/pipeline.py`
> - 调度器: `src/scheduler.py`
> - 通知服务: `src/notification.py` + `src/notification_sender/`
> - API 服务: `api/app.py`
> - 数据库: `src/storage.py`
