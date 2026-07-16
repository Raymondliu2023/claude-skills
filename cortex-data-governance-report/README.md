# cortex-data-governance-report

Generate a live Data, AI & ML Governed scorecard report for any PAC squad. Shows current scores, issue breakdown by group, and top 3 actions — pulled directly from Cortex at runtime.

---

## Preconditions

### 1. Claude Code
Install from https://claude.ai/code or via VS Code / JetBrains extension.

Verify: `claude --version`

### 2. Cortex personal access token + MCP config

Add to `~/.claude.json`:

```json
{
  "mcpServers": {
    "cortex-remote": {
      "type": "remote",
      "url": "https://api.getcortexapp.com/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_TOKEN_HERE"
      }
    }
  }
}
```

Get your token: https://app.getcortexapp.com → Settings → API Keys

Verify: `mcp list` shows `cortex-remote`

### 3. Skill file installed

```bash
mkdir -p ~/.claude/skills/cortex-data-governance-report
cp SKILL.md ~/.claude/skills/cortex-data-governance-report/SKILL.md
```

> No registry needed — this skill reads live data from Cortex only.

---

## How to use

Open Claude Code and run:

```
/cortex-data-governance-report
```

Then type a squad name:

```
> matrix
```

Or paste a Cortex squad tag or URL directly.

**Supported squad names:** matrix, autobot, ioc, lumos, callisto, bamboo, bluedolphin, panda, lanturn, capybara, hermes, hotel partner analytics

---

## Full walkthrough

**1. Run and enter squad name**

```
> /cortex-data-governance-report

✅ Cortex connected — fetching live data.

Which squad? (name, tag, or Cortex URL)
> matrix
```

**2. Report generated**

```
📊 Data Governance Report — Matrix Squad
Scorecard: Data, AI & ML Governed
Generated: 2026-07-16 (live from Cortex)
Cortex overview: https://app.getcortexapp.com/admin/scorecards/9015?...&teams=en2d6fb007377d5ec9

| Metric | Count |
|---|---|
| Total datasets | 30 |
| ✅ Done (100%) | 0 |
| Group 3 — Near-complete (75–99%) | 4 |
| Group 2 — Partial (40–74%) | 10 |
| Group 1 — No metadata (<40%) | 15 |
| ❓ Hive-to-UC | 1 |
| Path A (fixable via Ziggy) | 28 |
| Path B (HOPS/conf file) | 0 |

🎯 Top actions:
1. Push 4 near-complete datasets over the line — 1-2 rules each
   (carhire_web_mini_event_search_results_selected, prod_hotwire_redirects,
    web_carhire_pqs_rating, web_carhire_pqs_experience)
2. Batch Ziggy ticket for 15 Group 1 datasets — all need same 7 UC fields
3. Onboard Group 1 datasets to Monte Carlo after metadata is applied

---

## ✅ Done (0)
None yet.

## Group 3 — Near-complete (4)
| Dataset | Score | Path | Failing rules | Cortex |
|---|---|---|---|---|
| carhire_web_mini_event_search_results_selected | 87% | A | column_descriptions, business_critical_monte_carlo | [View](...) |
| prod_hotwire_redirects_redirects_redirect | 87% | A | column_descriptions, business_critical_monitors | [View](...) |
| web_carhire_pqs_rating | 73% | A | column_descriptions, business_criticality, sox_scope, ooh_support | [View](...) |
| web_carhire_pqs_experience | 73% | A | column_descriptions, business_criticality, sox_scope, ooh_support | [View](...) |

## Group 2 — Partial (10)
...

## Group 1 — No metadata (15)
...

---
To fix a dataset: /cortex-data-governance → paste the Cortex URL above
```

---

## Related

- [cortex-data-governance](../cortex-data-governance/) — fix one dataset at a time
- [PAC tribe inventory](https://skyscanner.atlassian.net/wiki/spaces/~raymondliu/pages/2757361877) — full dataset list per squad

---

## Support

Questions → Raymond Liu or [#data-platform-support](https://skyscanner.slack.com/archives/C043JRRJJ)
