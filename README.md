# claude-skills

Claude Code skills for PAC tribe engineers at Skyscanner — shareable, ready to use.

Each skill is a standalone folder. Clone the repo, copy the skill files to `~/.claude/skills/`, and run it from Claude Code.

---

## Skills

| Skill | What it does |
|---|---|
| [cortex-data-governance](./cortex-data-governance/) | Fix Unity Catalog dataset metadata issues on the Cortex Data, AI & ML Governed scorecard — generates pre-filled SQL + draft Ziggy ticket |

---

## Quick install

```bash
# Clone
git clone https://github.com/Raymondliu2023/claude-skills.git
cd claude-skills

# Install a skill
mkdir -p ~/.claude/skills/cortex-data-governance
cp cortex-data-governance/SKILL.md ~/.claude/skills/cortex-data-governance/
cp cortex-data-governance/registry.json ~/.claude/skills/cortex-data-governance/
```

Then open Claude Code and run `/cortex-data-governance`.

See each skill's `README.md` for full setup instructions and preconditions.

---

## Adding more skills

Each skill gets its own folder:
```
claude-skills/
├── README.md
├── cortex-data-governance/
│   ├── README.md
│   ├── SKILL.md
│   └── registry.json
└── next-skill/
    ├── README.md
    └── SKILL.md
```

---

*Maintained by Raymond Liu — questions via Slack or raise a GitHub issue.*
