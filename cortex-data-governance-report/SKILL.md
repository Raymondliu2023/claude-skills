# cortex-data-governance-report

Generate a live Data, AI & ML Governed scorecard report for a specific PAC squad — shows current scores, issue breakdown, and top actions. Pulls directly from Cortex at runtime so the report is always fresh.

---

## Trigger

User says "governance report", "data governance report for", "list all issues for [squad]", "how is [squad] doing on governance", or `/cortex-data-governance-report`.

---

## Step 0 — First-launch check

Verify preconditions before proceeding. Stop with clear instructions if any fail.

### Check 1 — Cortex MCP available
Call `mcp__cortex-remote__listScorecardScores` with tag=`data-ai-ml-governed`, pageSize=1, page=0.

If unavailable or auth error:
```
❌ Cortex MCP not configured.

Add to ~/.claude.json:
  "cortex-remote": {
    "type": "remote",
    "url": "https://api.getcortexapp.com/mcp",
    "headers": { "Authorization": "Bearer YOUR_TOKEN" }
  }

Get your token: https://app.getcortexapp.com → Settings → API Keys
Then restart Claude Code.
```

### If check passes:
```
✅ Cortex connected — fetching live data.
```

---

## Step 1 — Resolve squad

Accept any of:
- Squad name (e.g. `matrix`, `autobot`, `callisto`, `lumos`) — map to Cortex tag
- Cortex squad tag (e.g. `matrix-squad`)
- Cortex team URL (e.g. `https://app.getcortexapp.com/admin/catalog/entity/matrix-squad`)

**Squad tag mapping (PAC tribe):**

| Input | Cortex tag | Entity ID |
|---|---|---|
| matrix | matrix-squad | en2d6fb007377d5ec9 |
| autobot | autobot-squad | en2d6faf55687c76a8 |
| ioc / intelligent octopus | intelligentoctopus-squad | en2d6fafcaf5d624f9 |
| lumos | lumos-squad | en2d6faffa957b752c |
| callisto | callisto-squad | en2d6faf6813acd19e |
| bamboo | bamboo-squad | en2d6faf5a161e42df |
| bluedolphin | bluedolphin-squad | en2d6faf61b50fc965 |
| panda | panda-squad | en2d6fb062598274ec |
| lanturn | lanturn-squad | en2d6fafda04ad4884 |
| capybara | capybara-squad | en2d6faf6cdab7d0e8 |
| hermes | hermes-squad | en33b57276dde55de5 |
| hotel partner analytics / hpa | hotel-partner-analytics-squad | en3117d7025c4ca98d |

If input doesn't match, ask for clarification.

---

## Step 2 — Fetch live data from Cortex

Call `mcp__cortex-remote__listScorecardScores` with:
- `tag`: `data-ai-ml-governed`
- `pageSize`: 250
- Scan all pages until no more results

Filter entries where `service.owners.groups` contains the squad tag.

For each dataset extract:
- `service.name` — display name
- `service.tag` — Cortex tag
- `service.id` — entity ID
- `score.summary.percentage` — current score
- `score.rules` where `score=0` — failing rule names

---

## Step 3 — Classify datasets into groups

| Group | Score | Criteria |
|---|---|---|
| ✅ Done | 100% | All rules passing |
| Group 3 — Near-complete | 75–99% | 1–3 rules away from passing |
| Group 2 — Partial | 40–74% | Some UC metadata set, several rules still failing |
| Group 1 — No metadata | <40% | Most or all UC fields missing |
| ❓ Hive-to-UC | any | `hive_to_uc_intermediate` catalog prefix — not addressable via Path A or B |

Detect path per dataset:
- `storage_location` contains `skyscanner-data-platform-trusted-data-pipeline` → Path A
- `hive_to_uc_intermediate` catalog → Hive-to-UC
- otherwise → Path B

---

## Step 4 — Generate report

Output the following sections:

### Header
```
📊 Data Governance Report — {Squad Name}
Scorecard: Data, AI & ML Governed
Generated: {date} (live from Cortex)
Cortex overview: https://app.getcortexapp.com/admin/scorecards/9015?groupBy=&scoresTab=All&sortBy=level_asc&teams={entity_id}
```

### Summary table
```
| Metric | Count |
|---|---|
| Total datasets | X |
| ✅ Done (100%) | X |
| Group 3 — Near-complete (75–99%) | X |
| Group 2 — Partial (40–74%) | X |
| Group 1 — No metadata (<40%) | X |
| ❓ Hive-to-UC (needs Data Platform) | X |
| Path A (fixable via Ziggy) | X |
| Path B (HOPS/conf file) | X |
```

### Top actions
Generate 3 prioritised actions based on what will have the biggest impact:
```
🎯 Top actions:
1. [highest impact action — usually pushing near-complete datasets over the line]
2. [second action — batch Ziggy ticket for Group 1 datasets]
3. [third action — e.g. Monte Carlo onboarding or specific metadata fills]
```

Rules for ranking actions:
- Group 3 (near-complete) first — fewest rules to fix, fastest wins
- Group 1 batch — highest volume but needs full metadata fill
- Monte Carlo — flag separately as it's a different process

### Dataset breakdown (grouped)

Show each group as a collapsed section with a table:

```
## ✅ Done ({count})
| Dataset | Score | Cortex |
|---|---|---|
| {name} | 100% | [View](https://app.getcortexapp.com/...) |

## Group 3 — Near-complete ({count})
| Dataset | Score | Path | Failing rules | Cortex |
|---|---|---|---|---|
| {name} | {score}% | A | column_descriptions, matched_monitors | [View](...) |

## Group 2 — Partial ({count})
| Dataset | Score | Path | Failing rules | Cortex |
|---|---|---|---|---|

## Group 1 — No metadata ({count})
| Dataset | Score | Path | Failing rules | Cortex |
|---|---|---|---|---|

## ❓ Hive-to-UC ({count})
| Dataset | Score | Cortex |
|---|---|---|
| {name} | 50% | [View](...) |
Note: These are legacy Hive migration tables. Fix method unknown — contact Data Platform.
```

Cortex URL format per dataset:
`https://app.getcortexapp.com/admin/scorecards/9015?groupBy=&scoresTab=All&sortBy=level_asc&teams={squad_entity_id}&entity={dataset_entity_id}`

### Footer
```
To fix a dataset: /cortex-data-governance → paste the Cortex URL
Full tribe inventory: https://skyscanner.atlassian.net/wiki/spaces/~raymondliu/pages/2757361877
```

---

## Notes

- Report is always live — re-run anytime to see updated scores after fixes are applied
- Cortex URL in header links directly to filtered squad view in scorecard
- If a squad has >50 datasets, paginate the dataset breakdown — show summary table and top actions first, then offer "show full list?" before rendering all rows
- Hive-to-UC datasets always shown last — they are blocked on Data Platform, not the squad
