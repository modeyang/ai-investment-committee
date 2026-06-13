---
name: ai-investment-committee
description: |
  AI投资委员会：多角色并行股票投研分析。
  输入股票代码/名称，自动完成 预检→数据采集→5角色分析→投委会合议→输出报告+知识库沉淀。
  触发词："投委会分析"、"AI投委会"、"投资委员会"、"多角色分析"、"committee analysis"
allowed-tools: Bash(opencli:*), Bash(python3:*), Read, Write, SearchReplace, Glob, Grep, WebSearch, WebFetch, Agent(GeneralPurpose)
---

# AI 投资委员会 Skill

多角色并行股票投研分析系统。5 阶段流水线，JSON 中间合约串联，知识库双向联动。

## 版本

`1.0.0`

## 输入规范

```
用户提供: 股票代码或名称（如 "688300" / "联瑞新材"）
可选参数: --outputs html,minutes,risk,summary,excel
```

### 参数解析

- 默认输出：仅 HTML 主报告
- 可选值：`html`(必选), `minutes`(纪要), `risk`(风险清单), `summary`(一页纸), `excel`(证据表)
- 从用户消息中提取 `--outputs` 参数，逗号分割，控制 Phase 4 生成哪些文件
- 示例：`/ai-investment-committee 688300 --outputs html,minutes,risk`

## 输出规范

| 输出 | 必选 | 路径 | 说明 |
|------|------|------|------|
| HTML主报告 | 是 | `dashboard/{name}_AI投委会_{date}.html` | 可视化总览 |
| 投委会纪要 | 按需 | `dashboard/{name}_纪要_{date}.md` | 决策过程记录 |
| 风险清单 | 按需 | `dashboard/{name}_风险清单_{date}.md` | 结构化风险矩阵 |
| 一页纸总结 | 按需 | `dashboard/{name}_总结_{date}.md` | 决策者速览 |
| Excel证据表 | 按需 | `dashboard/{name}_证据表_{date}.xlsx` | 数据来源追溯 |
| 信号卡片 | 是 | `03_memory/signals/{name}_committee_{date}.md` | 知识库沉淀 |

**重要：HTML是唯一全量报告，其他文件各自差异化，不重复HTML内容。**

---

## 执行流程

### Phase 0: 预检 & 上下文加载

**目标：** 解析标的、加载知识库上下文、检测工具可用性。

#### Step 0.1: 解析股票代码

1. 用户输入可能是代码（如 "688300"）或名称（如 "联瑞新材"）
2. 如需映射，调用 `iwencai-unified: "{input} 股票代码 公司名称"` 获取标准代码和名称
3. 确定交易所：6开头=SSE(上交所), 0/3开头=SZSE(深交所), 4/8开头=BSE(北交所)

#### Step 0.2: 加载知识库上下文

1. **公司卡片**：`Glob("02_knowledge/companies/*{name}*")` → 如有则 `Read` 读取
2. **历史信号**：`Grep("{name}", "03_memory/signals/")` → 记录文件路径和摘要
3. **相关主题**：`Glob("02_knowledge/themes/*")` → 根据公司所属行业筛选匹配

#### Step 0.3: 检测工具可用性

```bash
# 检测 opencli
opencli doctor --no-live 2>&1
# 正常 → tools_available.opencli = true
# 失败 → tools_available.opencli = false, search_strategy = "websearch_only"
```

```
# 检测 API Key（从 .env 文件读取）
Read(".env") → 检查 IWENCAI_API_KEY 和 EM_API_KEY 是否存在
```

**工具可用性记录到 context.json.tools_available：**
- `opencli`: true/false
- `opencli_cookie_adapters`: true/false（Chrome扩展是否就绪）
- `iwencai_api`: true/false
- `mx_api`: true/false
- `search_strategy`: "opencli_first" | "websearch_only"

#### Step 0.4: 输出 context.json

按 `references/context_schema.json` 格式生成 context.json（内存中，不写文件）。

---

### Phase 1: 数据采集层

**目标：** 收集结构化金融数据 + 非结构化资讯，输出 data_bundle.json。

**数据源架构（三层）：**

#### Tier 1: 结构化金融数据（专业API，优先级最高）

| 工具 | 用途 | 调用方式 |
|------|------|---------|
| iwencai-unified | 主力：财务/行情/公告/估值/研报/股东/行业 | `Skill("iwencai-unified", "查询语句")` |
| MX_FinData | 补充：财务指标、估值对比 | `Skill("MX_FinData", "查询语句")` |
| westock-data | 补充：K线、筹码、资金流向 | `Skill("westock-data", "查询语句")` |

#### Tier 2: 非结构化资讯（opencli优先 + WebSearch兜底）

**降级策略（根据 context.tools_available.search_strategy）：**

| 优先级 | 工具 | 策略 | 说明 |
|--------|------|------|------|
| 1 | opencli 专用适配器 | PUBLIC/COOKIE | eastmoney/xueqiu/weixin/weibo/zhihu |
| 2 | opencli 通用搜索 | PUBLIC（无需浏览器） | google/brave/bing |
| 3 | Agent内置 WebSearch | 始终可用 | 兜底方案 |

**opencli 调用格式：**
```bash
opencli weixin search "{关键词}" -f json --limit 5
opencli google search "{关键词}" -f json --limit 5
opencli xueqiu search "{name}" -f json --limit 5
opencli weibo search "{关键词}" -f json --limit 5
opencli zhihu search "{关键词}" -f json --limit 5
```

**降级规则：** 如果 opencli 命令返回非零退出码或超时，自动降级到 WebSearch。

#### 各角色数据源路由

详细调用清单见 `references/datasource_routing.md`，包含每个角色的完整 iwencai/opencli/WebSearch 命令清单。

**概要路由：**
- 研究员：Tier1(公告+基本面+行业) + Tier2(weixin/google→WebSearch)
- 基本面分析师：Tier1(财务+估值+管理层+MX_FinData) + Tier2(eastmoney quote)
- 技术分析师：Tier1(行情+技术指标+westock) — 不需要Tier2
- 舆情分析师：Tier2(xueqiu/weibo/weixin/zhihu→WebSearch) + MX_FinSearch
- 风险官：Tier1(公告/事件/增减持) + Tier2(weixin/google→WebSearch)

#### 数据采集并行策略

使用 **5 个并行 GeneralPurpose 子Agent** 加速数据采集：
- Agent 1（研究员数据）：iwencai公告+基本面+行业 + opencli搜索
- Agent 2（基本面数据）：iwencai财务+估值+股东 + MX_FinData
- Agent 3（技术面数据）：iwencai行情+指标 + westock
- Agent 4（舆情数据）：opencli雪球/微博/公众号/知乎 + MX_FinSearch
- Agent 5（风险数据）：iwencai公告/事件 + opencli搜索

每个 Agent 返回结构化 JSON，主流程合并为 data_bundle.json。

#### 输出 data_bundle.json

按 `references/data_schema.json` 格式生成，包含：
- meta（元信息+数据来源）
- financial（财务数据，每条标注source）
- valuation（估值数据）
- technical（技术指标）
- events（公告/事件）
- news_sentiment（新闻/舆情，标注opencli来源或WebSearch来源）
- risk_events（风险事件）
- shareholders（股东信息）
- research_reports（研报）
- industry_data（行业数据）
- kb_context（来自Phase 0的知识库上下文）

---

### Phase 2: 分析引擎层

**目标：** 5个分析角色并行消费 data_bundle.json，各自输出结构化分析。

启动 **5 个并行 GeneralPurpose 子Agent**，每个Agent接收完整的 data_bundle.json 和 context.json。

#### 角色1: 研究员（Researcher）

**职责：** 客观信息聚合，不做主观判断。
**输出字段：** company_profile, key_events, industry_dynamics, recent_developments, data_sources

#### 角色2: 基本面分析师（Fundamental Analyst）

**职责：** 财务分析 + 估值判断 + 竞争格局评估。
**输出字段：** revenue_structure, profitability_trend, cash_flow_assessment, valuation_assessment, competitive_moat, growth_drivers
**评分：** scores.quality(1-10), scores.growth(1-10), scores.valuation(1-10, 10=最便宜)

#### 角色3: 技术分析师（Technical Analyst）

**职责：** 走势判断 + 指标分析 + 支撑/阻力位。
**输出字段：** trend_short/mid/long{score, label}, key_support[], key_resistance[], indicators_summary
**评分：** overall_score(1-10)

#### 角色4: 舆情分析师（Sentiment Analyst）

**职责：** 情绪分布 + 争议识别 + 潜在误读检测。
**输出字段：** news_ratio{positive, neutral, negative}, retail_mood, controversies, potential_misreads, heat_level
**评分：** overall_score(1-10, 10=极度乐观)

#### 角色5: 风险官（Risk Officer）

**职责：** 反面证据收集 + 风险评级矩阵 + 监控日历。
**输出字段：** risk_matrix[{category, rating, probability, impact, detail}], top_risks, low_risks, monitoring_calendar
**评分：** overall_score(1-10, 10=最安全)

#### 输出 analyses.json

5个Agent的输出合并为 analyses.json，按 `references/analysis_schema.json` 格式。

---

### Phase 3: 投委会合议

**目标：** 投资经理角色读取 analyses.json，综合评分、识别矛盾、形成结论。

#### Step 3.1: 加权评分

| 维度 | 权重 | 数据来源 |
|------|------|---------|
| 产业逻辑与赛道 | 20% | fundamental.growth_drivers |
| 公司质地与竞争力 | 20% | fundamental.competitive_moat |
| 成长性 | 20% | fundamental.scores.growth |
| 财务健康度 | 15% | fundamental.cash_flow_assessment |
| 估值合理性 | 15% | fundamental.scores.valuation |
| 风险收益比 | 10% | risk.overall_score + technical.overall_score |

**评分等级：** 1-3分=看空, 4-5分=中性, 6-7分=谨慎看多, 8-10分=强烈看多

#### Step 3.2: 矛盾识别

自动检测角色间结论冲突：

```
规则1: fundamental.quality > 7 AND valuation < 3 → "好公司但不是好价格"
规则2: technical.overall_score > 6 AND risk.risk_matrix含评级'高' → "趋势向上但风险累积"
规则3: sentiment.heat_level == '高' AND fundamental.growth < 6 → "市场热度与基本面不匹配"
规则4: researcher关键正面 AND risk.top_risks > 3条高危 → "信息面正面但结构性风险突出"
```

#### Step 3.3: 形成结论

- `one_liner`：一句话定性（如"好公司、好赛道，但不是好价格"）
- `core_contradiction`：核心矛盾描述
- `bull_case`：看多理由 Top3
- `bear_case`：看空理由 Top3
- `key_dates`：未来关键催化/风险事件时间线
- `verdict`：最终定性判断（强烈看多/谨慎看多/中性/谨慎看空/强烈看空）
- `committee_votes`：各角色投票意见

#### 输出 committee.json

按 `references/committee_schema.json` 格式。

---

### Phase 4: 输出 & 沉淀

**目标：** 生成最终交付物 + 知识库回写。

#### Step 4.1: HTML主报告（始终生成）

读取 `assets/report_template.html` 作为模板参考，注入 committee.json + analyses.json + data_bundle.json 数据，生成完整 HTML 报告。

**报告结构：**
- 概览卡片（股价/市值/PE/营收/净利）
- Section 1: 研究员报告
- Section 2: 基本面分析
- Section 3: 技术面分析
- Section 4: 舆情分析
- Section 5: 风险矩阵
- Section 6: 投委会合议
- Section 7: 投资经理总结

**输出路径：** `dashboard/{name}_AI投委会_{date}.html`

**注意：** Agent 直接用 Write 工具将填充好的 HTML 写入文件。模板中的占位符由 Agent 在生成时直接替换为实际值。

#### Step 4.2: 按需生成其他格式

根据 `--outputs` 参数决定生成哪些文件：

**minutes（纪要）：** 参考 `assets/minutes_template.md`。
- 独有内容：对话体讨论过程、争议分支、后续跟踪清单
- 输出：`dashboard/{name}_纪要_{date}.md`

**risk（风险清单）：** 参考 `assets/risk_template.md`。
- 独有内容：每条风险的监控指标、触发条件、应对预案
- 输出：`dashboard/{name}_风险清单_{date}.md`

**summary（一页纸）：** 参考 `assets/summary_template.md`。
- 独有内容：严格A4一页，面向决策者
- 输出：`dashboard/{name}_总结_{date}.md`

**excel（证据表）：** 调用 `assets/evidence_gen.py`。
```bash
python3 ~/.agents/skills/ai-investment-committee/assets/evidence_gen.py \
  /path/to/data_bundle.json \
  dashboard/{name}_证据表_{date}.xlsx
```
- 独有内容：原始数据分Sheet、数据来源列、时间戳
- 输出：`dashboard/{name}_证据表_{date}.xlsx`

#### Step 4.3: 知识库回写（信号卡片）

分析完成后，**必须**写入信号卡片到知识库。

**路径：** `03_memory/signals/{stock_name}_committee_{date}.md`

**格式（对齐 `00_meta/schemas/signal_schema.yaml` 的 required 字段）：**

```yaml
---
title: "{stock_name} AI投委会分析"
type: committee_analysis
status: active
created: {date}
verification_status: verified
grade: {verdict对应: A+(强烈看多) | A(谨慎看多) | B+(中性偏多) | B(中性) | C(看空)}
stock: {stock_code}
stock_name: {stock_name}
score: {weighted_score}
verdict: {verdict}
one_liner: "{one_liner}"
expiry: {date + 30天}
validity_period: "30天"
top_risks:
  - {risk1}
  - {risk2}
key_dates:
  - {event1: date}
themes:
  - {sector}
companies:
  - {stock_code}
source: "AI投委会分析"
tags:
  - 投委会
  - {sector}
---

## 核心结论
{one_liner}

## 综合评分
{score_breakdown表格}

## 关键风险
{top_risks列表}

## 看多逻辑
{bull_case}

## 看空逻辑
{bear_case}

## 数据来源
{sources_used列表}

## 免责声明
本报告由AI生成，仅供参考，不构成投资建议。投资有风险，入市需谨慎。
```

---

## 注意事项

1. **不构成投资建议**：所有输出必须包含免责声明
2. **不输出交易指令**：不提供具体买卖价位、数量建议
3. **数据标注来源**：每个数据点必须标注来源（iwencai/MX_FinData/opencli/WebSearch）
4. **无幻觉**：所有结论必须可追溯到 data_bundle.json 中的具体数据
5. **运行时间**：单标的完整分析目标 < 15分钟

## 参考资料

- Schema 合约：`references/context_schema.json`, `references/data_schema.json`, `references/analysis_schema.json`, `references/committee_schema.json`
- 数据源路由：`references/datasource_routing.md`
- 模板文件：`assets/report_template.html`, `assets/minutes_template.md`, `assets/risk_template.md`, `assets/summary_template.md`
- Excel生成器：`assets/evidence_gen.py`
