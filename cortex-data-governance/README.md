# cortex-data-governance

A Claude Code skill for PAC tribe engineers to fix **Data, AI & ML Governed** scorecard issues on Unity Catalog datasets.

It resolves any dataset identifier, shows all pending questions upfront, supports batch answering, always outputs the full SQL, **auto-creates a Databricks notebook**, **creates or links a Jira ticket**, and produces a ready-to-send Ziggy message.

---

## Preconditions

### 1. Claude Code

Install from https://claude.ai/code or via the VS Code / JetBrains extension.

### 2. Cortex personal access token

1. Go to https://app.getcortexapp.com → Settings → **API Keys**
2. Create a token — name it `claude-code-governance`
3. Add to `~/.claude.json`:

```json
{
  "mcpServers": {
    "cortex-remote": {
      "type": "remote",
      "url": "https://api.getcortexapp.com/mcp",
      "headers": { "Authorization": "Bearer YOUR_TOKEN_HERE" }
    }
  }
}
```

Restart Claude Code after saving.

### 3. Skill files installed

```bash
mkdir -p ~/.claude/skills/cortex-data-governance
cp SKILL.md ~/.claude/skills/cortex-data-governance/SKILL.md
cp registry.json ~/.claude/skills/cortex-data-governance/registry.json
```

### 4. Databricks token (optional — enables auto-notebook creation)

```bash
export DATABRICKS_TOKEN=your_personal_access_token
```

Get from: [skyscanner-prod.cloud.databricks.com](https://skyscanner-prod.cloud.databricks.com) → Settings → Developer → Access tokens.

Without this the skill still works — SQL is shown for manual copy.

---

## How to use

```
/cortex-data-governance
```

Accepts: dataset name, Cortex tag, Cortex URL, or Databricks URL.

The skill will:
1. Resolve the dataset and show current score + failing rules
2. Auto-detect fix path (Path A / Path B / Hive-to-UC)
3. Show all pending questions upfront — prepare answers before it asks
4. Accept answers all at once or one by one
5. Always output the full SQL block
6. Auto-create a Databricks notebook (if token is set)
7. Draft Ziggy Slack message with notebook URL
8. Create a new Jira ticket or add a comment to an existing one
9. Offer to show remaining datasets for your squad

---

## Full walkthrough

**1. Start**

```
> /cortex-data-governance

🔍 Checking preconditions...
✅ Cortex MCP — connected
✅ Registry — loaded (324 datasets)
✅ Databricks token — found
✅ All preconditions met — ready to fix.

Paste a dataset name, Cortex tag, Cortex URL, or Databricks URL:
> car_hire_quotes
```

**2. Dataset resolved**

```
Dataset: prod_trusted_bronze.internal.car_hire_quotes
Squad: autobot-squad | Score: 27% | Path: A (Trusted Data Pipeline)

Failing: column_descriptions, data_classification, data_category,
         business_criticality, sox_scope, ooh_support_required, data_retention
```

**3. Questions shown upfront**

```
Here are the questions I'll ask (7 in total). Take a moment to prepare.

1. Business criticality? → p1 (critical) / p2 (important) / p3 (low)
2. SOX scope? → yes / no
3. OOH support required? → yes / no
4. Data classification? → internal / confidential / restricted / public
5. Data category? → service_data | user_behaviour_data | ... (12 options)
6. Data retention period? → e.g. "2 years", "7 years", "indefinite"
7. Table description? → 1-2 sentences

Answer all at once:  1.p2; 2.no; 3.no; 4.internal; 5.service_data; 6.2 years; 7.Car hire quotes from Bumblebee.
Or type "ready" for one by one.
```

**4. Batch answer**

```
> 1.p2; 2.no; 3.no; 4.internal; 5.service_data; 6.2 years; 7.Car hire quotes returned by Bumblebee per search request.

Confirm? (yes / edit N)
> yes
```

**5. SQL output (always shown)**

```sql
-- Dataset: prod_trusted_bronze.internal.car_hire_quotes
-- Squad: autobot-squad | Score: 27% | Generated: 2026-07-21

COMMENT ON TABLE prod_trusted_bronze.internal.car_hire_quotes IS
'Car hire quotes returned by Bumblebee per search request.';

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

**6. Notebook created**

```
Creating Databricks notebook...
✅ Notebook created: https://skyscanner-prod.cloud.databricks.com/editor/notebooks/1234567890?o=1849662692269217
```

**7. Ziggy message**

```
Copy and send to #data-platform-support:

  Hi Ziggy, please help update UC metadata for the following table, thanks.

  Table: prod_trusted_bronze.internal.car_hire_quotes
  Script: https://skyscanner-prod.cloud.databricks.com/editor/notebooks/1234567890?o=1849662692269217
```

**8. Jira ticket**

```
Do you have a Jira ticket for this dataset fix?
Paste the ticket key (e.g. ATBT-12345), or type "create" to create one, or "skip".
> create

✅ Jira ticket created: ATBT-12050 — https://skyscanner.atlassian.net/browse/ATBT-12050
   [Data Governance] Fix UC metadata — car_hire_quotes
```

**9. Remaining datasets**

```
Would you like to see the remaining datasets for autobot-squad? (yes / no)
> yes

Remaining: 24 datasets still need fixes (showing worst 10):
 1  car_hire_bumblebee_worker_dropped_quotes   27%   column_descriptions, business_criticality (+3)
 2  car_hire_office_search                     27%   column_descriptions, data_category (+3)
...
```

---

## What the skill asks you

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

---

## Jira options

| Input | What happens |
|---|---|
| Paste existing key (e.g. `ATBT-12345`) | Adds a comment with notebook URL + rules addressed |
| `create` | Creates a new Task with dataset details, notebook URL, and next-steps checklist |
| `skip` | Continues without touching Jira |

---

## Fix paths

| Path | When | Fix method |
|---|---|---|
| **Path A** | Trusted Data Pipeline / Fivetran / Databricks Notebooks | Auto-create notebook → Ziggy ticket |
| **Path B** | SkySpark / HOPS pipelines | Edit conf files → `save_table_metadata` → PR |
| **Hive-to-UC ❓** | Legacy Hive migration tables | Skip — contact Data Platform |

---

## Registry

`registry.json` is pre-populated with all 324 PAC datasets. Answers are saved locally — never asked again for the same dataset. Re-download to get the latest.

---

## PAC tribe dataset inventory

https://skyscanner.atlassian.net/wiki/spaces/~raymondliu/pages/2757361877

---

## Support

Questions → Raymond Liu or [#data-platform-support](https://skyscanner.slack.com/archives/C043JRRJJ)
