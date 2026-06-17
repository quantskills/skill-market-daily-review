# 📰 Market Daily Review Skill

[简体中文](README.md) | **English**

> One sentence after the close generates a full A-share daily review: indices & valuation, market breadth, industry/concept hotspots, 龙虎榜 (Dragon-Tiger list), block trades, margin financing, northbound holdings — every number traceable, scheduled automation supported.

<p align="center">
  <img alt="report sections" src="https://img.shields.io/badge/report_sections-7-brightgreen">
  <img alt="data groups" src="https://img.shields.io/badge/data_groups-5-blue">
  <img alt="validator" src="https://img.shields.io/badge/validator-validate__report.py-orange">
  <img alt="automation" src="https://img.shields.io/badge/automation-after--close_cron-9cf">
  <img alt="requires" src="https://img.shields.io/badge/requires-pandadata--api-7c3aed">
  <img alt="license" src="https://img.shields.io/badge/license-GPLv3-blue">
</p>

---

## 📖 What is this

`market-daily-review` is an **Agent Skill** that generates A-share end-of-day review reports from Pandadata. It ships with a fixed report template, a unified data-collection order, explicit T+1 data labeling rules, and an **automatic validator** — after the report is written, `scripts/validate_report.py` checks section completeness, data-source annotations, and the disclaimer; failures are fixed before delivery.

It can be triggered manually ("今天复盘一下") or wired as an after-close scheduled task that writes `reports/daily/YYYYMMDD.md` at 18:30 on every trading day.

> All data contracts come from the sibling skill [`pandadata-api`](https://github.com/quantskills/skill-pandadata-api).

---

## ⚡ Generation Pipeline

```mermaid
flowchart TD
    A["⏰ Manual trigger or cron 18:30"] --> B{"📅 Trading-day check<br/>get_last_trade_date / get_trade_cal"}
    B -->|market closed| Z["Output 'market closed today' and stop"]
    B -->|trading day| C["📊 Data collection (5 groups)"]
    C --> C1["Indices & valuation<br/>index_daily · index_indicator"]
    C --> C2["Market breadth<br/>stock_daily · status_change"]
    C --> C3["Industries & concepts<br/>industry/concept constituents"]
    C --> C4["龙虎榜 & block trades<br/>lhb_list/detail · block_trade"]
    C --> C5["Margin & northbound<br/>margin · hsgt_hold (T+1 labeled)"]
    C1 --> D["🧮 Metric computation<br/>adv/dec counts · limit moves · hotspot ranks · net buys"]
    C2 --> D
    C3 --> D
    C4 --> D
    C5 --> D
    D --> E["📝 Render template<br/>references/report-template.md"]
    E --> F{"✅ validate_report.py"}
    F -->|FAIL| E
    F -->|OK| G["💾 reports/daily/YYYYMMDD.md"]

    style A fill:#e3f2fd,stroke:#1976d2
    style F fill:#fff3e0,stroke:#f57c00
    style G fill:#e8f5e9,stroke:#388e3c
    style Z fill:#f5f5f5,stroke:#9e9e9e
```

---

## 🗂️ Report Sections × Interface Map

| Section | Interfaces | Metrics produced |
|---|---|---|
| 1️⃣ Index overview & valuation | `get_index_daily` · `get_index_indicator` | Change %, turnover, PE/PB, valuation percentile |
| 2️⃣ Market breadth & sentiment | `get_stock_daily` / `get_stock_rt_daily` · `get_stock_status_change` | Advancers/decliners, limit-up/down counts (convention stated), turnover leaders, new ST / cap removals |
| 3️⃣ Industry & concept hotspots | `get_industry_constituents` · `get_concept_list` · `get_concept_constituents` | Industry/concept gainers with representative stocks |
| 4️⃣ 龙虎榜 & block trades | `get_lhb_list` · `get_lhb_detail` · `get_block_trade` | Listing reasons, seat net buys, block premium/discount distribution |
| 5️⃣ Margin & northbound | `get_margin` · `get_hsgt_hold` | Margin balance changes, top northbound adds/cuts (**T+1 data date always labeled**) |
| 6️⃣ Anomalies & risk notes | aggregate of the above | ST changes, unusual turnover, consecutive moves |
| 7️⃣ Data notes | — | Interfaces used, data cutoff, missing/degraded items, statistical conventions |

---

## 🚀 Quick Start

### 1️⃣ Install (together with pandadata-api)

```bash
# Claude Code (global)
cp -r skill-pandadata-api       ~/.claude/skills/pandadata-api
cp -r skill-market-daily-review ~/.claude/skills/market-daily-review
```

### 2️⃣ Manual trigger

```text
今天复盘一下
生成 20260610 的收盘总结
看看今天龙虎榜和北向资金动向
```

### 3️⃣ Scheduled daily review

One sentence to the Agent is enough:

```text
帮我设置每个交易日 18:30 自动生成当日复盘
```

The task is **idempotent**: if a report already exists for the day it is regenerated and overwritten; on non-trading days it only outputs "market closed today". 18:30 is chosen so delayed disclosures (margin, northbound) have landed.

### 4️⃣ Validate a report

```bash
python scripts/validate_report.py reports/daily/20260611.md
# OK            -> passed
# FAIL + list   -> missing sections/annotations listed; fix before delivery
```

Checks include: 9 required sections, data-source notes, data date/cutoff, T+1 labels for margin & northbound, and the not-investment-advice disclaimer.

---

## 📦 Directory Layout

```
market-daily-review/
├── SKILL.md                          # Skill entry: workflow, report rules, automation conventions
├── references/
│   ├── pandadata-map.md              # 🧭 Section-to-interface routing table + degradation strategy
│   └── report-template.md            # 📄 Review report Markdown template (with convention placeholders)
├── scripts/
│   └── validate_report.py            # ✅ Report completeness validator
└── agents/
    └── openai.yaml                   # OpenAI/Codex adapter
```

---

## 📐 Core Constraints

| Constraint | Description |
|---|---|
| 📅 Absolute dates | Report bodies use absolute dates like `2026-06-11`, never a vague "today" |
| 🕐 T+1 always labeled | Delayed disclosures (margin, northbound) must carry their actual data date |
| 📏 Conventions stated | Whether limit-move counts include ST or one-tick boards must be written out |
| 🛟 Graceful degradation | If market-wide/concept aggregation fails, keep usable sections (indices, 龙虎榜); record gaps in "Data notes" — **never estimate numbers** |
| 🚫 Describe, don't recommend | Factual summarization and structure only; no next-day trading advice |

---

## ⚠️ Disclaimer

Reports are factual market summaries for research reference only. Nothing here constitutes investment advice.

## 📜 License

This project is licensed under the GNU General Public License v3.0. See [LICENSE](LICENSE).

## 🐼 PandaAI / QUANTSKILLS Community

<div align="center">
  <img src="https://raw.githubusercontent.com/quantskills/.github/main/profile/assets/pandaai-community-qr.jpg" alt="PandaAI community QR code" width="220">
  <br>
  <sub>Scan the QR code to join the PandaAI community for QUANTSKILLS skills, agent workflows, and quantitative research practice.</sub>
</div>
