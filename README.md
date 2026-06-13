# AI 投资委员会 (AI Investment Committee)

多角色并行股票投研分析系统 — Agent Skill 实现。

输入股票代码或名称，自动完成 **预检 → 数据采集 → 5 角色分析 → 投委会合议 → 输出报告 + 知识库沉淀** 的完整投研流水线。

## 特性

- **5 阶段流水线**：Phase 0 预检 → Phase 1 数据采集 → Phase 2 分析引擎 → Phase 3 合议 → Phase 4 输出沉淀
- **5 角色并行分析**：研究员、基本面分析师、技术分析师、舆情分析师、风险官
- **三层数据源架构**：Tier 1 专业 API（iwencai/MX_FinData/westock）→ Tier 2 opencli 优先 + WebSearch 兜底 → Tier 3 知识库上下文
- **JSON 中间合约**：context.json → data_bundle.json → analyses.json → committee.json，Phase 间严格解耦
- **多格式输出**：HTML 仪表盘（必选）+ 纪要 / 风险清单 / 一页纸 / Excel 证据表（按需）
- **知识库联动**：分析结果自动沉淀为信号卡片，支持后续回溯和组合分析
- **加权评分 + 矛盾识别**：6 维度加权评分模型 + 4 条自动矛盾检测规则

## 快速开始

### 安装

```bash
# 1. 将 Skill 安装到 Agent 技能目录
cp -r skill/ ~/.agents/skills/ai-investment-committee/

# 2. 安装 Python 依赖（Excel 生成需要）
pip install openpyxl
```

### 前置依赖

| 依赖 | 用途 | 安装方式 |
|------|------|---------|
| opencli | Tier 2 数据采集（社交媒体/资讯） | `npm i -g opencli` |
| iwencai-unified | Tier 1 金融数据（同花顺问财） | Agent Skill，需配置 `IWENCAI_API_KEY` |
| MX_FinData | Tier 1 金融数据补充（东方财富） | Agent Skill，需配置 `EM_API_KEY` |
| westock-data | Tier 1 K 线/筹码/资金流向 | Agent Skill |
| openpyxl | Excel 证据表生成 | `pip install openpyxl` |

### 环境变量

在项目的 `.env` 文件中配置：

```env
IWENCAI_API_KEY=your_iwencai_key    # 同花顺问财 API Key
EM_API_KEY=your_em_key              # 东方财富 API Key
```

### 使用

在支持 Agent Skill 的 AI 编程助手中执行：

```
/ai-investment-committee 688300
```

可选参数 `--outputs` 控制输出格式：

```
/ai-investment-committee 688300 --outputs html,minutes,risk,summary,excel
```

| 输出 | 参数值 | 说明 |
|------|--------|------|
| HTML 仪表盘 | `html`（默认必选） | 暗色主题可视化总览 |
| 投委会纪要 | `minutes` | 决策过程记录（对话体） |
| 风险清单 | `risk` | 结构化风险矩阵 + 应对预案 |
| 一页纸总结 | `summary` | 面向决策者的 A4 速览 |
| Excel 证据表 | `excel` | 8 Sheet 原始数据追溯 |

## 项目结构

```
.
├── AGENTS.md                           # AI Agent 开发指南
├── README.md                           # 本文件
├── LICENSE
├── .gitignore
└── skill/
    ├── SKILL.md                        # Skill 核心指令（Agent 执行入口）
    ├── assets/
    │   ├── report_template.html        # HTML 仪表盘模板（暗色主题）
    │   ├── minutes_template.md         # 投委会纪要模板
    │   ├── risk_template.md            # 风险清单模板
    │   ├── summary_template.md         # 一页纸总结模板
    │   └── evidence_gen.py             # Excel 证据表生成器（openpyxl）
    └── references/
        ├── context_schema.json         # Phase 0 输出 schema
        ├── data_schema.json            # Phase 1 输出 schema
        ├── analysis_schema.json        # Phase 2 输出 schema
        ├── committee_schema.json       # Phase 3 输出 schema
        └── datasource_routing.md       # 数据源路由规则 + 命令清单
```

## 架构

### 流水线

```
Phase 0        Phase 1           Phase 2            Phase 3         Phase 4
预检      →    数据采集     →    分析引擎      →    合议       →    输出沉淀
context.json   data_bundle.json   analyses.json      committee.json   HTML/MD/XLSX
```

### 角色

| 角色 | 职责 | 主要数据源 |
|------|------|-----------|
| 研究员 | 客观信息聚合，不做主观判断 | iwencai(公告/基本面) + opencli(weixin/google) |
| 基本面分析师 | 财务分析 + 估值判断 + 竞争格局 | iwencai(财务/估值/研报) + MX_FinData |
| 技术分析师 | 走势判断 + 指标分析 + 支撑/阻力 | iwencai(行情/指标) + westock-data |
| 舆情分析师 | 情绪分布 + 争议识别 + 误读检测 | opencli(xueqiu/weibo/zhihu) + MX_FinSearch |
| 风险官 | 反面证据 + 风险评级矩阵 + 监控日历 | iwencai(事件/减持/质押) + opencli(weixin/google) |

### 评分模型

| 维度 | 权重 | 数据映射 |
|------|------|---------|
| 产业逻辑与赛道 | 20% | fundamental.growth_drivers |
| 公司质地与竞争力 | 20% | fundamental.competitive_moat |
| 成长性 | 20% | fundamental.scores.growth |
| 财务健康度 | 15% | fundamental.cash_flow_assessment |
| 估值合理性 | 15% | fundamental.scores.valuation |
| 风险收益比 | 10% | risk + technical 综合 |

**评分等级**：1-3 = 看空 / 4-5 = 中性 / 6-7 = 谨慎看多 / 8-10 = 强烈看多

### 矛盾识别

4 条自动检测规则：

1. **好公司但不是好价格** — 基本面 quality > 7 且 valuation < 3
2. **趋势向上但风险累积** — 技术面 > 6 且风险矩阵含高危评级
3. **市场热度与基本面不匹配** — 舆情热度高 且基本面 growth < 6
4. **信息面正面但结构性风险突出** — 研究员正面 且风险官高危 > 3 条

## 数据源降级策略

```
opencli doctor --no-live
│
├─ 正常 → opencli_first
│   ├─ 优先级1: opencli 专用适配器 (xueqiu/weixin/weibo/zhihu/eastmoney)
│   ├─ 优先级2: opencli 通用搜索 (google/brave/bing)
│   └─ 优先级3: Agent WebSearch (兜底)
│
└─ 失败 → websearch_only
    └─ 直接使用 Agent WebSearch
```

## 输出示例

### 信号卡片（知识库沉淀）

分析完成后自动写入 `03_memory/signals/{stock_name}_committee_{date}.md`，包含：

- 综合评分 + 评分分解
- 一句话定性（one_liner）
- 看多/看空 Top 3 理由
- 关键风险 + 关键日期
- 信号评级（A+/A/B+/B/C）
- 30 天有效期

## 开发

详见 [AGENTS.md](AGENTS.md)。

## 许可

MIT License — 详见 [LICENSE](LICENSE)。

## 免责声明

本项目由 AI 生成，仅供投研参考，**不构成投资建议**。投资有风险，入市需谨慎。
