# 数据源路由规则

AI投委会 Phase 1 数据采集层的完整调用清单。Agent 执行时按此文档调用各数据源。

---

## 1. 角色×数据源路由表

| 角色 | Tier 1 调用 | Tier 2 调用（opencli优先） | 说明 |
|------|-----------|--------------------------|------|
| 研究员 | iwencai(公告+基本面+行业) | opencli(weixin/google)→WebSearch兜底 | 全Tier覆盖 |
| 基本面分析师 | iwencai(财务+估值+管理层) + MX_FinData | opencli(eastmoney quote) | Tier 1为主 |
| 技术分析师 | iwencai(行情+技术指标) + westock-data(K线+筹码) | — | Tier 1为主 |
| 舆情分析师 | — | opencli(xueqiu/weibo/weixin/zhihu)→WebSearch | Tier 2为主 |
| 风险官 | iwencai(公告/事件/增减持) | opencli(weixin/google)→WebSearch兜底 | Tier 1+2混合 |

---

## 2. iwencai-unified 完整调用清单

### 研究员（3条 iwencai + 2条 opencli）

```
iwencai-unified: "{name} 最近3个月重大公告"           → hithink-event-query
iwencai-unified: "{name} 主营业务 行业分类 概念题材"    → hithink-basicinfo-query
iwencai-unified: "{name} 所处行业 景气度 产能"         → hithink-industry-query
```

**输出字段映射：**
- 公告 → data_bundle.events[]
- 主营业务 → data_bundle.meta（行业信息）
- 行业动态 → data_bundle.industry_data[]

### 基本面分析师（5条 iwencai + 1条 MX_FinData + 1条 opencli）

```
iwencai-unified: "{name} 营收 净利润 毛利率 净利率 最近4期"  → hithink-finance-query
iwencai-unified: "{name} 资产负债率 流动比率 现金流"         → hithink-finance-query
iwencai-unified: "{name} PE PB PS 市值 总股本"              → hithink-basicinfo-query
iwencai-unified: "{name} 研报评级 目标价 盈利预测"           → hithink-insresearch-query
iwencai-unified: "{name} 前十大股东 机构持仓"               → hithink-management-query
MX_FinData: "查询{name}同行业可比公司PE PB估值对比"
opencli eastmoney quote "{code}" -f json
```

**输出字段映射：**
- 营收/净利润/毛利率 → data_bundle.financial.revenue/net_profit/gross_margin
- 资产负债/现金流 → data_bundle.financial.cash_flow/balance_sheet
- PE/PB/市值 → data_bundle.valuation
- 研报 → data_bundle.research_reports[]
- 股东 → data_bundle.shareholders[]

### 技术分析师（2条 iwencai + 1条 westock-data）

```
iwencai-unified: "{name} 行情 成交量 换手率 量比 振幅"      → hithink-market-query
iwencai-unified: "{name} MACD KDJ RSI 布林带"              → hithink-market-query
westock-data: "{name} K线 技术指标 筹码分析 资金流向"
```

**输出字段映射：**
- 价格/成交量 → data_bundle.technical.price_current/volume
- 均线系统 → data_bundle.technical.ma_system
- MACD/RSI → data_bundle.technical.macd/rsi
- 支撑/阻力 → data_bundle.technical.support_levels/resistance_levels

### 风险官（4条 iwencai + 2条 opencli）

```
iwencai-unified: "{name} 监管函 问询函 处罚 警示函"        → hithink-event-query
iwencai-unified: "{name} 股东减持计划 限售解禁"            → hithink-management-query
iwencai-unified: "{name} 股权质押 质押比例"               → hithink-event-query
iwencai-unified: "{name} 业绩预告 业绩修正"               → hithink-event-query
```

**输出字段映射：**
- 监管/问询 → data_bundle.risk_events[type=监管]
- 减持/解禁 → data_bundle.risk_events[type=减持]
- 质押 → data_bundle.risk_events[type=质押]
- 业绩修正 → data_bundle.risk_events[type=业绩]

---

## 3. opencli 命令清单 + 降级策略

### 舆情分析师（4条 opencli + 2条专业Skill）

```
opencli xueqiu search "{name}" -f json --limit 10        ← 雪球讨论（需COOKIE）
opencli weixin search "{name}" -f json --limit 10        ← 公众号文章（PUBLIC）
opencli weibo search "{name}" -f json --limit 10         ← 微博舆情（需COOKIE）
opencli zhihu search "{name} 投资" -f json --limit 10    ← 知乎深度分析（需COOKIE）
MX_FinSearch: "搜索{name}相关研报"
ifind-repilot-news-search: "{name} 市场舆情 最新动态"
```

**降级命令：**
```
WebSearch("{name} 雪球 讨论")
WebSearch("{name} 最新新闻 评论")
WebSearch("{name} 散户 股吧")
```

**输出字段映射：**
- 雪球/微博/知乎 → data_bundle.news_sentiment[]
- 研报 → data_bundle.research_reports[]

### 研究员 opencli

```
opencli weixin search "{name} 产能扩张 客户拓展" -f json --limit 5   ← 公众号深度文章
opencli google search "{name} 最新动态" -f json --limit 5            ← 综合新闻
```

**降级：** `WebSearch("{name} 最新动态 产能扩张 客户拓展")`

### 基本面分析师 opencli

```
opencli eastmoney quote "{code}" -f json   ← 实时行情验证
```

### 风险官 opencli

```
opencli weixin search "{name} 诉讼 风险" -f json --limit 5          ← 公众号风险报道
opencli google search "{name} 诉讼 仲裁 监管" -f json --limit 5     ← 英文风险新闻
```

**降级：** `WebSearch("{name} 诉讼 仲裁 风险")`

---

## 4. 降级策略决策树

```
Phase 0 检测 opencli doctor --no-live
│
├─ 正常 (exit code 0)
│   └─ search_strategy = "opencli_first"
│       ├─ 尝试优先级1: opencli专用适配器（xueqiu/weixin/weibo/zhihu/eastmoney）
│       │   ├─ 成功 → 使用结果
│       │   └─ 失败 → 尝试优先级2
│       ├─ 尝试优先级2: opencli通用搜索（google/brave/bing）
│       │   ├─ 成功 → 使用结果
│       │   └─ 失败 → 降级到优先级3
│       └─ 优先级3: Agent WebSearch（兜底）
│
└─ 失败 (exit code != 0)
    └─ search_strategy = "websearch_only"
        └─ 直接使用 Agent WebSearch
```

---

## 5. 数据来源标注规范

每个数据点必须标注来源，用于 data_bundle.json 的 source 字段：

| 来源 | source值 |
|------|----------|
| iwencai-unified | "iwencai" |
| MX_FinData | "MX_FinData" |
| westock-data | "westock" |
| opencli eastmoney | "opencli/eastmoney" |
| opencli xueqiu | "opencli/xueqiu" |
| opencli weixin | "opencli/weixin" |
| opencli weibo | "opencli/weibo" |
| opencli zhihu | "opencli/zhihu" |
| opencli google | "opencli/google" |
| WebSearch | "WebSearch" |
| MX_FinSearch | "MX_FinSearch" |
| 知识库 | "kb" |
