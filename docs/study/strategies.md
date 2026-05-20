# 交易策略（Strategies / Skills）系统详解

## 一、概念总览

策略（Strategies，也叫 Skills）是注入到 Agent 分析链路中的 **自然语言交易模式定义**。每个策略描述一种特定的炒股方法论（如龙头战法、缩量回踩、波浪理论等），以 YAML 文件编写，由 LLM 在分析股票时遵照执行。

```
策略不是 Python 代码，而是 Prompt 片段
→ 通过 YAML 自然语言定义
→ 在 Agent 分析时注入 System Prompt
→ LLM 按照策略规则分析并给出交易建议
```

## 二、目录结构与文件清单

所有内置策略存放在 `strategies/` 目录：

| 文件名 | 策略名 | 类别 | 适用市场状态 | 优先级 |
|--------|--------|------|-------------|--------|
| `bull_trend.yaml` | 默认多头趋势 | 趋势类 | trending_up | 10（最高） |
| `shrink_pullback.yaml` | 缩量回踩 | 趋势类 | trending_down, sideways | 40 |
| `dragon_head.yaml` | 龙头策略 | 趋势类 | sector_hot | 90 |
| `ma_golden_cross.yaml` | 均线金叉 | 趋势类 | trending_down, sideways | — |
| `volume_breakout.yaml` | 放量突破 | 趋势类 | trending_up, volatile | — |
| `bottom_volume.yaml` | 底部放量 | 趋势类 | trending_down | — |
| `box_oscillation.yaml` | 箱体震荡 | 形态类 | sideways | — |
| `one_yang_three_yin.yaml` | 一阳穿三阴 | 反转类 | volatile | — |
| `chan_theory.yaml` | 缠论 | 形态类 | sideways, trending_up | — |
| `wave_theory.yaml` | 波浪理论 | 形态类 | trending_up | — |
| `emotion_cycle.yaml` | 情绪周期 | 反转类 | volatile, sector_hot | — |
| `event_driven.yaml` | 事件驱动 | 框架类 | — | — |
| `growth_quality.yaml` | 成长质量 | 趋势类 | trending_up | — |
| `expectation_repricing.yaml` | 预期重估 | 反转类 | sideways | — |
| `hot_theme.yaml` | 热点题材 | 趋势类 | sector_hot, volatile | — |

## 三、YAML 格式规范

每个策略文件包含以下字段：

```yaml
name: dragon_head                  # 唯一标识符（英文名）
display_name: 龙头策略              # 人类可读名称
description: 板块轮动中识别龙头股    # 简述，何时使用该策略
category: trend                    # 类别：trend/pattern/reversal/framework
core_rules: [2, 7]                 # 关联核心理念编号（1-7）
required_tools:                    # 依赖的 Agent 工具
  - get_realtime_quote
  - get_sector_rankings
  - search_stock_news
aliases: [龙头, 龙头战法]           # 中文别名，用于自然语言匹配
default_active: true               # 是否默认激活（仅 bull_trend）
default_router: true               # 是否参与路由器的自动选择
default_priority: 10               # 优先级数值，越小越靠前
market_regimes: [sector_hot]       # 适用的市场状态

instructions: |
  **策略详细说明...**
  1. 第一步分析...
  2. 第二步分析...
  评分调整：
  - 满足条件A: sentiment_score +10
  - 满足条件B: sentiment_score -5
```

## 四、四种策略类别

| 类别 | 说明 | 代表策略 |
|------|------|---------|
| **趋势类（trend）** | 跟随趋势方向交易 | 多头趋势、缩量回踩、龙头、放量突破、底部放量、均线金叉 |
| **形态类（pattern）** | 基于价格形态判断 | 箱体震荡、缠论、波浪理论 |
| **反转类（reversal）** | 捕捉拐点信号 | 一阳穿三阴、情绪周期、预期重估 |
| **框架类（framework）** | 通用分析框架 | 事件驱动、成长质量、热点题材 |

## 五、核心理念（Core Rules）

所有策略共享 7 条核心理念基线，通过 `core_rules` 字段关联：

| 编号 | 理念 | 内容 |
|------|------|------|
| 1 | **严进策略** | 不追高，乖离率 > 5% 不买入 |
| 2 | **趋势交易** | 只做 MA5 > MA10 > MA20 多头排列 |
| 3 | **效率优先** | 关注筹码集中度 |
| 4 | **买点偏好** | 缩量回踩 MA5/MA10 支撑 |
| 5 | **风险排查** | 减持、预亏、处罚、解禁 |
| 6 | **估值关注** | PE/PB 偏高时提示风险 |
| 7 | **强势放宽** | 强势趋势股可适当放宽乖离率 |

## 六、在 `main.py` 中的使用链路

```
python main.py
  │
  └── run_full_analysis()
        │
        └── StockAnalysisPipeline(config=config)
              │  构造时读取：
              │    self.analysis_skills = 请求传入的技能列表（可为 None）
              │    config.agent_skills   = .env 中的 AGENT_SKILLS 配置
              │
              └── analyze_stock(code, report_type, query_id)
                    │
                    │  # 判断是否进入 Agent 模式
                    │  use_agent = config.agent_mode                    # .env 显式开启
                    │           or self.analysis_skills                 # 请求传入技能
                    │           or config.agent_skills != ['all']       # 配置了非全部技能
                    │
                    └── 如果 use_agent = True:
                          _analyze_with_agent()
                            │
                            └── build_agent_executor(config, requested_skills)
                                  │
                                  ├── 1. ToolRegistry（缓存单例）
                                  │      注册所有可用工具
                                  │
                                  ├── 2. SkillManager（从磁盘加载）
                                  │      ├── 加载 strategies/*.yaml 内置策略
                                  │      └── 加载 AGENT_SKILL_DIR 自定义策略
                                  │
                                  ├── 3. SkillRouter 路由选择
                                  │      ├── 用户显式指定 → 直接用
                                  │      ├── manual 模式 → 读 .env AGENT_SKILLS
                                  │      ├── auto 模式 → 检测市场 regime 自动匹配
                                  │      └── 回退 → 默认策略（bull_trend）
                                  │
                                  └── 4. AgentExecutor 执行
                                        ├── 激活策略 → 注入 System Prompt
                                        └── LLM 分析股票
```

### 关键决策点

```python
# pipeline.py:315-325
use_agent = getattr(self.config, 'agent_mode', False)
if not use_agent:
    if self.analysis_skills:
        use_agent = True
if not use_agent:
    configured_skills = getattr(self.config, 'agent_skills', [])
    if configured_skills and configured_skills != ['all']:
        use_agent = True
```

**三条路都能触发策略**，按优先级从高到低：

| 优先级 | 触发条件 | 来源 | 示例 |
|--------|---------|------|------|
| 1 | `AGENT_MODE=true` | .env 显式开启 | 直接启用 Agent 模式 |
| 2 | API 传入 `skills` 参数 | 请求参数 | `{"stock_code": "600519", "skills": ["dragon_head"]}` |
| 3 | `AGENT_SKILLS` 配置了具体值 | .env | `AGENT_SKILLS=bull_trend,shrink_pullback`（自动启用 Agent 模式） |

> **注意**：`AGENT_SKILLS=['all']` 是特殊值，**不会** 自动开启 Agent 模式，被视为"不选择特定策略"。

**结论**：策略只在 **Agent 模式** 下生效。传统 LLM 分析不经过策略路由。但 Agent 模式不一定需要 `AGENT_MODE=true`，配置了具体 `AGENT_SKILLS` 也会自动切换。

**结论**：`python main.py`、`--serve`、`--schedule`、`--stocks 600519` 等所有运行模式都走同一条 `analyze_stock()` 代码路径，策略生效条件与运行模式无关。


## 七、配置项

### `.env` 中的策略相关配置

```env
# 必须开启 Agent 模式
AGENT_MODE=true

# 指定激活的策略（逗号分隔）
AGENT_SKILLS=bull_trend,shrink_pullback

# 自定义策略目录（可选）
AGENT_SKILL_DIR=/path/to/my/strategies

# 路由模式
AGENT_SKILL_ROUTING=auto       # auto: 自动检测市场状态匹配
                               # manual: 严格按 AGENT_SKILLS 配置
```

### API 请求中指定策略

```json
POST /api/v1/analysis/analyze
{
  "stock_code": "600519",
  "async_mode": true,
  "skills": ["dragon_head", "volume_breakout"]
}
```

### Bot 命令指定策略

```
/ask 600519 龙头策略          # 指定策略分析
/ask 600519 缩量回踩           # 指定策略分析
/strategies                   # 查看所有策略
/strategies active            # 查看已激活策略
```

## 八、策略路由（SkillRouter）

路由逻辑决定**哪个策略被激活**：

```
SkillRouter.select_skills(ctx)
  │
  ├─ 优先级1: 用户显式请求
  │     分析请求中指定了 skills 参数 → 直接使用
  │
  ├─ 优先级2: 手动模式 (routing_mode = "manual")
  │     读取 .env 中 AGENT_SKILLS 配置 → 过滤可用列表 → 使用
  │
  ├─ 优先级3: 自动模式 (routing_mode = "auto")
  │     检测市场 regime → 匹配策略 → 使用
  │     │
  │     ├── trending_up   → bull_trend, volume_breakout, ...
  │     ├── trending_down → shrink_pullback, bottom_volume, ...
  │     ├── sideways      → box_oscillation, chan_theory, ...
  │     ├── volatile      → volume_breakout, emotion_cycle, ...
  │     └── sector_hot    → dragon_head, hot_theme, ...
  │
  └─ 优先级4: 默认回退
        使用 default_active: true 的策略（bull_trend）
```

## 九、市场状态（Regime）自动检测

系统从技术分析器的输出中自动检测市场状态：

```python
# router.py:75-99
def _detect_regime(ctx):
    ma_alignment = raw.get("ma_alignment")   # bullish / bearish / neutral
    trend_score = raw.get("trend_score", 50)  # 0-100
    volume_status = raw.get("volume_status")   # heavy / normal / light

    bullish + trend_score >= 70  → "trending_up"
    bearish + trend_score <= 30  → "trending_down"
    neutral / 35-65 分           → "sideways"
    heavy 放量 + 30-70 分        → "volatile"
    ctx.meta.sector_hot          → "sector_hot"
```

## 十、策略激活规则

```python
# SkillManager.activate()
manager.activate(["bull_trend", "shrink_pullback"])
# → 仅激活这两个策略，其他策略 disabled

manager.activate(["all"])
# → 激活所有策略
```

激活后的策略其 `instructions` 会被格式化为 System Prompt 片段：

```
#### 趋势类技能

### 技能 1: 默认多头趋势（关联核心理念：第1、2、3条）

**适用场景**: 常规个股分析的默认策略...

（Default Bull Trend Strategy）
适用场景：
- 优先寻找"趋势向上 + 风险可控 + 不追高"的机会...

### 技能 2: 缩量回踩（关联核心理念：第1、2、4条）

**适用场景**: 检测缩量回踩均线支撑信号...

（Shrink Volume Pullback Strategy）
入场判定标准：
- 股票必须处于上升趋势...
```

## 十一、自定义策略

### 方式一：创建 YAML 文件

在自定义目录中创建 `.yaml` 文件：

```yaml
# my_strategy.yaml
name: my_custom_strategy
display_name: 我的自定义策略
description: 描述何时使用这个策略
category: trend
core_rules: [1, 2]
instructions: |
  **我的策略详细说明**

  1. 第一步...
  2. 第二步...

  评分调整：
  - 满足条件: sentiment_score +5
```

### 方式二：SKILL.md 格式（Bundle）

支持类似 Claude Code 的 SKILL.md 格式：

```markdown
---
name: my_strategy
display_name: 我的策略
description: 策略描述
category: trend
default-active: false
---

## 我的策略详细说明

### 入场条件
1. ...

### 评分调整
...
```

### 启用自定义策略

```env
AGENT_SKILL_DIR=D:\1hello\daima\agent\daily_stock_analysis\strategies
AGENT_SKILLS=my_custom_strategy
```

自定义策略会**覆盖**同名内置策略。

## 十二、完整数据流

```
.env 配置
  │  AGENT_MODE=true
  │  AGENT_SKILLS=bull_trend,shrink_pullback
  │  AGENT_SKILL_DIR=...
  │
  ▼
SkillManager
  │  load_builtin_skills()    → strategies/*.yaml
  │  load_custom_skills(DIR)  → 自定义目录
  │
  ▼
SkillRouter
  │  select_skills(ctx)       → 决定激活哪些策略
  │
  ▼
AgentExecutor
  │  get_skill_instructions() → 生成 Prompt 片段
  │
  ▼
LLM (Gemini/Claude)
  │  System Prompt 中包含策略指令
  │  分析股票并遵循策略规则
  │
  ▼
AnalysisResult
     sentiment_score, operation_advice, trend_prediction...
```

## 十三、常见问题

### Q1: 策略和 Agent 的关系？
策略是 Agent 的**分析视角**。Agent 模式启用后，策略以自然语言指令注入 LLM 的 System Prompt，引导 LLM 按特定方法论分析。

### Q2: 不启用 Agent 模式时策略生效吗？
不生效。传统模式（`agent_mode=false`）使用固定 Prompt，不经过策略路由。

### Q3: 可以同时激活多个策略吗？
可以。多个策略的指令会按类别拼接后一起注入 Prompt。

### Q4: 如何查看所有可用策略？
Bot 中发送 `/strategies` 查看完整列表，`/strategies active` 查看已激活的策略。

### Q5: 策略修改后需要重启吗？
需要。策略在 Pipeline 初始化时从磁盘加载，修改后需重新运行 `main.py` 才会生效。
