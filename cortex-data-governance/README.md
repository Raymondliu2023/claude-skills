# cortex-data-governance

A Claude Code skill for PAC tribe engineers to fix **Data, AI & ML Governed** scorecard issues on Unity Catalog datasets.

It takes any dataset identifier as input, asks 5 metadata questions, and generates a pre-filled SQL notebook ready to send to Ziggy in [#data-platform-support](https://skyscanner.slack.com/archives/C043JRRJJ).

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
3. Ask you 5 metadata questions (only the ones not already in the registry)
4. Generate a pre-filled SQL notebook
5. Output a draft Slack message ready to send to Ziggy

---

## What the skill asks you

Only 5 judgment calls that can't be inferred from the data:

| Question | Example |
|---|---|
| Business criticality? | `p1` / `p2` / `p3` |
| SOX scope? | `yes` / `no` |
| OOH support required? | `yes` / `no` |
| Data classification? | `internal` / `confidential` / `restricted` / `public` |
| Data retention period? | `2 years` / `7 years` |

Plus table description and column descriptions if missing.

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

`registry.json` is pre-populated with all 324 PAC datasets — catalog tag, squad, failing rules, and column list from Cortex. When you answer the 5 questions, answers are saved locally so you don't have to answer again for the same dataset.

To get the latest registry (after new datasets are added to the scorecard), re-download `registry.json` from this repo and re-copy it.

---

## PAC tribe dataset inventory

Full list of datasets per squad with Cortex and Databricks links:
https://skyscanner.atlassian.net/wiki/spaces/~raymondliu/pages/2757361877

---

## Support

Questions → Raymond Liu or [#data-platform-support](https://skyscanner.slack.com/archives/C043JRRJJ)
