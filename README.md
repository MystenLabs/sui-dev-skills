# Sui Dev Skills

A collection of Claude skills for Sui development. Each skill is modular and can be used independently or composed together for full-stack Sui projects.

## Skills

| Skill | Description | Status |
|-------|-------------|--------|
| [`sui-move/`](./sui-move/) | Idiomatic Sui Move 2024 — object model, modern syntax, testing | ✅ Ready |
| `sdk/` | TypeScript SDK — transaction building, signing, querying | 🔜 Coming soon |
| `frontend/` | Frontend integration — wallet adapters, dApp kit, React patterns | 🔜 Coming soon |

## Installation

### Global install (recommended — works across all your projects)

```bash
git clone https://github.com/MystenLabs/sui-dev-skills ~/.claude/skills/sui-dev-skills
```

Claude Code auto-discovers skills in `~/.claude/skills/` and activates them automatically based on context.

### Project-local install (commit to your repo)

```bash
git clone https://github.com/MystenLabs/sui-dev-skills .claude/skills/sui-dev-skills
```

Commit it to share the skill with your whole team — anyone who opens the project in Claude Code gets it automatically.

### Composing skills

For full-stack Sui dapps, install all skills and Claude will activate whichever are relevant. You can also pin specific skills in your `CLAUDE.md`:

```markdown
# My Sui Dapp

@~/sui-dev-skills/sui-move/SKILL.md
@~/sui-dev-skills/sdk/SKILL.md
@~/sui-dev-skills/frontend/SKILL.md
```

Claude reads all referenced skills before starting work, so it will apply conventions from all of them.

## Running evals

Requires [Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview).

Each skill has its own `evals/evals.json`. To evaluate a skill, run Claude Code from this repo's parent directory:

```bash
cd ~/  # parent of sui-dev-skills/
claude
```

Then tell Claude:
> "Use the skill-creator to run evals on the skill at `./sui-dev-skills/sui-move/`"

Eval workspace outputs are written to a sibling directory (e.g. `sui-move-workspace/`) and are gitignored.

## Contributing

PRs welcome for:
- New skills (SDK, frontend, indexer, zkLogin, etc.)
- Improvements to existing skills
- Additional evals and benchmark prompts
- Verified real-world examples