# AGENTS.md — AI 投资委员会 开发指南

本文档面向 AI Agent（编程助手），提供项目架构、开发约定和优化指南。  
修改本项目代码前**必须**阅读本文件。

---

## 1. 项目定位

AI 投资委员会是一个 **Agent Skill**（非独立应用），由 AI 编程助手（如 Qoder/Cursor/Claude Code）在用户触发时加载执行。

核心契约：**用户输入股票代码 → Agent 自主执行 5 阶段流水线 → 输出结构化投研报告**。

运行环境：Agent 运行时（非 Python 进程），通过 SKILL.md 中的 `allowed-tools` 声明所需工具。

---

## 2. 架构决策

### 2.1 模块化管道（非单体）

5 个 Phase 通过 **JSON 中间合约** 串联，每个 Phase 只读取上一阶段输出：

```
Phase 0 → context.json
Phase 1 → data_bundle.json   (读取 context.json)
Phase 2 → analyses.json      (读取 data_bundle.json + context.json)
Phase 3 → committee.json     (读取 analyses.json)
Phase 4 → HTML/MD/XLSX       (读取 committee.json + analyses.json + data_bundle.json)
```

**约束**：Phase N 不直接调用 Phase N-1 的工具。如需回退重采集，在 Phase 内部自行处理。

### 2.2 JSON Schema 合约

每个中间 JSON 有对应的 schema 文件（`references/*_schema.json`）。修改数据结构时**必须**同步更新 schema。

| 文件 | 生产方 | 消费方 |
|------|--------|--------|
| context_schema.json | Phase 0 | Phase 1, 2 |
| data_schema.json | Phase 1 | Phase 2, 4 |
| analysis_schema.json | Phase 2 | Phase 3, 4 |
| committee_schema.json | Phase 3 | Phase 4 |

### 2.3 三层数据源

| 层级 | 工具 | 特征 |
|------|------|------|
| Tier 1 | iwencai-unified, MX_FinData, westock-data | 结构化、高可靠、需 API Key |
| Tier 2 | opencli（优先）→ WebSearch（兜底） | 非结构化、需降级策略 |
| Tier 3 | 知识库文件（02_knowledge/, 03_memory/） | 历史上下文、可选 |

### 2.4 并行子 Agent 策略

Phase 1 和 Phase 2 各使用 **5 个并行 GeneralPurpose 子 Agent**。子 Agent 是无状态的，通过 prompt 接收数据、返回 JSON。主流程负责合并。

---

## 3. 文件职责

### 3.1 核心入口

- **`skill/SKILL.md`** — Agent 的完整执行指令。包含 frontmatter（name/allowed-tools）、5 阶段流水线步骤、评分权重、矛盾规则、输出模板引用。这是 Agent 唯一读取的"程序"文件。

### 3.2 模板（assets/）

| 文件 | 用途 | 修改场景 |
|------|------|---------|
| report_template.html | HTML 仪表盘骨架（暗色主题 CSS + 8 section） | 调整报告布局/样式 |
| minutes_template.md | 投委会纪要结构 | 调整纪要格式 |
| risk_template.md | 风险清单结构 | 调整风险展示 |
| summary_template.md | 一页纸总结结构 | 调整总结格式 |
| evidence_gen.py | Excel 8-Sheet 生成器 | 修改 Sheet 结构/样式 |

**注意**：模板是 Agent 生成输出时的**参考骨架**，不是 Jinja2/Mustache 模板引擎。Agent 读取模板后，用实际数据填充生成最终文件。

### 3.3 Schema 合约（references/）

JSON Schema draft-07 格式。修改时需同步：
1. 对应的 schema 文件
2. SKILL.md 中的字段说明
3. evidence_gen.py 中的 Sheet 构建逻辑（如涉及 data_bundle 字段变更）

### 3.4 数据源路由（references/datasource_routing.md）

完整的 iwencai/opencli/WebSearch 命令清单 + 降级策略决策树。新增数据源时更新此文件。

---

## 4. 开发约定

### 4.1 SKILL.md 修改规则

- frontmatter 的 `allowed-tools` 必须与实际使用的工具保持一致
- Phase 步骤编号格式：`Step X.Y`（如 Step 0.1, Step 3.2）
- 新增参数时在"输入规范 > 参数解析"节同步说明
- 矛盾规则编号：`规则N`，修改时同步 committee_schema.json 的 `contradictions_detected` 字段

### 4.2 数据来源标注

每个数据点**必须**在 JSON 中标注 `source` 字段，值域见 `datasource_routing.md` 第 5 节。

### 4.3 信号卡片对齐

信号卡片（Phase 4.3）的 frontmatter **必须**与 `00_meta/schemas/signal_schema.yaml` 的 required 字段保持一致：
- `title`, `type`, `status`, `created`, `verification_status`

### 4.4 evidence_gen.py 修改

- 依赖：仅 `openpyxl`（标准库 + openpyxl）
- 输入：`sys.argv[1]` = data_bundle.json 路径
- 输出：`sys.argv[2]` = .xlsx 路径
- 修改 Sheet 时需同步更新 `build_*` 函数和 README 中的 Sheet 列表

### 4.5 测试方法

```bash
# 1. Python 语法检查
python3 -m py_compile skill/assets/evidence_gen.py

# 2. evidence_gen.py 冒烟测试
echo '{}' | python3 skill/assets/evidence_gen.py /dev/stdin /tmp/test.xlsx

# 3. JSON Schema 合法性
for f in skill/references/*.json; do
  python3 -c "import json; json.load(open('$f'))" && echo "OK: $f"
done

# 4. 文件完整性检查（应有 11 个源文件）
find skill/ -type f ! -path '*__pycache__*' | wc -l  # 期望输出: 11
```

---

## 5. 扩展指南

### 5.1 新增分析角色

1. 在 `SKILL.md` Phase 2 新增角色节（角色名/职责/输出字段/评分）
2. 更新 `references/analysis_schema.json`，在顶层 properties 中新增角色 key
3. 更新 `references/datasource_routing.md` 的路由表和命令清单
4. 更新 `SKILL.md` Phase 3 评分权重表（确保权重总和 = 100%）
5. 更新 `skill/assets/report_template.html` 新增对应 section
6. 更新 `skill/assets/minutes_template.md` 新增角色汇报节

### 5.2 新增数据源

1. 在 `references/datasource_routing.md` 新增命令 + 降级路径
2. 如属 Tier 1：在 `SKILL.md` Phase 1 Tier 1 表新增行
3. 如属 Tier 2：在降级策略决策树中新增分支
4. 更新 `references/data_schema.json`（如需新字段）
5. 更新 `references/datasource_routing.md` 第 5 节来源标注表

### 5.3 新增输出格式

1. 在 `skill/assets/` 新增模板文件
2. 在 `SKILL.md` 输入规范中新增 `--outputs` 可选值
3. 在 `SKILL.md` Phase 4 Step 4.2 新增生成逻辑
4. 更新 `SKILL.md` 输出规范表

### 5.4 修改评分模型

1. 更新 `SKILL.md` Phase 3 Step 3.1 权重表（总和必须 = 100%）
2. 更新 `references/committee_schema.json` 的 `score_breakdown` 字段
3. 更新 `skill/assets/evidence_gen.py` 的 `build_scores()` 函数
4. 更新信号卡片模板中的 `score` 字段说明

---

## 6. 已知限制 & 优化方向

| 限制 | 当前状态 | 优化方向 |
|------|---------|---------|
| 单标的串行 | 5 角色并行但标的间串行 | 批量分析支持（多标的并行） |
| 模板静态 | HTML 模板为静态 CSS | 可考虑 Chart.js 动态图表 |
| 无增量更新 | 每次全量采集 | 支持增量更新（对比上次 data_bundle） |
| 矛盾规则硬编码 | 4 条固定规则 | 可配置规则引擎 + 历史回测 |
| 评分主观 | Agent 主观打分 | 引入量化因子校准评分 |
| 信号卡片单向 | 只写不读 | 组合级信号聚合 + 信号过期自动归档 |
| 无回测验证 | 无历史验证 | 信号准确率回测框架 |

---

## 7. 关键路径文件

修改影响范围排序（从高到低）：

1. **`skill/SKILL.md`** — 改动影响全部 Phase，是最高风险文件
2. **`skill/references/data_schema.json`** — Phase 1→2→4 全部消费，改动级联最广
3. **`skill/references/analysis_schema.json`** — Phase 2→3→4 消费
4. **`skill/references/datasource_routing.md`** — 影响数据采集完整性
5. **`skill/assets/report_template.html`** — 仅影响 HTML 输出
6. **`skill/assets/evidence_gen.py`** — 仅影响 Excel 输出

---

## 8. 提交约定

```
feat(phase2): 新增宏观经济分析师角色
fix(schema): data_schema.json 补充 free_cash_flow 字段
refactor(phase1): opencli 降级策略改为超时触发
docs(skill): 更新评分权重说明
test(evidence): 新增空输入冒烟测试
```

格式：`type(scope): description`

| type | 说明 |
|------|------|
| feat | 新功能/新角色/新数据源 |
| fix | 修复 bug 或逻辑错误 |
| refactor | 重构（不改变行为） |
| docs | 文档更新 |
| test | 测试相关 |
| chore | 构建/工具链 |

| scope | 说明 |
|-------|------|
| phase0-4 | 对应 Phase |
| schema | JSON Schema |
| routing | 数据源路由 |
| template | 输出模板 |
| evidence | Excel 生成器 |
| skill | SKILL.md |
