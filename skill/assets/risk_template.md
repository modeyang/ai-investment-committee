# {{stock_name}} 风险清单

**日期：** {{date}}
**标的：** {{stock_name}} ({{stock_code}})
**风险评分：** {{risk.overall_score}}/10（10=最安全）

---

## 风险评级矩阵

| 类别 | 评级 | 概率 | 影响 | 详情 |
|------|------|------|------|------|
{{risk_matrix_table}}

---

## 高危风险详解

{{#top_risks}}
### {{category}}

- **评级：** {{rating}}
- **触发条件：** {{trigger_condition}}
- **监控指标：** {{monitoring_indicator}}
- **应对预案：** {{response_plan}}

{{/top_risks}}

---

## 低风险项

{{low_risks_list}}

---

## 监控日历

| 日期 | 事件 | 预期影响 |
|------|------|---------|
{{monitoring_calendar_table}}

---

## 止损/止盈参考

- **关键支撑位：** {{key_support}}
- **关键阻力位：** {{key_resistance}}

---

*本清单由AI投资委员会系统生成，不构成投资建议。*
