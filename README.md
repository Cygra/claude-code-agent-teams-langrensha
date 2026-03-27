# claude-code-agent-teams-langrensha

利用 Claude Code 的 Agent Teams 的能力让 teammates 玩狼人杀。
Let teammates in Claude Code Agent teams play Werewolf (狼人杀) with each other.

## Quick Start

```
git clone https://github.com/Cygra/claude-code-agent-teams-langrensha ~/.claude/skills/langrensha
```

Trigger it with something like "来一局狼人杀", "玩狼人杀", or "play Werewolf".

## What is 狼人杀 (Werewolf)?

https://en.wikipedia.org/wiki/Mafia_(party_game)

A social deduction game where players are secretly assigned roles — wolves try to eliminate villagers, while villagers try to identify and vote out the wolves.

## Game Setup

7 AI agents play the game, moderated by you (team-lead) as the Sheriff (警长):

| Role | Team | Count |
|------|------|-------|
| 🐺 狼人 Werewolf | Wolves | 2 |
| 🔮 预言家 Seer | Villagers | 1 |
| 🧪 女巫 Witch | Villagers | 1 |
| 🏹 猎人 Hunter | Villagers | 1 |
| 👤 平民 Villager | Villagers | 2 |

## What is Claude Code agent teams?

https://code.claude.com/docs/en/agent-teams

---

*Related: [Connect Four](https://github.com/Cygra/claude-code-agent-teams-connect-four) · [Tic-Tac-Toe](https://github.com/Cygra/claude-code-agent-teams-tic-tac-toe)*
