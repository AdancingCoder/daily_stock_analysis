# `python main.py --stocks 600519` 完整执行流程与数据流

> 本文档逐行追踪一条命令从输入到输出的完整生命周期，帮助开发者理解各模块如何协作。

## 一、启动阶段（main.py）

### Step 1: 解析命令行参数

```
python main.py --stocks 600519
```

`parse_arguments()` ([main.py:221](main.py#L221)) 解析后得到：

| 参数 | 值 | 说明 |
|------|-----|------|
| `--stocks` | `"600519"` | 指定分析目标 |
| `--debug` | `False` | 默认关闭 |
| `--dry-run` | `False` | 默认关闭 |
| `--no-notify` | `False` | 默认发送通知 |
| `--serve/--webui` | `False` | 不启动 Web |
| `--schedule` | `False` | 不是定时模式 |
| `--backtest` | `False` | 不是回测模式 |
| `--market-review` | `False` | 不是大盘复盘 |

### Step 2: 环境初始化

```python
# main.py:31-33
setup_env()          # 加载 .env 文件到环境变量
```

**配置加载顺序**：`.env 文件` → `进程环境变量` → `Config 单例`

### Step 3: 加载全局配置

```python
# main.py:779
config = get_config()   # 返回 Config 单例
```

`Config` ([src/config.py](src/config.py)) 从环境变量读取所有配置项：
- AI 模型配置：`LITELLM_MODEL`, `GEMINI_API_KEY`, `OPENAI_API_KEY` 等
- 数据源配置：`TUSHARE_TOKEN`, `REALTIME_SOURCE_PRIORITY` 等
- 通知配置：`WECHAT_WEBHOOK_URL`, `FEISHU_WEBHOOK_URL` 等
- 自选股：`STOCK_LIST`（本例中被 `--stocks 600519` 覆盖）
- 并发数：`MAX_WORKERS`（默认 3）

### Step 4: 初始化 Bootstrap 日志

```python
# main.py:768
_setup_bootstrap_logging(debug=args.debug)
```

### Step 5: 选择运行模式

`main()` ([main.py:756](main.py#L756)) 按优先级判断：

```
1. --backtest     → 回测模式
2. --market-review → 仅大盘复盘
3. --schedule     → 定时任务
4. 默认           → 单次运行（本例走此分支）
```

本例走 **模式3: 正常单次运行** ([main.py:968](main.py#L968))：

```python
# main.py:968
run_full_analysis(config, args, stock_codes)
```

其中 `stock_codes = ["600519"]`（由 `--stocks` 解析）。

## 二、完整分析阶段（run_full_analysis）

### Step 6: 交易日过滤

```python
# main.py:471-483
_compute_trading_day_filter(config, args, ["600519"])
```

- 检查 600519 所属市场（A股）今日是否开盘
- 如果是非交易日且未设置 `--force-run`，该股票会被跳过
- 如果跳过所有股票且大盘复盘也不需要执行，整个流程提前返回

### Step 7: 创建 Pipeline

```python
# main.py:502-508
pipeline = StockAnalysisPipeline(
    config=config,
    max_workers=args.workers,   # 默认从配置读取，通常为 3
    query_id=uuid.uuid4().hex,  # 生成唯一查询 ID
    query_source="cli",         # 来源标记
    save_context_snapshot=None  # 使用配置默认值
)
```

**Pipeline 初始化各模块** ([pipeline.py:78](src/core/pipeline.py#L78))：

| 模块 | 代码位置 | 职责 |
|------|----------|------|
| `self.db` | `get_db()` | 数据库访问（SQLite） |
| `self.fetcher_manager` | `DataFetcherManager()` | 多数据源管理器 |
| `self.trend_analyzer` | `StockTrendAnalyzer()` | 技术指标分析（均线/MACD等） |
| `self.analyzer` | `GeminiAnalyzer()` | LLM 分析器（调用 AI） |
| `self.notifier` | `NotificationService()` | 通知推送服务 |
| `self.search_service` | `SearchService()` | 新闻/舆情搜索服务 |
| `self.social_sentiment_service` | `SocialSentimentService()` | 社交舆情（仅美股） |

### Step 8: 执行 Pipeline.run()

```python
# main.py:511-516
results = pipeline.run(
    stock_codes=["600519"],
    dry_run=False,
    send_notification=True,
    merge_notification=False
)
```

## 三、Pipeline.run() 并发调度

### Step 9: 预取数据

```python
# pipeline.py:1756
self.fetcher_manager.prefetch_stock_names(["600519"], use_bulk=False)
```

轻量预取股票名称到内存缓存，避免并发分析时显示「股票600519」。

### Step 10: 线程池并发执行

```python
# pipeline.py:1782-1795
with ThreadPoolExecutor(max_workers=self.max_workers) as executor:
    future_to_code = {
        executor.submit(
            self.process_single_stock,
            "600519",
            skip_analysis=False,     # dry_run=False
            single_stock_notify=False,
            report_type=ReportType.SIMPLE,
            analysis_query_id=uuid.uuid4().hex,
            current_time=resume_reference_time,
        ): "600519"
    }
```

- 单只股票时只有一个线程
- 多只股票时并发执行（默认 max_workers=3）
- 每只股票故障隔离，单股失败不影响其他

## 四、单股处理流程（process_single_stock）

### Step 11: 冻结目标交易日

```python
# pipeline.py:1655
frozen_td = self._resolve_resume_target_date("600519", current_time=...)
```

确定本轮分析的目标交易日，用于断点续传判断。

### Step 12: 获取并保存行情数据

```python
# pipeline.py:1659
self.fetch_and_save_stock_data("600519", current_time=...)
```

#### 12.1: 获取股票名称

```python
# pipeline.py:215
self.fetcher_manager.get_stock_name("600519", allow_realtime=False)
```

名称获取链路：
```
内存缓存 → STOCK_NAME_MAP 硬编码映射 → stocks.index.json 索引
→ 实时行情查询 → 各数据源 get_stock_name 方法
```

#### 12.2: 断点续传检查

```python
# pipeline.py:222
if not force_refresh and self.db.has_today_data(code, target_date):
    return True, None  # 已有数据，跳过网络请求
```

#### 12.3: 从数据源获取日线数据

```python
# pipeline.py:230
df, source_name = self.fetcher_manager.get_daily_data("600519", days=30)
```

**数据源故障切换链路** ([data_provider/base.py:1079](data_provider/base.py#L1079))：

```
A 股优先级（默认，未配置 TUSHARE_TOKEN）:
  1. EfinanceFetcher (P0)  ──┐
  2. AkshareFetcher (P1)     │
  3. PytdxFetcher (P2)       │  失败自动切换到下一个
  4. BaostockFetcher (P3)    │
  5. YfinanceFetcher (P4)    ──┘

配置了 TUSHARE_TOKEN 时:
  1. TushareFetcher (P0)     ──┐
  2. EfinanceFetcher (P0)      │
  3. AkshareFetcher (P1)       │  失败自动切换
  4. PytdxFetcher (P2)         │
  5. BaostockFetcher (P3)      │
  6. YfinanceFetcher (P4)      ──┘

美股/港股有专用路由（Finnhub/AlphaVantage/Longbridge/YFinance）
```

每个数据源内部还会自动计算技术指标：
- MA5, MA10, MA20（移动平均线）
- Volume Ratio（量比）

#### 12.4: 保存到数据库

```python
# pipeline.py:236
self.db.save_daily_data(df, "600519", source_name)
```

### Step 13: AI 分析（analyze_stock）

```python
# pipeline.py:1675
result = self.analyze_stock("600519", report_type, query_id=...)
```

这是整个流程最复杂的部分，内部执行以下步骤：

#### 13.1: 获取实时行情

```python
# pipeline.py:276
realtime_quote = self.fetcher_manager.get_realtime_quote("600519")
```

实时行情故障切换链路（按 `REALTIME_SOURCE_PRIORITY` 配置）：
```
默认: efinance → akshare_em(东财) → akshare_sina(新浪) → tencent(腾讯)
可选: tushare
```

如果首个数据源返回了基础数据但缺少部分字段（量比、换手率等），会继续从后续数据源**补充**缺失字段。

返回的 `UnifiedRealtimeQuote` 包含：
- `price`: 当前价格
- `change_pct`: 涨跌幅
- `volume_ratio`: 量比
- `turnover_rate`: 换手率
- `pe_ratio`: 市盈率
- `pb_ratio`: 市净率
- `total_mv`: 总市值
- `circ_mv`: 流通市值

#### 13.2: 获取筹码分布

```python
# pipeline.py:301
chip_data = self.fetcher_manager.get_chip_distribution("600519")
```

筹码分布带**熔断器**保护，每个数据源独立熔断。返回 `ChipDistribution`：
- `profit_ratio`: 获利比例
- `avg_cost`: 平均成本
- `concentration_90`: 90% 筹码集中度
- `concentration_70`: 70% 筹码集中度

#### 13.3: 基本面聚合

```python
# pipeline.py:334
fundamental_context = self.fetcher_manager.get_fundamental_context("600519")
```

基本面聚合包含 7 个模块块，采用 **fail-open** 语义（单个块失败不影响其他）：

| 模块 | 数据来源 | 内容 |
|------|----------|------|
| `valuation` | 实时行情 | PE/PB/总市值/流通市值 |
| `growth` | AkShare 基本面 | 营收增长/利润增长/ROE |
| `earnings` | AkShare 基本面 | 盈利数据/分红/股息率 |
| `institution` | AkShare 基本面 | 机构持仓/北上资金 |
| `capital_flow` | AkShare 资金流 | 主力/超大/大/中/小单资金流向 |
| `dragon_tiger` | AkShare 龙虎榜 | 是否上龙虎榜、近几日上榜次数 |
| `boards` | 板块排行 | 所属板块/概念涨跌榜 |

每个模块有独立的超时控制和重试机制，同时写入 `fundamental_snapshot` 供后续查询。

**ETF 降级**：ETF 代码的 `capital_flow`/`dragon_tiger`/`boards` 返回 `not_supported`。

**预算超时控制**：总预算默认 `FUNDAMENTAL_STAGE_TIMEOUT_SECONDS_DEFAULT`，超时后跳过剩余模块。

#### 13.4: 趋势分析（两条路径共用）

```python
# pipeline.py:367-379
trend_result = self.trend_analyzer.analyze(df, code)
```

从数据库获取最近 ~90 天的 K 线数据，计算：
- 趋势状态：多头排列 / 空头排列 / 短期向好 / 短期走弱 / 震荡整理
- MA 对齐情况
- 趋势强度
- 乖离率（BIAS）：相对 MA5/MA10
- 量价状态：放量/缩量
- 买入信号：强买入/买入/观望/卖出/强卖出
- 信号评分（signal_score）
- 支撑位/压力位

#### 13.5: 判断分析路径：传统 vs Agent

```python
# pipeline.py:315-325
use_agent = config.agent_mode  # 显式开启
if not use_agent:
    use_agent = bool(self.analysis_skills)  # 配置了特定策略
if not use_agent:
    use_agent = configured_skills != ['all']  # 定时任务配置了策略
```

- **默认**：走传统分析链路（`pipeline.py:397-535`）
- **Agent 模式**：走 Agent 分析链路（`pipeline.py:383-931`）

以下以 **默认传统分析链路** 为例继续追踪。

#### 13.6: 多维度情报搜索（新闻与舆情）

```python
# pipeline.py:404
intel_results = self.search_service.search_comprehensive_intel(
    stock_code="600519",
    stock_name="贵州茅台",
    max_searches=5
)
```

最多执行 5 次搜索，覆盖多个维度：
1. **最新消息**：股票名称 + 最新消息
2. **风险排查**：股票名称 + 风险/违规/诉讼
3. **业绩预期**：股票名称 + 业绩/财报/财报预期
4. **行业/板块**：所属行业最新动态
5. **政策影响**：相关政策变化

搜索结果格式化为 `news_context` 字符串，同时保存到数据库供历史查询。

如果是美股，还会叠加 **社交舆情**（Reddit/X/Polymarket）：
```python
# pipeline.py:438
social_context = self.social_sentiment_service.get_social_context("600519")
```

#### 13.7: 获取分析上下文（技术面数据）

```python
# pipeline.py:452
context = self.db.get_analysis_context(code)
```

从数据库获取最近交易日的 OHLCV 数据 + 技术指标。

#### 13.8: 增强上下文

```python
# pipeline.py:469
enhanced_context = self._enhance_context(
    context, 
    realtime_quote,   # 实时行情
    chip_data,        # 筹码分布
    trend_result,     # 趋势分析
    stock_name,       # 股票名称
    fundamental_context,  # 基本面
)
```

将上述所有数据合并为一个大的 `enhanced_context` 字典，包含：
- 历史 K 线数据（today/yesterday）
- 实时行情（价格、涨跌幅、量比、换手率、PE、PB 等）
- 筹码分布（获利比例、集中度）
- 趋势分析结果（趋势状态、买卖信号、信号评分）
- 基本面数据（估值、增长、盈利、机构、资金流、龙虎榜、板块）
- 股票名称和代码
- ETF/指数标记
- 报告语言设置

**盘中实时更新**：如果实时行情可用且是交易时间，会用实时价格覆盖 `today` 数据，确保 MA 计算使用最新价格（Issue #234）。

#### 13.9: 调用 LLM 分析

```python
# pipeline.py:492
result = self.analyzer.analyze(
    enhanced_context,
    news_context=news_context,
    progress_callback=self._emit_progress,
    stream_progress_callback=_on_llm_stream,
)
```

**analyzer.analyze() 内部流程** ([src/analyzer.py:2437](src/analyzer.py#L2437))：

1. **构建 System Prompt**：根据报告语言（zh/en）、股票类型（A股/港股/美股/ETF）生成不同的系统提示，包含交易理念和风险控制规则

2. **构建 User Prompt**：将 `enhanced_context` 格式化为结构化文本：
   ```
   请分析以下股票:
   股票: 600519 (贵州茅台)
   日期: 2026-05-19
   
   === 技术面数据 ===
   今日: 收盘价、开盘价、最高价、最低价、成交量、MA5/MA10/MA20
   昨日: 同上
   量价趋势: 放量/缩量
   MA状态: 多头排列/空头排列/震荡
   
   === 实时行情 ===
   当前价格: xxx
   涨跌幅: x.xx%
   量比: x.xx (温和放量/明显放量/正常/萎缩)
   换手率: x.xx%
   
   === 筹码分布 ===
   获利比例: xx%
   90%集中度: xx (集中/分散)
   
   === 基本面 ===
   估值: PE/PB/总市值
   增长: 营收增长率/利润增长率
   盈利: ROE/毛利率/分红
   机构: 机构持仓/北上资金
   资金流: 主力/超大单/大单/中单/小单
   龙虎榜: 是否上榜
   
   === 最新消息 ===
   (搜索结果格式化)
   
   请以 JSON 格式返回分析报告
   ```

3. **调用 LLM**：通过 LiteLLM Router 调用配置的模型（支持多模型 fallback）：
   - 流式接收响应
   - 支持完整性校验重试（最多 `report_integrity_retry` 次）
   - JSON 解析失败自动修复（json_repair 库）

4. **解析响应**：将 LLM 返回的 JSON 解析为 `AnalysisResult` 对象：
   - `sentiment_score`: 综合评分（0-100）
   - `trend_prediction`: 趋势预测（看多/看空/震荡）
   - `operation_advice`: 操作建议（买入/卖出/观望）
   - `confidence_level`: 置信度（高/中/低）
   - `analysis_summary`: 分析摘要
   - `risk_warning`: 风险警告
   - `dashboard`: 完整决策仪表盘（JSON 结构）
   - `chip_health`: 筹码健康度
   - `support_level`/`resistance_level`: 支撑/压力位

5. **后处理**：
   - 填充筹码结构（如果 LLM 未返回完整筹码分析）
   - 填充价格位置（结合趋势分析和实时行情）
   - 稳定决策（结合技术面 + 基本面）
   - 内容完整性校验（可选，失败时占位补全）

6. **保存分析历史**：
   ```python
   # pipeline.py:526
   self.db.save_analysis_history(
       result=result,
       query_id=query_id,
       report_type=report_type.value,
       news_content=news_context,
       context_snapshot=context_snapshot,
       save_snapshot=self.save_context_snapshot
   )
   ```

## 五、结果返回与通知推送

### Step 14: 收集结果

```python
# pipeline.py:1798-1803
for future in as_completed(future_to_code):
    result = future.result()
    if result and result.success:
        results.append(result)
```

### Step 15: 统计与保存本地报告

```python
# pipeline.py:1849-1854
logger.info(f"成功: {success_count}, 失败: {fail_count}, 耗时: {elapsed_time:.2f} 秒")
self._save_local_report(results, report_type)
```

报告保存到本地文件（目录由配置指定）。

### Step 16: 发送通知

```python
# pipeline.py:1867
self._send_notifications(results, report_type)
```

**通知推送链路** ([pipeline.py:1929](src/core/pipeline.py#L1929))：

1. 生成决策仪表盘报告（Markdown 格式）
2. 去重和噪声控制检查（避免重复推送相同内容）
3. Markdown 转图片（如果配置了 `wkhtmltoimage` 或 `markdown-to-file`）
4. 逐个渠道推送，单一渠道失败不影响其他：

| 渠道 | 特殊处理 |
|------|----------|
| 企业微信 | 发精简版（平台长度限制），可选转图片 |
| 飞书 | 发完整报告 |
| Telegram | 可选发图片 |
| Discord | 发完整报告 |
| Slack | 可选发图片 |
| 邮件 | 按股票分组发送到对应收件人，可嵌入图片 |
| PushPlus/Server酱/ntfy/Gotify/Pushover/AstrBot | 发完整报告 |
| 自定义 Webhook | 可选发图片 |
| 上下文推送 | 推送到当前会话上下文（Bot/Web 场景） |

### Step 17: 输出摘要

```python
# main.py:571-577
for r in sorted(results, key=lambda x: x.sentiment_score, reverse=True):
    emoji = r.get_emoji()
    logger.info(f"{emoji} {r.name}({r.code}): {r.operation_advice} | 评分 {r.sentiment_score} | {r.trend_prediction}")
```

### Step 18: 飞书云文档（如果配置）

```python
# main.py:582-621
feishu_doc.create_daily_doc(doc_title, full_content)
```

自动创建飞书每日复盘文档，并将文档链接推送到通知渠道。

### Step 19: 自动回测（如果配置）

```python
# main.py:624-641
service.run_backtest(force=False, ...)
```

对历史分析结果进行回测评估。

## 六、完整数据流全景图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          python main.py --stocks 600519                       │
└──────────────────────────┬───────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ 1. main.py 启动阶段                                                          │
│   解析参数 → 加载 .env → 初始化 Config 单例 → 初始化日志 → 选择运行模式        │
└──────────────────────────┬───────────────────────────────────────────────────┘
                           │ run_full_analysis(config, args, ["600519"])
                           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ 2. 交易日检查                                                                │
│   trading_calendar.get_market_for_stock("600519") → "cn"                     │
│   trading_calendar.get_open_markets_today() → {cn: True/False}               │
│   如果非交易日且无 --force-run → 跳过此股票                                   │
└──────────────────────────┬───────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ 3. StockAnalysisPipeline 初始化                                              │
│   ├── db = get_db()                          # SQLite 数据库                 │
│   ├── fetcher_manager = DataFetcherManager() # 多数据源管理器                  │
│   ├── trend_analyzer = StockTrendAnalyzer()  # 技术分析                       │
│   ├── analyzer = GeminiAnalyzer()            # LLM 分析                       │
│   ├── notifier = NotificationService()       # 通知推送                       │
│   ├── search_service = SearchService()       # 新闻搜索                       │
│   └── social_sentiment_service               # 社交舆情（仅美股）              │
└──────────────────────────┬───────────────────────────────────────────────────┘
                           │ pipeline.run(["600519"])
                           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ 4. 预取阶段                                                                  │
│   ├── prefetch_stock_names(["600519"])  # 预取股票名称到内存缓存              │
│   └── prefetch_realtime_quotes(["600519"])  # ≥5只时预取行情缓存              │
└──────────────────────────┬───────────────────────────────────────────────────┘
                           │ ThreadPoolExecutor(max_workers=3)
                           │ process_single_stock("600519")
                           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ 5. 获取并保存行情数据                                                         │
│   ├── get_stock_name("600519")                                               │
│   │   缓存 → 硬编码映射 → 索引文件 → 实时行情 → 各数据源                       │
│   │                                                                          │
│   ├── 断点续传检查: db.has_today_data("600519", target_date)                  │
│   │                                                                          │
│   └── get_daily_data("600519", days=30)                                      │
│       数据源故障切换: Efinance → AkShare → Pytdx → Baostock → YFinance       │
│       ↓                                                                      │
│       DataFrame[日期, 开盘, 最高, 最低, 收盘, 成交量, 成交额, 涨跌幅,          │
│                 MA5, MA10, MA20, 量比]                                        │
│       ↓                                                                      │
│       db.save_daily_data(df, "600519", source_name)  # 写入 SQLite            │
└──────────────────────────┬───────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ 6. AI 分析 (analyze_stock)                                                    │
│                                                                              │
│  6.1 实时行情                                                                 │
│      get_realtime_quote("600519")                                             │
│      故障切换: efinance → akshare_em → akshare_sina → tencent                │
│      首个成功数据源作为主数据，后续数据源补充缺失字段                           │
│      → UnifiedRealtimeQuote{price, change_pct, volume_ratio,                 │
│                             turnover_rate, pe_ratio, pb_ratio, ...}          │
│                                                                              │
│  6.2 筹码分布                                                                 │
│      get_chip_distribution("600519")                                          │
│      带熔断器保护，每个数据源独立熔断                                          │
│      → ChipDistribution{profit_ratio, avg_cost, concentration_90, ...}       │
│                                                                              │
│  6.3 基本面聚合                                                              │
│      get_fundamental_context("600519")                                        │
│      7 个模块块并行聚合，fail-open 语义，独立超时控制                          │
│      → {valuation, growth, earnings, institution,                            │
│         capital_flow, dragon_tiger, boards}                                  │
│                                                                              │
│  6.4 趋势分析                                                                 │
│      trend_analyzer.analyze(df)                                               │
│      从数据库获取 ~90 天 K 线 → 计算技术指标                                    │
│      → TrendAnalysisResult{trend_status, ma_alignment, trend_strength,       │
│         bias_ma5, bias_ma10, volume_status, buy_signal, signal_score, ...}   │
│                                                                              │
│  6.5 新闻搜索                                                                 │
│      search_service.search_comprehensive_intel("600519", "贵州茅台", 5)       │
│      最多 5 次搜索：最新消息 + 风险排查 + 业绩预期 + 行业动态 + 政策影响       │
│      → news_context (格式化 Markdown 字符串)                                  │
│                                                                              │
│  6.6 增强上下文                                                               │
│      _enhance_context() 合并所有数据                                           │
│      → enhanced_context {                                                     │
│          today: {close, open, high, low, volume, ma5, ma10, ma20},           │
│          yesterday: {...},                                                    │
│          realtime: {price, change_pct, volume_ratio, turnover_rate, ...},    │
│          chip: {profit_ratio, avg_cost, concentration_90, ...},              │
│          trend_analysis: {trend_status, ma_alignment, signal_score, ...},    │
│          fundamental_context: {valuation, growth, earnings, ...},            │
│          is_index_etf: False,                                                 │
│          report_language: "zh",                                               │
│          ...                                                                  │
│        }                                                                      │
│                                                                              │
│  6.7 LLM 调用                                                                │
│      analyzer.analyze(enhanced_context, news_context)                         │
│      ├── 构建 System Prompt（交易理念 + 风险控制规则）                         │
│      ├── 构建 User Prompt（结构化数据 + 新闻）                                │
│      ├── LiteLLM Router 调用配置的模型（支持多模型 fallback）                  │
│      ├── 流式接收响应 + 进度回调                                              │
│      ├── JSON 解析（json_repair 自动修复）                                    │
│      ├── 内容完整性校验（可选重试）                                            │
│      └── → AnalysisResult {                                                  │
│            sentiment_score: 72,                                               │
│            trend_prediction: "短期看多，中期震荡",                             │
│            operation_advice: "观望，等待回调至支撑位",                          │
│            confidence_level: "中",                                            │
│            analysis_summary: "...",                                           │
│            risk_warning: ["风险点1...", "风险点2..."],                        │
│            dashboard: { core_conclusion, battle_plan, intelligence, ... },   │
│            chip_health: "中性",                                                │
│            support_level: 1680.00,                                            │
│            resistance_level: 1800.00,                                         │
│            ...                                                                │
│          }                                                                    │
│                                                                              │
│  6.8 后处理                                                                   │
│      fill_chip_structure_if_needed()   # 筹码结构补全                          │
│      fill_price_position_if_needed()   # 价格位置补全                          │
│      stabilize_decision_with_structure()  # 结合技术面+基本面稳定决策          │
│                                                                              │
│  6.9 保存分析历史                                                              │
│      db.save_analysis_history(result, query_id, report_type, ...)             │
└──────────────────────────┬───────────────────────────────────────────────────┘
                           │ results = [AnalysisResult]
                           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ 7. 结果处理与推送                                                             │
│                                                                              │
│  7.1 统计: 成功 1, 失败 0, 耗时 xx.xx 秒                                     │
│                                                                              │
│  7.2 保存本地报告                                                             │
│      notifier.save_report_to_file(report)  # Markdown 文件                   │
│                                                                              │
│  7.3 噪声控制检查                                                             │
│      notifier.evaluate_noise_control()  # 去重 + 冷却                          │
│                                                                              │
│  7.4 Markdown 转图片（如果配置）                                              │
│      markdown_to_image(report) → image_bytes                                  │
│                                                                              │
│  7.5 逐个渠道推送（单一失败不影响其他）                                        │
│      ├── 企业微信: send_to_wechat(dashboard_content)  # 精简版                │
│      ├── 飞书:     send_to_feishu(report)                                     │
│      ├── Telegram: send_to_telegram(report) 或 _send_telegram_photo           │
│      ├── Discord:  send_to_discord(report)                                    │
│      ├── Slack:    send_to_slack(report) 或 _send_slack_image                 │
│      ├── 邮件:     send_to_email(report, receivers) 或 _send_email_with_image │
│      └── ... (PushPlus/Server酱/ntfy/Gotify/Pushover/AstrBot/Custom)          │
│                                                                              │
│  7.6 飞书云文档（如果配置）                                                    │
│      feishu_doc.create_daily_doc("2026-05-19 18:00 大盘复盘", content)        │
│                                                                              │
│  7.7 自动回测（如果配置）                                                      │
│      BacktestService.run_backtest()                                           │
└──────────────────────────┬───────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│ 8. 输出摘要                                                                   │
│   ===== 分析结果摘要 =====                                                    │
│   ⚪ 贵州茅台(600519): 观望 | 评分 72 | 短期看多，中期震荡                     │
│   任务执行完成                                                                 │
└──────────────────────────────────────────────────────────────────────────────┘
```

## 七、关键设计模式

### 7.1 策略模式 + 故障切换（数据源层）

```
                    ┌─────────────────────┐
                    │  DataFetcherManager  │
                    │  (策略管理器)        │
                    └──────┬──────────────┘
                           │ get_daily_data()
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │Efinance  │→│ AkShare   │→│ Pytdx    │  失败自动切换
        │ (P0)     │ │ (P1)     │ │ (P2)     │
        └──────────┘ └──────────┘ └──────────┘
```

### 7.2 熔断器模式（筹码分布）

每个数据源独立的熔断器，连续失败后进入熔断状态，一段时间后尝试恢复（HALF_OPEN）。

### 7.3 Fail-Open 语义（基本面聚合）

7 个基本面模块块，任一失败不影响整体流程，仅标记状态为 `partial` 或 `failed`。

### 7.4 线程池并发 + 故障隔离

```
ThreadPoolExecutor(max_workers=3)
  ├── Thread-1: 处理股票A ──→ 成功/失败（不影响其他）
  ├── Thread-2: 处理股票B ──→ 成功/失败（不影响其他）
  └── Thread-3: 处理股票C ──→ 成功/失败（不影响其他）
```

### 7.5 断点续传

```
db.has_today_data(code, target_date)
  ├── True  → 跳过网络请求，直接使用数据库已有数据
  └── False → 从数据源获取并保存
```

### 7.6 多模型 Fallback（LLM 层）

```
LiteLLM Router
  ├── Model-1（主模型，如 Gemini）──失败→
  ├── Model-2（备用模型，如 Claude）──失败→
  └── Model-3（兜底模型，如 OpenAI）
```

### 7.7 通知去重与噪声控制

- **去重键**：`report:aggregate:{report_type}:{stock_codes}`
- **冷却时间**：相同内容在冷却期内不重复推送
- **渠道隔离**：单一渠道失败不影响其他渠道

## 八、数据库表结构（SQLite）

分析过程中涉及的主要数据库表：

| 表名 | 用途 |
|------|------|
| `daily_data` | 日线行情数据（OHLCV + 技术指标） |
| `analysis_history` | 分析历史记录（结果 + 原始响应） |
| `news_intel` | 新闻情报搜索结果 |
| `fundamental_snapshot` | 基本面数据快照 |
| `backtest_results` | 回测结果 |
| `usage_records` | LLM 调用用量记录 |

## 九、关键时间节点

以 `600519` 为例，典型耗时分布（仅供参考）：

| 阶段 | 耗时 | 说明 |
|------|------|------|
| 配置加载 | < 100ms | 内存操作 |
| 名称预取 | ~200ms | 轻量查询 |
| 日线数据获取 | 1-5s | 取决于数据源响应速度 |
| 实时行情 | 1-3s | 网络请求 |
| 筹码分布 | 1-3s | 网络请求（可能熔断跳过） |
| 基本面聚合 | 5-15s | 多个模块并行，受预算超时控制 |
| 趋势分析 | < 100ms | 本地计算 |
| 新闻搜索 | 5-15s | 最多 5 次搜索（可被禁用） |
| LLM 分析 | 10-60s | 取决于模型和响应长度 |
| 结果解析 | < 100ms | 本地操作 |
| 通知推送 | 1-5s | 取决于渠道数量 |
| **总计** | **30-120s** | 因配置和网络而异 |

---

> 通过本文档，你可以清楚地理解从输入 `python main.py --stocks 600519` 到最终收到推送通知的完整链路。如需修改某个环节的行为，可以根据上述流程图定位到对应的代码文件和方法。
