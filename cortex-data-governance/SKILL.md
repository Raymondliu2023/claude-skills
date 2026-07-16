# cortex-data-governance

Fix a single Unity Catalog dataset's **Data, AI & ML Governed** scorecard issues in Cortex — interactively gather UC metadata, detect the fix path, generate SQL notebook (Path A) or conf diff (Path B), and produce a draft Ziggy ticket or PR description.

---

## Trigger

User says "fix dataset", "governance fix", "fix scorecard for", "fix [dataset name]", or pastes a Cortex/Databricks URL for a specific dataset.

---

## Step 0 — First-launch check (run once, silently)

On every run, verify preconditions before doing anything else. If any check fails, stop immediately and show a clear error with fix instructions. Do not proceed.

```
🔍 Checking preconditions...
```

### Check 1 — Skill files present

Verify both files exist:
- `~/.claude/skills/cortex-data-governance/SKILL.md`
- `~/.claude/skills/cortex-data-governance/registry.json`

If missing:
```
❌ Setup incomplete — registry.json not found.

Run the setup steps in README.md:
  cp registry.json ~/.claude/skills/cortex-data-governance/registry.json

Download from: https://github.com/Raymondliu2023/claude-skills
```

### Check 2 — Cortex MCP available

Call `mcp__cortex-remote__listScorecardScores` with tag=`data-ai-ml-governed`, pageSize=1, page=0.

If the tool is unavailable or returns an auth error:
```
❌ Cortex MCP not configured.

You need a Cortex personal access token to use this skill.

Steps:
1. Go to https://app.getcortexapp.com → Settings → API Keys
2. Create a personal access token
3. Add to your Claude Code MCP config (~/.claude.json):

   "cortex-remote": {
     "type": "remote",
     "url": "https://api.getcortexapp.com/mcp",
     "headers": {
       "Authorization": "Bearer YOUR_TOKEN_HERE"
     }
   }

4. Restart Claude Code and try again.
```

If tool exists but returns 403 / no access to scorecard `data-ai-ml-governed`:
```
❌ Cortex access error — you may not have access to the PAC tribe scorecard.
Contact Raymond Liu or check your Cortex permissions.
```

### Check 3 — Registry readable

Load `registry.json`. If it's empty `{}` or missing the `_note` field, show:
```
⚠️ Registry is empty — no pre-populated dataset knowledge found.
Download the latest registry.json from https://github.com/Raymondliu2023/claude-skills
and copy it to ~/.claude/skills/cortex-data-governance/registry.json
```
This is a warning, not a blocker — the skill still works, it will just ask all questions.

### If all checks pass:
```
✅ All preconditions met — ready to fix.
```

Then proceed to Step 1.

---

## Registry

Answers are stored at `~/.claude/skills/cortex-data-governance/registry.json`.

Each entry is keyed by Cortex tag (e.g. `prod_trusted_bronze.internal.car_hire_quotes`):

```json
{
  "prod_trusted_bronze.internal.car_hire_quotes": {
    "squad": "autobot-squad",
    "path": "A",
    "sox_scope": "false",
    "business_criticality": "p2",
    "ooh_support": "false",
    "data_classification": "internal",
    "data_category": "service_data",
    "retention_period": "2 years",
    "table_description": "Car hire quotes returned by Bumblebee per search request.",
    "column_descriptions": {
      "quote_id": "Unique identifier for the quote.",
      "vendor_code": "Car hire vendor code."
    },
    "answered_at": "2026-07-16"
  }
}
```

If a field is not yet answered, omit it — the skill will ask for it on the next run.

---

## Step 1 — Resolve the dataset

Accept any of:
- Cortex scorecard URL (extract `entity=enXXX` param → call `mcp__cortex-remote__getEntityDetails` by ID)
- Cortex tag (e.g. `prod_trusted_bronze.internal.car_hire_quotes`) → call `mcp__cortex-remote__getEntityDetails` by tag
- Databricks URL (`/explore/data/{catalog}/{schema}/{table}`) → reconstruct tag as `{catalog}.{schema}.{table}`
- Dataset name (free text) → search `mcp__cortex-remote__searchCatalog` with `types: ["resource"]`, confirm with user if multiple matches

From the entity details, extract:
- Full tag (canonical key)
- `definition.storage_location` (for path detection)
- `metadata[uc_metadata].columns_without_description` (list of columns needing descriptions)
- `metadata[monte_carlo].is_monitored`
- Current `score` from scorecard (call `mcp__cortex-remote__listScorecardScores` filtered to this entity)
- List of failing rules

---

## Step 2 — Detect fix path

| `storage_location` contains | Path |
|---|---|
| `skyscanner-data-platform-trusted-data-pipeline` | **Path A** — SQL + Ziggy ticket |
| `hops` or SkySpark-style path | **Path B** — conf file + PR |
| `hive_to_uc_intermediate` catalog prefix | **Hive-to-UC** — flag for Data Platform, skip |

If Hive-to-UC: tell the user this dataset is a legacy Hive migration table — fix method unknown, needs Data Platform input. Stop.

---

## Step 3 — Check registry

Load `registry.json`. If the tag exists and all required fields are filled, skip to Step 5.

If tag exists but some fields are missing, ask only for the missing ones.

If tag is new, ask all questions.

---

## Step 4 — Ask metadata questions interactively

Ask one question at a time. Show the dataset name and current score for context.

```
Dataset: prod_trusted_bronze.internal.car_hire_quotes (autobot-squad, 27%)
Failing: column_descriptions, business_criticality, sox_scope, ooh_support_required, data_retention, data_classification, data_category, monte_carlo_monitored

Let's fill in the metadata. Answer each question — type your answer or "skip" to leave it for later.

1. Business criticality? (p1 = critical service / p2 = important / p3 = low)
   > 
2. SOX scope? (yes / no — does this table contain financial or revenue data?)
   > 
3. OOH support required? (yes / no — does an on-call engineer need to respond to alerts on this table out of hours?)
   > 
4. Data classification? (internal / confidential / restricted / public)
   > 
5. Data category? Options:
   business_analytical_data | business_insights_data | business_performance_data |
   inventory_data | inventory_snapshot_data | service_analytical_data | service_data |
   service_snapshot_data | traveller_profile_data | user_behaviour_data |
   user_consent_data | user_preference_data
   > 
6. Data retention period? (e.g. "2 years", "7 years", "indefinite")
   > 
7. Table description? (1-2 sentences describing what this table contains, granularity, and source)
   > 
8. Column descriptions? (paste as "column_name: description" pairs, one per line — or type "skip" to leave blank)
   > 
```

After all questions: show a summary and ask for confirmation before saving.

Save answered fields to `registry.json`.

---

## Step 5 — Generate fix artifact

### Path A — SQL notebook

Generate a Databricks notebook (`.sql` or markdown code block) with all applicable `ALTER TABLE` / `COMMENT ON TABLE` statements pre-filled:

```sql
-- Dataset: prod_trusted_bronze.internal.car_hire_quotes
-- Squad: autobot-squad | Score: 27% | Generated: 2026-07-16

-- Table description
COMMENT ON TABLE prod_trusted_bronze.internal.car_hire_quotes IS
'Car hire quotes returned by Bumblebee per search request.
Granularity: One row per quote per search.';

-- Data classification
ALTER TABLE prod_trusted_bronze.internal.car_hire_quotes
  SET TBLPROPERTIES ( skyscanner.data_classification = "internal" );

-- Data category
ALTER TABLE prod_trusted_bronze.internal.car_hire_quotes
  SET TBLPROPERTIES ( skyscanner.data_category = "service_data" );

-- Business criticality
ALTER TABLE prod_trusted_bronze.internal.car_hire_quotes
  SET TBLPROPERTIES ( skyscanner.business_criticality = "p2" );

-- SOX scope
ALTER TABLE prod_trusted_bronze.internal.car_hire_quotes
  SET TBLPROPERTIES ( skyscanner.sox_scope = "false" );

-- OOH support
ALTER TABLE prod_trusted_bronze.internal.car_hire_quotes
  SET TBLPROPERTIES ( skyscanner.ooh_support = "false" );

-- Retention period
ALTER TABLE prod_trusted_bronze.internal.car_hire_quotes
  SET TBLPROPERTIES ( skyscanner.retention_period = "2 years" );

-- Column descriptions
ALTER TABLE prod_trusted_bronze.internal.car_hire_quotes
  ALTER COLUMN quote_id COMMENT 'Unique identifier for the quote.';
ALTER TABLE prod_trusted_bronze.internal.car_hire_quotes
  ALTER COLUMN vendor_code COMMENT 'Car hire vendor code.';
```

Only include statements for rules that are currently failing. Skip any field already set (score=1 on that rule).

### Path A — Draft Ziggy ticket

Output a ready-to-send Slack message for [#data-platform-support](https://skyscanner.slack.com/archives/C043JRRJJ):

```
Hi Ziggy, please help run the following SQL on prod_trusted_bronze.internal.car_hire_quotes to update the UC metadata.

[paste SQL above]

Thanks!
```

> **Note:** Ziggy ticket format TBC — update this template once the example is received from colleague.

### Path B — Conf diff template

Output a template showing where to add metadata in the SkySpark conf file, following the `save_table_metadata` pattern.

Reference PR: https://github.com/Skyscanner/hops-transformation-spark-pipeline/pull/162/changes

> Path B template TBC — confirm format with Callisto team.

---

## Step 6 — Monte Carlo

If `monte_carlo_monitored = false` is a failing rule, show a reminder after the SQL:

```
⚠️ Monte Carlo: This dataset is not monitored. Onboarding to Monte Carlo is a separate step — 
ask your squad's data engineer to register it in Monte Carlo after the metadata SQL is applied.
```

---

## Step 7 — Output summary

```
✅ cortex-data-governance — [dataset name]

Path: A (Trusted Data Pipeline)
Rules addressed: column_descriptions, business_criticality, sox_scope, ooh_support, data_retention, data_classification, data_category
Rules skipped (still open): monte_carlo_monitored (see note above)

SQL notebook ready — copy into a Databricks notebook and send to Ziggy.
Registry updated: ~/.claude/skills/cortex-data-governance/registry.json
```

---

## SQL property key reference

| Rule | ALTER TABLE key |
|---|---|
| `business_criticality` | `skyscanner.business_criticality` → `"p1"` / `"p2"` / `"p3"` |
| `sox_scope` | `skyscanner.sox_scope` → `"true"` / `"false"` |
| `ooh_support_required` | `skyscanner.ooh_support` → `"true"` / `"false"` |
| `data_classification` | `skyscanner.data_classification` → `"internal"` / `"confidential"` / `"restricted"` / `"public"` |
| `data_category` | `skyscanner.data_category` → valid category string |
| `data_retention` | `skyscanner.retention_period` → e.g. `"2 years"` |
| `dataset_description` | `COMMENT ON TABLE ... IS '...'` |
| `column_descriptions` | `ALTER COLUMN ... COMMENT '...'` per column |
| `table_update_frequency` | `skyscanner.table_update_frequency` → `"daily"` / `"hourly"` / `"unscheduled"` |

Source: [Metadata requirements in Unity Catalog](https://skyscanner.atlassian.net/wiki/spaces/UNDERSTAND/pages/633701327)

---

## Notes

- **Ziggy ticket format:** TBC — update Step 5 Path A once example received from colleague (as of 2026-07-16)
- **Path B conf format:** TBC — confirm with Callisto team
- **Hive-to-UC tables:** 75 across PAC — skip and flag; escalate to Data Platform
- **Registry is shared** — answers from one squad lead benefit others (e.g. similar tables across squads)
- **First verification target:** Matrix squad — Will Chen to test first run
