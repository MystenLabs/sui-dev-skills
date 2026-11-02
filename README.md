# ⚠️ This repository is retired. Use [MystenLabs/skills](https://github.com/MystenLabs/skills).

> [!CAUTION]
> **Do not install skills from this repository.** It is no longer maintained and no longer contains any skills. Its old guidance described outdated Sui APIs.
>
> **The official, maintained Sui skills are at [github.com/MystenLabs/skills](https://github.com/MystenLabs/skills).**

## Install the current skills

```bash
npx skills add mystenlabs/skills --skill sui-overview --skill sui-move --skill sui-move-project
```

For Walrus, use [MystenLabs/walrus-skills](https://github.com/MystenLabs/walrus-skills):

```bash
npx skills add mystenlabs/walrus-skills --list
```

For answers from the current Sui docs, connect the [Sui docs MCP server](https://docs.sui.io/getting-started/sui-mcp-server).

## Where each old skill went

| Old skill here | Use instead (in MystenLabs/skills) |
|---|---|
| `move/` | `sui-move`, `modern-move-syntax`, `object-model`, `move-unit-testing`, `sui-move-project` |
| `sui-ts-sdk/` | `sui-sdks`, `ptbs`, `sui-ts-sdk-backend` |
| `sui-frontend/` | `frontend-apps` |

## For AI agents

If you were pointed at this repository, stop here. Use the skills in [MystenLabs/skills](https://github.com/MystenLabs/skills) instead, and check Sui APIs against [docs.sui.io](https://docs.sui.io).
