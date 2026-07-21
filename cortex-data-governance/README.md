# cortex-data-governance

A Claude Code skill for PAC tribe engineers to fix **Data, AI & ML Governed** scorecard issues on Unity Catalog datasets.

It resolves any dataset identifier, shows all pending questions upfront, supports batch answering, **auto-creates a Databricks notebook via API**, and produces a ready-to-send Ziggy message.

---

## Preconditions

Before you can run this skill you need all of the following:

### 1. Claude Code

Install from https://claude.ai/code or via the VS Code / JetBrains extension.

Verify: `claude --version`

### 2. Cortex personal access token

1. Go to https://app.getcortexapp.com → top-right avatar → **Settings** → **API Keys**
2. Click **Create API Key** — give it a name like `claude-code-governance`
3. Copy the token

### 3. Cortex MCP configured in Claude Code

Add the following to your Claude Code global MCP config at `~/.claude.json`:

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

Restart Claude Code after saving.

### 4. Skill files installed

```bash
mkdir -p ~/.claude/skills/cortex-data-governance
cp SKILL.md ~/.claude/skills/cortex-data-governance/SKILL.md
cp registry.json ~/.claude/skills/cortex-data-governance/registry.json
```

### 5. Databricks token (optional — enables auto-notebook creation)

Add to your shell profile (`~/.zshrc` or `~/.bashrc`):

```bash
export DATABRICKS_TOKEN=your_personal_access_token
```

Get your token: [skyscanner-prod.cloud.databricks.com](https://skyscanner-prod.cloud.databricks.com) → top-right avatar → **Settings** → **Developer** → **Access tokens** → **Generate new token**.

Without this token the skill still works — it shows the SQL and asks you to paste the notebook URL manually.

---

## How to use

Open Claude Code in any directory and run:

```
/cortex-data-governance
```

Then paste any of the following when prompted:

- **Cortex URL** — e.g. `https://app.getcortexapp.com/admin/scorecards/9015?...&entity=en3d1f29a99f493f6a`
- **Databricks URL** — e.g. `https://skyscanner-prod.cloud.databricks.com/explore/data/prod_trusted_bronze/internal/car_hire_quotes`
- **Dataset name** — e.g. `car_hire_quotes`
- **Cortex tag** — e.g. `prod_trusted_bronze.internal.car_hire_quotes`

The skill will:
1. Resolve the dataset and show current score + failing rules
2. Auto-detect the fix path (Path A / Path B / Hive-to-UC)
3. Show all pending questions upfront — prepare answers before it asks
4. Accept answers all at once or one by one
5. **Auto-create a Databricks notebook** via the workspace API (if token is set)
6. Output a ready-to-send Ziggy Slack message with the notebook URL pre-filled
7. Offer to show remaining datasets for your squad

---

## Full walkthrough

**1. Run the skill**

```
> /cortex-data-governance

🔍 Checking preconditions...
✅ Cortex MCP — connected
✅ Registry — loaded (324 datasets)
✅ Databricks token — found (auto-notebook creation enabled)
✅ All preconditions met — ready to fix.
```

**2. Paste a dataset**

```
Paste a Cortex URL, Databricks URL, dataset name, or Cortex tag:
> car_hire_quotes
```

**3. Skill shows current state**

```
Dataset: prod_trusted_bronze.internal.car_hire_quotes
Squad: autobot-squad
Score: 27% (4/15 rules passing)
Path: A (Trusted Data Pipeline)

Failing rules: column_descriptions, data_classification, data_category,
               business_criticality, sox_scope, ooh_support_required,
               data_retention, monte_carlo_monitored
```

**4. Pending questions shown upfront**

```
Here are the questions I'll ask (7 in total). Take a moment to prepare.

1. Business criticality? → p1 (critical) / p2 (important) / p3 (low)
2. SOX scope? → yes / no
3. OOH support required? → yes / no
4. Data classification? → internal / confidential / restricted / public
5. Data category? → service_data | user_behaviour_data | ... (12 options)
6. Data retention period? → e.g. "2 years", "7 years", "indefinite"
7. Table description? → 1-2 sentences

You can answer all at once:  1.p2; 2.no; 3.no; 4.internal; 5.service_data; 6.2 years; 7.Car hire quotes from Bumblebee.
Or type "ready" to be asked one by one.
```

**5. Answer all at once**

```
> 1.p2; 2.no; 3.no; 4.internal; 5.service_data; 6.2 years; 7.Car hire quotes returned by Bumblebee per search request.

Confirm? (yes / edit N)
> yes
```

**6. Skill auto-creates the Databricks notebook**

```
Creating Databricks notebook...
✅ Notebook created: https://skyscanner-prod.cloud.databricks.com/editor/notebooks/1234567890?o=1849662692269217
```

**7. Ready-to-send Ziggy message**

```
Copy and send to #data-platform-support:

  Hi Ziggy, please help update UC metadata for the following table, thanks.

  Table: prod_trusted_bronze.internal.car_hire_quotes
  Script: https://skyscanner-prod.cloud.databricks.com/editor/notebooks/1234567890?o=1849662692269217

⚠️ Monte Carlo: This dataset is not monitored — onboard separately after metadata is applied.
✅ Answers saved to registry.
```

**8. See remaining datasets**

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

## Fix paths

| Path | When | Fix method |
|---|---|---|
| **Path A** | Trusted Data Pipeline / Fivetran / Databricks Notebooks | Auto-create notebook → Ziggy ticket |
| **Path B** | SkySpark / HOPS pipelines | Edit conf files → `save_table_metadata` → PR |
| **Hive-to-UC ❓** | Legacy Hive migration tables | Skip — contact Data Platform |

---

## Registry

`registry.json` is pre-populated with all 324 PAC datasets. Answers are saved locally so you don't repeat them for the same dataset. Re-download to get the latest.

---

## PAC tribe dataset inventory

https://skyscanner.atlassian.net/wiki/spaces/~raymondliu/pages/2757361877

---

## Support

Questions → Raymond Liu or [#data-platform-support](https://skyscanner.slack.com/archives/C043JRRJJ)
