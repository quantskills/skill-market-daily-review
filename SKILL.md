---
name: market-daily-review
description: Generate A-share end-of-day market review reports with Pandadata data,
  covering trade-date checks, index performance and valuation, market breadth, limit-up/down
  sentiment, industries/concepts, 龙虎榜, block trades, margin financing, northbound
  holdings, risk notes, and optional scheduled after-close automation. Use when the
  user asks for 今日复盘, 收盘总结, 每日市场报告, A股复盘, 龙虎榜复盘, 北向资金动向, or to set up an automated
  daily market review.
license: GPL-3.0-only
metadata:
  organization: QuantSkills
  organization_url: https://github.com/quantskills
  repository: skill-market-daily-review
  repository_url: https://github.com/quantskills/skill-market-daily-review
  project_type: skill
  collection: market-daily-review
  creator: abgyjaguo
  maintainer: abgyjaguo
quantSkills:
  project_type: skill
  category: monitor
  tags:
  - a-share
  - daily-review
  - market-breadth
  - sentiment
  - pandadata
  platforms:
  - claude-code
  - codex
  - hermes
  - openclaw
  - cursor
  status: stable
  validation_level: runnable
  maintainer_type: official
  summary_zh: 收盘后一句话生成 A 股当日复盘：指数与估值、市场宽度、行业概念热点、龙虎榜、大宗、两融、北向 —— 每个数字可溯源，支持定时自动生成。
  summary_en: A-share end-of-day review skill covering indexes, valuation, breadth,
    sentiment, sectors, themes, and capital-flow clues.
  license: GPL-3.0
  requires:
  - skill-pandadata-api
---

```json qsh-form
{
  "version": 1,
  "task": {
    "placeholder": "补充复盘口径或特别要求（可选）",
    "required": false
  },
  "fields": [
    {
      "key": "date",
      "label": "复盘日期",
      "type": "date",
      "help": "留空时使用最近已完成的 A 股交易日"
    },
    {
      "key": "focus",
      "label": "重点关注",
      "type": "text",
      "placeholder": "例如：北向持仓、半导体板块、龙虎榜"
    }
  ],
  "prompt_template": "{{#task}}任务与材料：\n{{task}}\n\n{{/task}}{{#attachments}}用户上传的材料（已放入工作区）：\n{{attachments}}\n\n{{/attachments}}执行 A 股收盘复盘。{{#date}}以 {{date}} 为目标日期。{{/date}}未指定日期时以最近已完成的 A 股交易日为目标日期。{{#focus}}重点关注 {{focus}}。{{/focus}}先核验交易日，再覆盖指数表现与估值、市场宽度、涨跌停情绪、行业与概念、龙虎榜、大宗交易、两融、北向持仓和风险提示，逐项标注数据接口与日期并说明滞后或缺失数据，输出中文报告。"
}
```

# Market Daily Review

Use this skill to generate factual A-share after-close review reports. Prefer Pandadata as the data source, keep every statistic traceable to an interface and data date, and never invent missing figures.

## Workflow

1. Determine the target date. If the user does not provide one, use the latest completed A-share trading day. Check `get_last_trade_date` and `get_trade_cal`; if the target date is closed, return a short "今日休市" note instead of a full report.
2. Load `pandadata-api` before making real API calls. Use its method index or search script to confirm parameters and fields; do not guess Pandadata signatures.
3. Collect data in this order:
   - Trading calendar and stock universe: `get_last_trade_date`, `get_trade_cal`, `get_trade_list`.
   - Index performance and valuation: `get_index_daily`, `get_index_indicator`.
   - Market breadth and sentiment: `get_stock_daily` or `get_stock_rt_daily`, plus `get_stock_status_change`.
   - Hot sectors and concepts: `get_industry_constituents`, `get_concept_list`, `get_concept_constituents`.
   - Funds and notable trades: `get_lhb_list`, `get_lhb_detail`, `get_block_trade`, `get_margin`, `get_hsgt_hold`.
4. Compute breadth and ranking metrics from raw rows: rising/falling counts, limit-up/limit-down counts, turnover leaders, industry/concept leaders, 龙虎榜 top net buy/sell names, block-trade discount/premium distribution, margin balance change, and northbound holding changes.
5. Generate Markdown using `references/report-template.md`. Save the report to `reports/daily/YYYYMMDD.md` unless the user gives another path.
6. Run `scripts/validate_report.py <report-path>` after writing the report. Fix missing sections, missing source notes, or missing data-date labels before presenting the result.

## Pandadata Reference

Read `references/pandadata-map.md` when planning calls, selecting fields, or deciding how to degrade if a data interface is unavailable. The map is a routing aid only; the exact call contract must still come from `pandadata-api`.

## Report Rules

- Write in Chinese unless the user requests another language.
- Use absolute dates such as `2026-06-11`; avoid ambiguous "today" in the final report body.
- Mark T+1 datasets clearly. Margin financing, northbound holdings, and some exchange disclosures may lag the market date.
- State the limit-up/limit-down counting rules, including whether ST stocks and one-price boards are included.
- Keep the report factual: summarize structure, flows, and anomalies; do not give tomorrow's trading instructions or personalized investment advice.
- When a data call fails, keep the report useful by generating available sections and adding a concise missing-data note under "数据说明".

## Automation

When the user asks for an automated daily review, create an after-close task for trading days only, preferably after `18:30 Asia/Shanghai` so delayed datasets have time to settle. Make the task idempotent: if `reports/daily/YYYYMMDD.md` already exists, regenerate and overwrite it.
