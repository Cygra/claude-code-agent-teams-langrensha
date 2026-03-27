# claude-code-agent-teams-langrensha

利用 Claude Code 的 Agent Teams 能力，让 AI teammates 玩一局完整的狼人杀。

<img width="2456" height="1262" alt="image" src="https://github.com/user-attachments/assets/d8d3fd73-6881-445d-9e97-7a483dcd2dcc" />

## 快速开始

```
git clone https://github.com/Cygra/claude-code-agent-teams-langrensha ~/.claude/skills/langrensha
```

用以下任意语句触发技能：「来一局狼人杀」、「我要玩狼人杀」、「play Werewolf」。

## 游戏规则

https://zh.wikipedia.org/wiki/%E7%8B%BC%E4%BA%BA%E6%9D%80

狼人杀是一款社交推理游戏。玩家被秘密分配阵营——狼人想要淘汰所有好人，而好人需要找出并投票驱逐所有狼人。

## 游戏配置

本技能采用经典 **12人局**标准配置，由你（team-lead）扮演**警长**主持：

| 角色 | 阵营 | 人数 | 技能 |
|------|------|------|------|
| 🐺 狼人 | 狼人 | 4 | 每晚击杀一名玩家 |
| 🔮 预言家 | 好人（神职） | 1 | 每晚查验一名玩家的身份 |
| 🧪 女巫 | 好人（神职） | 1 | 解药救人 + 毒药杀人，各一次 |
| 🏹 猎人 | 好人（神职） | 1 | 被淘汰时可开枪带走一人 |
| 🛡️ 守卫 | 好人（神职） | 1 | 每晚守护一名玩家免受击杀 |
| 👤 平民 | 好人 | 4 | 无特殊技能，靠推理和投票 |

玩家代号为 player-1 ~ player-12，代号本身不透露角色，保证推理的公平性。

## 关于 Claude Code Agent Teams

https://code.claude.com/docs/en/agent-teams

---

*相关项目：[四子棋](https://github.com/Cygra/claude-code-agent-teams-connect-four) · [井字棋](https://github.com/Cygra/claude-code-agent-teams-tic-tac-toe)*
