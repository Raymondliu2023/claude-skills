# cortex-data-governance

A Claude Code skill for PAC tribe engineers to fix **Data, AI & ML Governed** scorecard issues on Unity Catalog datasets.

It resolves any dataset identifier, shows all pending questions upfront so you can prepare, supports batch answering in one shot, generates a pre-filled SQL notebook ready to send to Ziggy, and lists remaining datasets for your squad after each fix.

---

## Preconditions

Before you can run this skill you need all four of the following:

### 1. Claude Code

Install from https://claude.ai/code or via the VS Code / JetBrains extension.

Verify: `claude --version`

### 2. Cortex personal access token

1. Go to https://app.getcortexapp.com → top-right avatar → **Settings** → **API Keys**
2. Click **Create API Key** — give it a name like `claude-code-governance`
3. Copy the token

### 3. Cortex MCP configured in Claude Code

Add the following to your Claude Code global MCP config at `~/.claude.json` (create it if it doesn't exist):

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

Replace `YOUR_TOKEN_HERE` with the token from step 2. Restart Claude Code after saving.

Verify: open Claude Code and run `mcp list` — you should see `cortex-remote` listed.

### 4. Skill files installed

```bash
# Create the skill directory
mkdir -p ~/.claude/skills/cortex-data-governance

# Copy the skill files
cp SKILL.md ~/.claude/skills/cortex-data-governance/SKILL.md
cp registry.json ~/.claude/skills/cortex-data-governance/registry.json
```

---

## How to use

Open Claude Code in any directory and run:

```
/cortex-data-governance
```

Then paste any of the following when prompted:

- **Cortex URL** — e.g. `https://app.getcortexapp.com/admin/scorecards/9015?...&entity=en3d1f29a99f493f6a`
- **Databricks URL** — e.g. `https://skyscanner-prod.cloud.databricks.com/explore/data/prod_trusted_bronze/internal/car_hire_quotes`
- **Dataset name** — e.g. `car_hire_quotes` (skill will search and confirm the match)
- **Cortex tag** — e.g. `prod_trusted_bronze.internal.car_hire_quotes`

The skill will:
1. Resolve the dataset and show current score + failing rules
2. Auto-detect the fix path (Path A / Path B / Hive-to-UC)
3. Show all pending questions upfront — so you can prepare answers before it asks
4. Accept answers all at once in one reply, or one by one
5. Generate a pre-filled SQL notebook
6. Output a draft Slack message ready to send to Ziggy
7. Offer to show remaining datasets for your squad that still need fixes

---

## Full walkthrough

Here is what a complete run looks like end-to-end.

**1. Run the skill and paste a dataset**

```
> /cortex-data-governance

🔍 Checking preconditions...
✅ All preconditions met — ready to fix.

Paste a Cortex URL, Databricks URL, dataset name, or Cortex tag:
> https://skyscanner-prod.cloud.databricks.com/explore/data/prod_trusted_bronze/internal/car_hire_quotes
```

**2. Skill resolves the dataset and shows current state**

```
Dataset: prod_trusted_bronze.internal.car_hire_quotes
Squad: autobot-squad
Score: 27% (4/15 rules passing)
Path: A (Trusted Data Pipeline)

Failing rules:
  - column_descriptions
  - data_classification
  - data_category
  - business_criticality
  - sox_scope
  - ooh_support_required
  - data_retention
  - monte_carlo_monitored
```

**3. Skill shows all pending questions upfront**

```
Here are the questions I'll ask (7 in total). Take a moment to prepare, then reply with your answers.

1. Business criticality? → p1 (critical) / p2 (important) / p3 (low)
2. SOX scope? → yes / no (does this table contain financial or revenue data?)
3. OOH support required? → yes / no (does on-call need to respond to alerts on this table out of hours?)
4. Data classification? → internal / confidential / restricted / public
5. Data category? →
   business_analytical_data | business_insights_data | business_performance_data |
   inventory_data | inventory_snapshot_data | service_analytical_data | service_data |
   service_snapshot_data | traveller_profile_data | user_behaviour_data |
   user_consent_data | user_preference_data
6. Data retention period? → e.g. "2 years", "7 years", "indefinite"
7. Table description? → 1-2 sentences describing what this table contains, granularity, and source

You can answer all at once:  1.p2; 2.no; 3.no; 4.internal; 5.service_data; 6.2 years; 7.Car hire quotes returned by Bumblebee per search request.
Or type "ready" to be asked one by one.
Type "skip" for any answer you want to leave for later.
```

**4. Answer all at once (batch mode)**

```
> 1.p2; 2.no; 3.no; 4.internal; 5.service_data; 6.2 years; 7.Car hire quotes returned by Bumblebee per search request. One row per quote per search session.

Here's what I'll save:
  business_criticality: p2
  sox_scope: false
  ooh_support: false
  data_classification: internal
  data_category: service_data
  retention_period: 2 years
  table_description: Car hire quotes returned by Bumblebee per search request. One row per quote per search session.

Confirm? (yes / edit N)
> yes
```

**5. Skill generates SQL**

```sql
-- Dataset: prod_trusted_bronze.internal.car_hire_quotes
-- Squad: autobot-squad | Score: 27% | Generated: 2026-07-21

COMMENT ON TABLE prod_trusted_bronze.internal.car_hire_quotes IS
'Car hire quotes returned by Bumblebee per search request. One row per quote per search session.';

ALTER TABLE prod_trusted_bronze.internal.car_hire_quotes
  SET TBLPROPERTIES ( skyscanner.data_classification = "internal" );

ALTER TABLE prod_trusted_bronze.internal.car_hire_quotes
  SET TBLPROPERTIES ( skyscanner.data_category = "service_data" );

ALTER TABLE prod_trusted_bronze.internal.car_hire_quotes
  SET TBLPROPERTIES ( skyscanner.business_criticality = "p2" );

ALTER TABLE prod_trusted_bronze.internal.car_hire_quotes
  SET TBLPROPERTIES ( skyscanner.sox_scope = "false" );

ALTER TABLE prod_trusted_bronze.internal.car_hire_quotes
  SET TBLPROPERTIES ( skyscanner.ooh_support = "false" );

ALTER TABLE prod_trusted_bronze.internal.car_hire_quotes
  SET TBLPROPERTIES ( skyscanner.retention_period = "2 years" );
```

**6. Save SQL to Databricks and send to Ziggy**

```
Next steps:
1. Copy the SQL above into a new Databricks notebook:
   https://skyscanner-prod.cloud.databricks.com

2. Copy the notebook URL from your browser, then send this message to
   #data-platform-support:

   Hi Ziggy, please help update UC metadata for the following table, thanks.

   Table: prod_trusted_bronze.internal.car_hire_quotes
   Script: https://skyscanner-prod.cloud.databricks.com/editor/notebooks/YOUR_NOTEBOOK_ID

⚠️ Monte Carlo: This dataset is not monitored — onboard it to Monte Carlo
   separately after the metadata SQL is applied.

✅ Answers saved to registry — won't be asked again for this dataset.
```

**7. See remaining datasets for your squad**

```
Would you like to see the remaining datasets for autobot-squad that still need fixes? (yes / no)
> yes

Remaining datasets for autobot-squad — 24 still need fixes (showing worst 10):

 #  Dataset                                          Score   Failing rules
 1  car_hire_bumblebee_worker_dropped_quotes         27%     column_descriptions, business_criticality, sox_scope (+3)
 2  car_hire_office_search                           27%     column_descriptions, data_category, data_retention (+3)
 3  car_hire_vendor_mapping_search                   27%     column_descriptions, business_criticality, ooh_support (+3)
...

Run the skill again with any dataset name or Cortex URL to fix the next one.
```

---

## What the skill asks you

Only judgment calls that can't be inferred from the data. Questions are shown upfront — only the ones not already answered in the registry:

| Question | Example |
|---|---|
| Business criticality? | `p1` / `p2` / `p3` |
| SOX scope? | `yes` / `no` |
| OOH support required? | `yes` / `no` |
| Data classification? | `internal` / `confidential` / `restricted` / `public` |
| Data category? | `service_data` / `user_behaviour_data` / ... (12 options) |
| Data retention period? | `2 years` / `7 years` |
| Table description? | 1–2 sentences |
| Column descriptions? | `column_name: description` per line — asked separately |

Everything else (failing rules, storage location, column list, fix path) is loaded automatically from Cortex.

---

## Fix paths

| Path | When | Fix method |
|---|---|---|
| **Path A** | Trusted Data Pipeline / Fivetran / Databricks Notebooks | SQL `ALTER TABLE` → Databricks notebook → Ziggy ticket |
| **Path B** | SkySpark / HOPS pipelines | Edit conf files → `save_table_metadata` → PR |
| **Hive-to-UC ❓** | Legacy Hive migration tables | Skip — contact Data Platform |

Path is detected automatically. No user input needed.

---

## Registry

`registry.json` is pre-populated with all 324 PAC datasets — catalog tag, squad, failing rules, and column list from Cortex. When you answer questions, answers are saved locally so you don't have to answer again for the same dataset.

To get the latest registry (after new datasets are added to the scorecard), re-download `registry.json` from this repo and re-copy it.

---

## PAC tribe dataset inventory

Full list of datasets per squad with Cortex and Databricks links:
https://skyscanner.atlassian.net/wiki/spaces/~raymondliu/pages/2757361877

---

## Support

Questions → Raymond Liu or [#data-platform-support](https://skyscanner.slack.com/archives/C043JRRJJ)
