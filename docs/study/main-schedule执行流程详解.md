# `python main.py --schedule` 执行流程详解

## 一、整体架构概览

```
main.py (--schedule)
  ├── 1. 环境初始化 & 配置加载
  ├── 2. 日志系统初始化
  ├── 3. 配置验证 & 特殊模式检查
  ├── 4. Web 服务启动（可选）
  ├── 5. 进入定时任务模式
  │     ├── Scheduler 调度器
  │     │     ├── 每日定时任务 (daily job)
  │     │     └── 后台周期任务 (background tasks)
  │     └── scheduled_task() 回调
  │           └── run_full_analysis()
  │                 ├── 个股分析 Pipeline
  │                 ├── 大盘复盘
  │                 ├── 合并推送
  │                 ├── 飞书文档生成
  │                 └── 自动回测
```

---

## 二、启动阶段（main 函数入口）

### 2.1 命令行参数解析

```
main() → parse_arguments()
```

解析 `--schedule` 标志，同时兼容以下相关参数：
- `--no-run-immediately`: 启动时不立即执行一次
- `--no-notify`: 不发送推送通知
- `--single-notify`: 单股推送模式
- `--no-market-review`: 跳过大盘复盘
- `--force-run`: 跳过交易日检查
- `--debug`: 调试模式
- `--workers N`: 并发线程数

### 2.2 环境引导（Import-time）

模块加载时已执行：
1. `_INITIAL_PROCESS_ENV` 保存进程初始环境变量快照
2. `setup_env()` 加载 `.env` 文件
3. 代理配置（若 `USE_PROXY=true` 且非 GitHub Actions 环境）

### 2.3 日志系统初始化

两步走策略：

| 步骤 | 函数 | 作用 |
|------|------|------|
| Bootstrap 日志 | `_setup_bootstrap_logging()` | 仅输出到 stderr，确保早期错误有日志 |
| 配置日志 | `_setup_runtime_logging()` | 输出到 `config.log_dir` 目录的文件 + 控制台 |

### 2.4 配置加载与验证

```python
config = get_config()       # 从 .env 加载 Config 单例
warnings = config.validate() # 验证配置完整性
```

### 2.5 通知诊断（可选）

若传入 `--check-notify`，运行通知渠道诊断并退出：
```python
run_notification_diagnostics(config) → print 结果 → return
```

---

## 三、定时任务模式入口

### 3.1 模式判定

```python
if args.schedule or config.schedule_enabled:
    # 进入定时任务模式
```

两个触发条件（任一即可）：
- 命令行 `--schedule` 参数
- `.env` 中 `SCHEDULE_ENABLED=true`

### 3.2 关键配置项

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `SCHEDULE_TIME` | `"18:00"` | 每日执行时间 |
| `SCHEDULE_RUN_IMMEDIATELY` | `true` | 启动时是否立即执行一次 |
| `STOCK_LIST` | 逗号分隔代码 | 自选股列表 |

### 3.3 股票列表解析策略

```python
_resolve_scheduled_stock_codes(stock_codes) → 始终返回 None
```

**关键设计**：定时模式**忽略**启动时的股票快照，每次执行前从 `.env` **重新读取**最新的 `STOCK_LIST`。这样 WebUI 修改自选股后无需重启即可生效（Issue #529）。

### 3.4 调度时间提供器

```python
_build_schedule_time_provider(config.schedule_time)
```

三级回退策略：
1. 进程级环境变量 `SCHEDULE_TIME`（启动前设置）
2. `.env` 文件中的持久化值（WebUI 可修改）
3. 系统默认 `"18:00"`

**关键特性**：调度器每轮循环检查前读取最新值，支持运行时动态修改执行时间（无需重启）。

---

## 四、Scheduler 调度器核心机制

### 4.1 创建调度器

```python
from src.scheduler import run_with_schedule

run_with_schedule(
    task=scheduled_task,
    schedule_time=config.schedule_time,
    run_immediately=should_run_immediately,
    background_tasks=background_tasks,
    schedule_time_provider=schedule_time_provider,
)
```

### 4.2 Scheduler 内部结构

```
Scheduler
  ├── schedule (schedule 库)
  │     └── every().day.at("HH:MM").do(_safe_run_task)
  ├── GracefulShutdown (SIGTERM/SIGINT 信号处理)
  ├── _daily_job (每日定时任务)
  └── _background_tasks[] (后台周期任务列表)
```

### 4.3 调度主循环

```python
while self._running and not self.shutdown_handler.should_shutdown:
    self._refresh_daily_schedule_if_needed()   # 检查 SCHEDULE_TIME 是否变更
    self.schedule.run_pending()                 # 执行到点的定时任务
    self._run_background_tasks()                # 执行到间隔的后台任务
    time.sleep(30)                              # 每 30 秒轮询一次
```

**心跳日志**：每小时整点（minute==0 && second<30）打印一次 "调度器运行中... 下次执行: XX:XX:XX"。

### 4.4 优雅退出

```python
class GracefulShutdown:
    # 注册 SIGINT / SIGTERM 信号
    # 收到信号后设置 shutdown_requested = True
    # 当前任务完成后循环自然退出
```

### 4.5 后台任务（Background Tasks）

在 schedule 模式下，可能注册的后台任务：

| 任务名 | 触发条件 | 间隔 | 说明 |
|--------|----------|------|------|
| `agent_event_monitor` | `agent_event_monitor_enabled=true` | `agent_event_monitor_interval_minutes`（默认 5 分钟） | 事件监控 & 告警触发 |

后台任务运行机制：
- 每个任务在独立 daemon 线程中执行
- 间隔 < 30s 自动钳制为 30s（匹配主循环轮询频率）
- 同一任务不会重叠执行（已有线程运行时跳过）

---

## 五、定时任务执行体：`scheduled_task()`

这是每次定时触发的核心回调：

```python
def scheduled_task():
    runtime_config = _reload_runtime_config()   # 重新加载 .env 配置
    run_full_analysis(runtime_config, args, scheduled_stock_codes)
```

### 5.1 配置热重载

```python
_reload_runtime_config()
  ├── _reload_env_file_values_preserving_overrides()  # 刷新 .env 环境变量
  ├── Config.reset_instance()                         # 重置 Config 单例
  └── get_config()                                    # 重新获取配置
```

**目的**：每次定时执行都读取最新的 `.env` 配置，支持运行时修改股票列表、通知配置等。

---

## 六、完整分析流程：`run_full_analysis()`

这是整个系统的核心执行函数，包含三大阶段：

### 6.1 交易日过滤（Issue #373）

```python
_compute_trading_day_filter(config, args, stock_codes)
  → (filtered_codes, effective_region, should_skip_all)
```

**逻辑**：
1. 获取今日开盘市场列表 `get_open_markets_today()`
2. 逐个检查股票所属市场是否开盘
3. 非交易日股票自动跳过（打印 "今日休市股票已跳过: {...}"）
4. 计算大盘复盘的有效区域（如 CN/HK/US 的组合）
5. 若**所有股票均非交易日**且**大盘复盘区域为空** → 跳过整轮执行

**跳过条件**：
- `--force-run` 强制执行时跳过此检查
- `trading_day_check_enabled=false` 时跳过此检查

### 6.2 个股分析 Pipeline

```python
pipeline = StockAnalysisPipeline(
    config=config,
    max_workers=args.workers,           # 默认从配置读取
    query_id=uuid.uuid4().hex,          # 唯一查询 ID
    query_source="cli",                 # 来源标记
)

results = pipeline.run(
    stock_codes=stock_codes,
    dry_run=args.dry_run,
    send_notification=not args.no_notify,
    merge_notification=merge_notification,
)
```

#### 6.2.1 Pipeline.run() 详细流程

```
pipeline.run()
  ├── 刷新 STOCK_LIST（从 .env 重新读取）
  ├── 冻结参考时间 resume_reference_time（统一断点续传判断）
  ├── 批量预取实时行情（股票 >= 5 只时启用）
  ├── 预取股票名称（非 dry_run 时）
  ├── 读取报告类型（simple/brief/full）
  └── ThreadPoolExecutor(max_workers=N) 并发处理
        │
        ├─ 线程1: process_single_stock(code1)
        ├─ 线程2: process_single_stock(code2)
        └─ 线程3: process_single_stock(code3)
```

#### 6.2.2 单股处理流程 `process_single_stock()`

```
process_single_stock(code)
  ├── 1. 冻结目标交易日 set_frozen_target_date()
  ├── 2. fetch_and_save_stock_data()
  │     ├── 检查断点续传: has_today_data(code, target_date)
  │     │     └── 已有数据 → 跳过网络请求
  │     └── 否则: get_daily_data() → save_daily_data()
  ├── 3. analyze_stock()（非 dry_run 时）
  │     │
  │     ├── Step 1: 获取实时行情（量比、换手率）
  │     │     └── get_realtime_quote() → 自动故障切换
  │     ├── Step 2: 获取筹码分布
  │     │     └── get_chip_distribution() → 带熔断保护
  │     ├── Step 2.5: 基本面聚合
  │     │     └── get_fundamental_context() → 异常降级为 failed
  │     ├── Step 3: 趋势分析（技术分析引擎）
  │     │     └── trend_analyzer.analyze() → 均线/量价/信号
  │     │
  │     ├── 【分支 A: Agent 模式】（agent_mode=true 或配置了 skills）
  │     │     ├── build_agent_executor()
  │     │     ├── _ensure_agent_history() → 预取 240+ 天 K 线
  │     │     ├── executor.run(message) → LLM Agent 分析
  │     │     ├── _agent_result_to_analysis_result() → 格式转换
  │     │     │     └── 缺失字段 → trend 引擎 fallback
  │     │     └── save_analysis_history()
  │     │
  │     └── 【分支 B: 传统模式】
  │           ├── Step 4: 多维度情报搜索
  │           │     └── search_comprehensive_intel() → 最多 5 次搜索
  │           ├── Step 4.5: 社交舆情（美股）
  │           │     └── get_social_context() → Reddit/X/Polymarket
  │           ├── Step 5: 获取分析上下文（技术面历史数据）
  │           ├── Step 6: 增强上下文（实时行情+筹码+趋势+基本面）
  │           ├── Step 7: AI 分析（LLM 调用）
  │           │     └── analyzer.analyze() → 带流式进度回调
  │           └── Step 8: 保存分析历史
  │
  ├── 4. 单股推送（single_stock_notify=true 时）
  │     └── _send_single_stock_notification() → 加锁串行发送
  └── 5. 重置目标交易日 reset_frozen_target_date()
```

#### 6.2.3 并发控制与结果收集

```python
with ThreadPoolExecutor(max_workers=N) as executor:
    # 提交所有任务
    future_to_code = {executor.submit(...): code for code in stock_codes}

    # 按完成顺序收集结果
    for future in as_completed(future_to_code):
        result = future.result()
        if result and result.success:
            results.append(result)
            # 单股推送模式：串行发送通知（加锁）
```

**关键设计**：
- `max_workers` 默认 3，低并发避免触发数据源反爬
- `analysis_delay` 配置可在每只股票间添加延迟（避免 API 限流）
- 单股异常不影响其他股票（try/except 包裹）

### 6.3 个股分析与大盘复盘之间的延迟

```python
analysis_delay = getattr(config, 'analysis_delay', 0)
if analysis_delay > 0:
    time.sleep(analysis_delay)  # 避免 API 限流
```

### 6.4 大盘复盘（Market Review）

```python
if config.market_review_enabled and not args.no_market_review:
    _run_market_review_with_shared_lock(
        config,
        run_market_review,
        notifier=pipeline.notifier,
        analyzer=pipeline.analyzer,
        search_service=pipeline.search_service,
        override_region=effective_region,
    )
```

**锁机制**（`market_review_lock`）：
- `try_acquire_market_review_lock()` → 获取分布式锁
- 若已有实例在执行中 → 跳过本次大盘复盘
- 执行完成后释放锁
- **目的**：防止多个调度实例/容器同时运行大盘复盘

### 6.5 合并推送（Issue #190）

```python
merge_notification = (
    config.merge_email_notification
    and config.market_review_enabled
    and not args.no_market_review
    and not config.single_stock_notify
)

if merge_notification and (results or market_report):
    # 拼接: 大盘复盘 + 个股仪表盘
    combined_content = "📈 大盘复盘\n\n...\n\n---\n\n🚀 个股决策仪表盘\n\n..."
    pipeline.notifier.send(combined_content, route_type="report")
```

**触发条件**（全部满足）：
- `merge_email_notification=true`
- `market_review_enabled=true`
- 未传入 `--no-market-review`
- 非单股推送模式

### 6.6 飞书云文档生成

```python
from src.feishu_doc import FeishuDocManager

feishu_doc = FeishuDocManager()
if feishu_doc.is_configured() and (results or market_report):
    doc_title = f"{YYYY-MM-DD HH:MM} 大盘复盘"
    full_content = "📈 大盘复盘\n\n...\n\n---\n\n🚀 个股决策仪表盘\n\n..."
    doc_url = feishu_doc.create_daily_doc(doc_title, full_content)
```

**流程**：
1. 生成带时间戳的文档标题
2. 拼接大盘复盘 + 个股仪表盘内容
3. 调用飞书 API 创建云文档
4. 可选：将文档链接推送到群里

### 6.7 自动回测（Auto Backtest）

```python
if getattr(config, 'backtest_enabled', False):
    service = BacktestService()
    stats = service.run_backtest(
        force=False,
        eval_window_days=config.backtest_eval_window_days,
        min_age_days=config.backtest_min_age_days,
        limit=200,
    )
```

**触发条件**：`BACKTEST_ENABLED=true`
- 对历史分析结果进行评估
- 验证分析建议的准确性
- 失败时静默忽略（不影响主流程）

---

## 七、完整数据流图

```
┌─────────────────────────────────────────────────────┐
│                    .env 配置文件                      │
│  STOCK_LIST, SCHEDULE_TIME, 各种 API Key 等          │
└──────────────────┬──────────────────────────────────┘
                   │ get_config()
                   ▼
┌─────────────────────────────────────────────────────┐
│                   Config 单例                         │
│  stock_list, schedule_time, max_workers, 等          │
└──────────────────┬──────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────┐
│              Scheduler 定时调度器                      │
│  ┌─────────────────┐  ┌──────────────────────────┐  │
│  │ daily job       │  │ background tasks         │  │
│  │ every().day.at()│  │ event_monitor (可选)     │  │
│  └────────┬────────┘  └──────────────────────────┘  │
└───────────┼─────────────────────────────────────────┘
            │ 定时触发
            ▼
┌─────────────────────────────────────────────────────┐
│          scheduled_task() 回调                        │
│  1. _reload_runtime_config()  ← 重新读取 .env         │
│  2. run_full_analysis()                              │
└──────────────────┬──────────────────────────────────┘
                   │
          ┌────────┴────────┐
          ▼                 ▼
┌──────────────────┐  ┌──────────────────────────────┐
│ 交易日过滤        │  │  大盘复盘（如果启用）           │
│ 跳过非交易日股票   │  │  run_market_review() + 锁      │
└────────┬─────────┘  └──────────┬───────────────────┘
         │                       │
         ▼                       │
┌────────────────────────────────────────────────────┐
│          StockAnalysisPipeline.run()                 │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  ThreadPoolExecutor(max_workers=N)           │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌────────┐│   │
│  │  │ process     │ │ process     │ │process ││   │
│  │  │ single_stock│ │ single_stock│ │ ...    ││   │
│  │  │  ┌────────┐ │ │  ┌────────┐ │ │        ││   │
│  │  │  │ 获取数据│ │ │  │ 获取数据│ │ │        ││   │
│  │  │  │ ↓      │ │ │  │ ↓      │ │ │        ││   │
│  │  │  │ AI分析 │ │ │  │ AI分析 │ │ │        ││   │
│  │  │  └────────┘ │ │  └────────┘ │ │        ││   │
│  │  └─────────────┘ └─────────────┘ └────────┘│   │
│  │         ↓ 结果收集 ↓                         │   │
│  │  [AnalysisResult, AnalysisResult, ...]       │   │
│  └─────────────────────────────────────────────┘   │
│         ↓                                           │
│  _send_notifications() / 合并推送                    │
│  _save_local_report()                               │
└──────────────────┬──────────────────────────────────┘
                   │
          ┌────────┴────────┐
          ▼                 ▼
┌──────────────────┐  ┌──────────────────────┐
│ 飞书文档生成       │  │ 自动回测（如果启用）    │
│ create_daily_doc │  │ run_backtest()       │
└──────────────────┘  └──────────────────────┘
          │                 │
          ▼                 ▼
┌─────────────────────────────────────────────────────┐
│                   推送通知                            │
│  企业微信 / 邮件 / Telegram / 飞书 / Discord / ...    │
│  （或合并推送：大盘 + 个股）                           │
└─────────────────────────────────────────────────────┘
```

---

## 八、关键时序图

```
时间 ───────────────────────────────────────────────────────►

T+0     程序启动
        ├── 加载 .env → Config
        ├── 初始化日志
        └── 创建 Scheduler

T+0.1   ┌─ 如果 run_immediately=true ──────────────┐
        │  scheduled_task()                         │
        │  ├── reload .env                           │
        │  ├── 交易日过滤                            │
        │  ├── pipeline.run() → 个股分析             │
        │  ├── 大盘复盘                              │
        │  ├── 合并推送                              │
        │  ├── 飞书文档                              │
        │  └── 自动回测                              │
        └───────────────────────────────────────────┘

T+X     进入主循环 (每 30s 检查一次)
        ├── [30s] run_pending? → 否
        ├── [30s] background_tasks? → 执行
        ├── [60s] run_pending? → 否
        ├── ...
        └── [HH:MM] run_pending? → 是!
                    │
                    ▼
              scheduled_task() （同 T+0.1 流程）

T+∞     收到 SIGTERM/SIGINT
        ├── shutdown_requested = True
        ├── 等待当前任务完成
        └── 退出循环
```

---

## 九、配置项速查表

### 9.1 调度相关

| 环境变量 | 默认值 | 说明 |
|----------|--------|------|
| `SCHEDULE_ENABLED` | `false` | 是否启用定时任务 |
| `SCHEDULE_TIME` | `"18:00"` | 每日执行时间 (HH:MM) |
| `SCHEDULE_RUN_IMMEDIATELY` | `true` | 启动时立即执行一次 |

### 9.2 并发控制

| 环境变量 | 默认值 | 说明 |
|----------|--------|------|
| `MAX_WORKERS` | `3` | 并发线程数 |
| `ANALYSIS_DELAY` | `0` | 个股间延迟（秒） |

### 9.3 交易日历

| 环境变量 | 默认值 | 说明 |
|----------|--------|------|
| `TRADING_DAY_CHECK_ENABLED` | `true` | 是否检查交易日 |

### 9.4 通知相关

| 环境变量 | 默认值 | 说明 |
|----------|--------|------|
| `MERGE_EMAIL_NOTIFICATION` | `false` | 合并大盘+个股推送 |
| `SINGLE_STOCK_NOTIFY` | `false` | 单股推送模式 |

### 9.5 大盘复盘

| 环境变量 | 默认值 | 说明 |
|----------|--------|------|
| `MARKET_REVIEW_ENABLED` | `true` | 是否启用大盘复盘 |
| `MARKET_REVIEW_REGION` | `"cn"` | 复盘区域 (cn/hk/us) |

### 9.6 回测

| 环境变量 | 默认值 | 说明 |
|----------|--------|------|
| `BACKTEST_ENABLED` | `false` | 是否启用自动回测 |
| `BACKTEST_EVAL_WINDOW_DAYS` | `10` | 回测评估窗口 |
| `BACKTEST_MIN_AGE_DAYS` | `14` | 最小分析年龄 |

---

## 十、异常处理与容错机制

| 场景 | 处理策略 | 影响范围 |
|------|----------|----------|
| 单股数据获取失败 | 降级使用已有数据分析 | 仅影响该股票 |
| 单股 AI 分析失败 | 记录日志，继续处理下一只 | 仅影响该股票 |
| 搜索服务不可用 | 跳过新闻搜索，继续分析 | 仅缺少新闻上下文 |
| 推送通知失败 | 记录警告，不影响其他推送 | 仅该渠道 |
| 大盘复盘锁冲突 | 跳过本次大盘复盘 | 仅大盘复盘 |
| 飞书文档创建失败 | 记录错误，不影响主流程 | 仅飞书文档 |
| 自动回测失败 | 静默忽略 | 仅回测 |
| 交易日检查失败 | `--force-run` 强制跳过 | 全局 |
| 配置重载失败 | 使用当前 Config 单例 | 全局 |
| 调度器收到 SIGTERM | 等待当前任务完成后退出 | 全局 |

**核心原则**：单个环节失败不阻断整个分析流程（fail-open 设计）。

---

## 十一、文件引用索引

| 文件 | 职责 |
|------|------|
| [main.py](main.py) | 主入口、参数解析、模式分发 |
| [src/scheduler.py](src/scheduler.py) | 定时调度器、主循环、信号处理 |
| [src/core/pipeline.py](src/core/pipeline.py) | 核心分析流水线、线程池调度 |
| [src/config.py](src/config.py) | 配置管理（Config 单例） |
| [src/core/trading_calendar.py](src/core/trading_calendar.py) | 交易日历判断 |
| [src/core/market_review_lock.py](src/core/market_review_lock.py) | 大盘复盘分布式锁 |
| [src/core/market_review.py](src/core/market_review.py) | 大盘复盘逻辑 |
| [src/core/config_manager.py](src/core/config_manager.py) | 配置热重载 |
| [src/feishu_doc.py](src/feishu_doc.py) | 飞书云文档生成 |
| [src/services/backtest_service.py](src/services/backtest_service.py) | 回测服务 |
| [src/services/alert_worker.py](src/services/alert_worker.py) | 事件监控后台任务 |
