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

### Check 3 — Databricks token (optional — for auto-notebook creation)

Check if `DATABRICKS_TOKEN` is set in the environment:
```bash
echo $DATABRICKS_TOKEN
```

If empty or unset: note silently — the skill will fall back to showing the SQL for manual copy. Do NOT block the run. Show in precondition summary:
```
⚠️ DATABRICKS_TOKEN not set — will show SQL for manual copy instead of auto-creating notebook.
   To enable auto-create: export DATABRICKS_TOKEN=your_token in your shell profile.
```

If set: note silently as `✅ Databricks token found — will auto-create notebook.`

### Check 4 — Registry readable

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

### Step 4a — Show pending questions upfront

Before asking anything, determine which questions are actually needed for this dataset (registry fields missing + rules still failing). Then display **only those questions** with their numbers, options, and hints — so the user can prepare answers in advance.

```
Dataset: prod_trusted_bronze.internal.car_hire_quotes (autobot-squad, 27%)
Failing: column_descriptions, business_criticality, sox_scope, ooh_support_required, data_retention, data_classification, data_category

Here are the questions I'll ask (5 in total). Take a moment to prepare, then reply with your answers.

1. Business criticality? → p1 (critical) / p2 (important) / p3 (low)
2. SOX scope? → yes / no (does this table contain financial or revenue data?)
3. OOH support required? → yes / no (does on-call need to respond to alerts on this table out of hours?)
4. Data classification? → internal / confidential / restricted / public
5. Data retention period? → e.g. "2 years", "7 years", "indefinite"

You can answer all at once:  1.p2; 2.no; 3.no; 4.internal; 5.2 years
Or type "ready" to be asked one by one.
Type "skip" for any answer you want to leave for later.
```

Only list questions for fields that are actually missing. If the registry already has `business_criticality` filled, don't show Q1. The numbering restarts from 1 for the pending questions (not fixed positions).

**Column descriptions** — if needed, always ask separately after the other questions, since the format is multi-line. In the upfront list, show it as:
```
N. Column descriptions? → answer after the others, or type "skip"
```

**Data category** — include the 12 options in the upfront list just like the one-by-one flow, so the user can pick before answering.

---

### Step 4b — Accept answers

**Batch format (preferred):** User replies with `1.answer; 2.answer; ...`

Parse each `N.answer` by splitting on `;`, then map back to the question list shown in Step 4a. Strip whitespace. Accept both `1. p2` and `1.p2`.

For column descriptions in batch: accept `N.col1: desc, col2: desc` with comma-separated pairs. If the user typed `N.skip`, mark as skipped.

**One-by-one format:** If the user types `ready`, ask each question individually in sequence (original flow).

**Mixed:** If only some answers are provided in batch (e.g. `1.p2; 2.no`), accept those and ask the remaining questions one by one.

After all answers collected: show a confirmation summary and ask for confirmation before saving.

```
Here's what I'll save:
  business_criticality: p2
  sox_scope: false
  ooh_support: false
  data_classification: internal
  retention_period: 2 years

Confirm? (yes / edit N)
```

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

### Path A — Create Databricks notebook and draft Ziggy message

**If `DATABRICKS_TOKEN` is set — auto-create the notebook:**

Build the notebook content as a base64-encoded Databricks source file (format: `SOURCE`, language: `SQL`):

```
NOTEBOOK_CONTENT="-- Dataset: {tag}\n-- Squad: {squad} | Score: {score}% | Generated: {date}\n\n{all SQL statements joined with \n\n}"
NOTEBOOK_B64=$(echo -e "$NOTEBOOK_CONTENT" | base64)
NOTEBOOK_PATH="/Users/{squad}/governance/{table_name}_{date}"
```

Call the Databricks workspace import API:

```bash
curl -s -X POST https://skyscanner-prod.cloud.databricks.com/api/2.0/workspace/import \
  -H "Authorization: Bearer $DATABRICKS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"path\": \"$NOTEBOOK_PATH\",
    \"format\": \"SOURCE\",
    \"language\": \"SQL\",
    \"content\": \"$NOTEBOOK_B64\",
    \"overwrite\": false
  }"
```

If the API call succeeds (HTTP 200): extract the notebook path and construct the URL:
```
NOTEBOOK_URL="https://skyscanner-prod.cloud.databricks.com/editor/notebooks$(echo $NOTEBOOK_PATH | sed 's|/|%2F|g')?o=1849662692269217"
```

Show the user:
```
✅ Notebook created: {NOTEBOOK_URL}
```

If the API call fails (auth error, path conflict, etc.): fall back to the manual flow below and show the error message.

---

**If `DATABRICKS_TOKEN` is not set — manual fallback:**

Show the SQL block (already output above) and tell the user:

```
Save the SQL above into a new Databricks notebook at:
https://skyscanner-prod.cloud.databricks.com

Copy the notebook URL from your browser, then paste it below.
> 
```

Wait for the user to paste the notebook URL before continuing.

---

**Draft Ziggy Slack message (always — auto or manual):**

Once the notebook URL is known, output the ready-to-send message:

```
Hi Ziggy, please help update UC metadata for the following table, thanks.

Table: {catalog}.{schema}.{table_name}
Script: {NOTEBOOK_URL}
```

**Note:** Ziggy only needs the table name and notebook link — no SQL inline. Keep it brief.

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

### Step 7a — Offer remaining datasets

After the summary, ask:

```
Would you like to see the remaining datasets for [squad-name] that still need fixes? (yes / no)
```

**Squad name source:** Use the `squad` field from the registry entry just saved (e.g. `autobot-squad`). If the registry entry has no `squad` field (first-time run), ask: "What's your squad name? (e.g. autobot-squad)" and save it to the registry entry before querying.

**If yes:** Call `mcp__cortex-remote__listScorecardScores` with `tag: "data-ai-ml-governed"`, then filter results to entities where the Cortex team/owner matches the squad. Sort by score ascending (worst first). Cap display at **10 datasets** — show a count of the rest.

Display format:

```
Remaining datasets for autobot-squad — 12 still need fixes (showing worst 10):

 #  Dataset                                          Score   Failing rules
 1  prod_trusted_bronze.internal.car_hire_quotes     27%     column_descriptions, business_criticality, sox_scope (+3)
 2  prod_trusted_bronze.internal.car_hire_vendors    31%     column_descriptions, data_category, data_retention (+1)
 3  ...

Run the skill again with any dataset name or Cortex URL to fix the next one.
```

Truncate rule names at 3 in the table — show `(+N more)` if there are more.

**If no:** End the session cleanly with no further output.

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

- **Ziggy ticket format:** Confirmed 2026-07-16 — save SQL to Databricks notebook, send notebook link to Ziggy in #data-platform-support. Keep message brief: table name + notebook URL only.
- **Path B conf format:** TBC — confirm with Callisto team
- **Hive-to-UC tables:** 75 across PAC — skip and flag; escalate to Data Platform
- **Registry is shared** — answers from one squad lead benefit others (e.g. similar tables across squads)
- **First verification target:** Matrix squad — Will Chen to test first run
