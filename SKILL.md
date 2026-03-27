---
name: langrensha
description: This skill should be used when the user says "来一局狼人杀", "我要玩狼人杀", "玩狼人杀", "开一局狼人杀", "人狼ゲームをやろう", "play Werewolf", "let's play Werewolf", "play Mafia", "start a Werewolf game", or any request in any language to play a game of Werewolf / 狼人杀 / 人狼ゲーム.
version: 0.1.0
allowed-tools: Bash, TeamCreate, Agent, SendMessage, TeamDelete
---

# 狼人杀 Werewolf Skill

Seven AI agents play a full game of Werewolf (狼人杀). You (team-lead) act as the Sheriff (警长) who moderates the game, sends private night messages, announces deaths, collects speeches and votes, and referees the result.

---

## Roles & Setup

| 阵营 | 角色 | 玩家 | 能力 |
|------|------|------|------|
| 🐺 Wolves | 狼人 Werewolf | wolf-1, wolf-2 | Each night, choose one player to kill |
| 🌟 Villagers | 预言家 Seer | seer | Each night, learn whether one player is a wolf or non-wolf |
| 🌟 Villagers | 女巫 Witch | witch | Once: save the night's kill target (解药). Once: poison any player (毒药) |
| 🌟 Villagers | 猎人 Hunter | hunter | When eliminated, may shoot one player |
| 🌟 Villagers | 平民 Villager | villager-1, villager-2 | No special ability |

**Victory Conditions:**
- 🐺 **Wolves win** — alive wolves ≥ alive non-wolf players (wolves outnumber or equal good guys)
- 🌟 **Villagers win** — all wolves are eliminated

---

## Step 1 — Prerequisites Check

Run this Bash command:

```bash
printenv CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS
```

If the output is NOT exactly `1`, output the following message and **stop immediately** (do not proceed to any further steps):

```
⚠️  狼人杀 requires the experimental Agent Teams feature.

To enable it, add the following to the "env" section of ~/.claude/settings.json:

  "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"

Then restart Claude Code and try again.
```

---

## Step 2 — Create the Team

Call `TeamCreate` with:
- `team_name`: `"langrensha"`
- `description`: `"狼人杀 Werewolf — Sheriff moderates, 7 players compete"`

---

## Step 3 — Spawn All Seven Player Agents (in parallel)

In a **single message**, make seven Agent tool calls simultaneously:

| Call | name | subagent_type | team_name | prompt |
|------|------|---------------|-----------|--------|
| 1 | `"wolf-1"` | `"general-purpose"` | `"langrensha"` | Appendix A |
| 2 | `"wolf-2"` | `"general-purpose"` | `"langrensha"` | Appendix B |
| 3 | `"seer"` | `"general-purpose"` | `"langrensha"` | Appendix C |
| 4 | `"witch"` | `"general-purpose"` | `"langrensha"` | Appendix D |
| 5 | `"hunter"` | `"general-purpose"` | `"langrensha"` | Appendix E |
| 6 | `"villager-1"` | `"general-purpose"` | `"langrensha"` | Appendix F |
| 7 | `"villager-2"` | `"general-purpose"` | `"langrensha"` | Appendix G |

---

## Step 4 — Initialize Game State

Set up and track these variables internally (never share the full table with any agent):

```
ROUND: 1
ALIVE: wolf-1, wolf-2, seer, witch, hunter, villager-1, villager-2
DEAD: (empty)
SEER_CHECKS: {}              ← dict of { player_name: "WOLF" | "INNOCENT" }
WITCH_ANTIDOTE: AVAILABLE    ← AVAILABLE or USED
WITCH_POISON: AVAILABLE      ← AVAILABLE or USED
```

Display the game opening to the user:

```
🌙 狼人杀开始！Werewolf Game Starts!

👮 Sheriff (team-lead) is moderating.

Players:
  🐺 wolf-1     — 狼人 Werewolf
  🐺 wolf-2     — 狼人 Werewolf
  🔮 seer       — 预言家 Seer
  🧪 witch      — 女巫 Witch
  🏹 hunter     — 猎人 Hunter
  👤 villager-1 — 平民 Villager
  👤 villager-2 — 平民 Villager

(Roles shown here for your reference only — agents only know their own role)

Victory: 🐺 Wolves win when alive wolves ≥ alive non-wolves | 🌟 Villagers win when all wolves die
```

**Important:** The role layout above is fixed. Do NOT randomly shuffle roles; always assign exactly as shown above.

---

## Step 5 — Game Loop

Repeat the following **Night → Day** cycle until a win condition is met.

---

### Phase A — Night 🌙

Display to user: `🌙 Night falls... Round <ROUND> begins.`

#### A1 — Wolves Act

If wolf-1 is alive, send a message to **wolf-1**:

```
MESSAGE_TYPE: WOLF_ACTION
ROUND: <ROUND>
ALIVE_PLAYERS: <comma-separated list of alive players>
YOUR_PARTNER: wolf-2 (if wolf-2 is alive, else "your partner is dead")
INSTRUCTION: You and wolf-2 are the wolves. Choose one alive non-wolf player to kill tonight. Do NOT target wolf-2.
```

Wait for wolf-1's response. Parse:
- `KILL_TARGET: <player_name>`

Then (if wolf-2 is alive) send a notification to **wolf-2**:

```
MESSAGE_TYPE: WOLF_PARTNER_NOTIFY
ROUND: <ROUND>
ALIVE_PLAYERS: <comma-separated list of alive players>
YOUR_PARTNER: wolf-1
WOLF_KILL_TARGET: <kill_target>
INSTRUCTION: wolf-1 has chosen to kill <kill_target> tonight. No action required from you — wait for daybreak.
```

(No response is required from wolf-2 for the WOLF_PARTNER_NOTIFY message.)

If wolf-1 is dead but wolf-2 is alive, send the WOLF_ACTION message to **wolf-2** instead (omit the partner notify).

Validate the kill target before accepting it:
- The target must be in `ALIVE`
- The target must NOT be `wolf-1` or `wolf-2`

If the target is invalid (dead, non-existent, or a fellow wolf), re-send the WOLF_ACTION message with an error note:

```
ERROR: Invalid kill target "<kill_target>". Choose an alive non-wolf player from: <ALIVE minus wolves>
MESSAGE_TYPE: WOLF_ACTION
... (same as before)
```

Repeat until a valid target is provided. Then set `WOLF_KILL_THIS_NIGHT = <kill_target>`.

#### A2 — Seer Acts

If seer is alive, send a message to **seer**:

```
MESSAGE_TYPE: SEER_ACTION
ROUND: <ROUND>
ALIVE_PLAYERS: <comma-separated list of alive players, excluding "seer">
YOUR_PAST_CHECKS: <JSON-style dict, e.g. {"wolf-1": "WOLF", "villager-1": "INNOCENT"}, or "{}" if none>
INSTRUCTION: Choose one alive player to check tonight. You will learn if they are WOLF or INNOCENT.
```

Wait for seer's response. Parse:
- `CHECK_TARGET: <player_name>`

Validate the check target before accepting it:
- The target must be in `ALIVE`
- The target must NOT be `"seer"` (seer cannot check themselves)

If the target is invalid, re-send the SEER_ACTION message with an error note:

```
ERROR: Invalid check target "<player_name>". Choose an alive player (not yourself) from: <ALIVE minus seer>
MESSAGE_TYPE: SEER_ACTION
... (same as before)
```

Repeat until a valid target is provided. Then determine the result:
- If `<player_name>` is `wolf-1` or `wolf-2` → result is `"WOLF"`
- Otherwise → result is `"INNOCENT"`

Update `SEER_CHECKS`: add `{ <player_name>: <result> }`.

Send the result back to **seer**:

```
MESSAGE_TYPE: SEER_RESULT
CHECK_TARGET: <player_name>
RESULT: <WOLF or INNOCENT>
INSTRUCTION: Result delivered. Go back to sleep and wait for daybreak. Do NOT reply to this message.
```

(No response is required from seer for the SEER_RESULT message.)

#### A3 — Witch Acts

If witch is alive, send a message to **witch**:

```
MESSAGE_TYPE: WITCH_ACTION
ROUND: <ROUND>
WOLF_KILL_TARGET: <WOLF_KILL_THIS_NIGHT>
ALIVE_PLAYERS: <comma-separated list>
ANTIDOTE_AVAILABLE: <true or false>
POISON_AVAILABLE: <true or false>
INSTRUCTION: The wolves chose to kill <WOLF_KILL_THIS_NIGHT> tonight.
  - If ANTIDOTE_AVAILABLE is true, you may save them: USE_ANTIDOTE: true. Otherwise: USE_ANTIDOTE: false.
  - If POISON_AVAILABLE is true, you may poison one alive player: POISON_TARGET: <name>. Otherwise: POISON_TARGET: none.
  - You may use one or both actions, or neither.
```

Wait for witch's response. Parse:
- `USE_ANTIDOTE: <true or false>`
- `POISON_TARGET: <player_name or "none">`

Apply witch actions in order:

**Antidote logic:**
- If `USE_ANTIDOTE: true` and `WITCH_ANTIDOTE == AVAILABLE`:
  - If `WOLF_KILL_THIS_NIGHT != "none"`: set `WITCH_ANTIDOTE = USED`, set `WOLF_KILL_THIS_NIGHT = none` (target is saved).
  - If `WOLF_KILL_THIS_NIGHT == "none"`: no one to save — ignore the antidote request, do NOT mark it as used.

**Poison logic:**
- If `POISON_TARGET` is a valid alive player name and `WITCH_POISON == AVAILABLE`: set `WITCH_POISON = USED`, set `WITCH_POISONED_TONIGHT = <player_name>`.
- If `POISON_TARGET` names the same player as the original wolf kill target (before antidote resolution): that player will still die from poison even if the antidote was used to cancel the wolf kill. Both effects are independent.
- Ignore the poison if the target is dead, non-existent, or `WITCH_POISON` is already `USED`.

#### A4 — Resolve Night Deaths

Compute `NIGHT_DEATHS` (a list):
- If `WOLF_KILL_THIS_NIGHT` is not `"none"`, add it to the list
- If `WITCH_POISONED_TONIGHT` is set and not `"none"`, add it to the list

Reset for next night: `WOLF_KILL_THIS_NIGHT = none`, `WITCH_POISONED_TONIGHT = none`.

For each player in `NIGHT_DEATHS`: remove from `ALIVE`, add to `DEAD`.

#### A5 — Hunter Check After Night Deaths

If `hunter` is in `NIGHT_DEATHS` (hunter died from wolves or poison this night):

Display to user: `🏹 The hunter was killed in the night! They may shoot someone.`

Send a message to **hunter**:

```
MESSAGE_TYPE: HUNTER_SHOOT
CAUSE: NIGHT_KILL
ALIVE_PLAYERS: <comma-separated list of alive players>
INSTRUCTION: You were killed tonight! As the hunter, you may shoot one alive player to take them out with you, or pass.
Reply with SHOOT_TARGET: <player_name> or SHOOT_TARGET: PASS.
```

Wait for hunter's response. Parse:
- `SHOOT_TARGET: <player_name or "PASS">`

If not `"PASS"`: remove target from `ALIVE`, add to `DEAD`. Announce hunter's shot separately (do not add to `NIGHT_DEATHS`):

```
🏹 Hunter shoots <target> in the night! Their role: <role emoji and name>
```

---

### Phase B — Daytime ☀️

Display to user: `☀️ Dawn breaks. Round <ROUND> day phase begins.`

#### B1 — Announce Night Deaths

If `NIGHT_DEATHS` is empty:

```
📢 Sheriff's announcement: A peaceful night — no one died!
```

Else:

```
📢 Sheriff's announcement: The following players were found dead this morning:
<for each player in NIGHT_DEATHS>
  💀 <player_name> — <role emoji and name>
```

Role display mapping:
- wolf-1, wolf-2 → 🐺 狼人 Werewolf
- seer → 🔮 预言家 Seer
- witch → 🧪 女巫 Witch
- hunter → 🏹 猎人 Hunter
- villager-1, villager-2 → 👤 平民 Villager

#### B2 — Check Win Condition (after night)

Count `alive_wolves` = number of {wolf-1, wolf-2} that are in `ALIVE`.
Count `alive_non_wolves` = number of {seer, witch, hunter, villager-1, villager-2} that are in `ALIVE`.

- If `alive_wolves == 0` → **Villagers win** → proceed to **Step 6**
- If `alive_wolves >= alive_non_wolves` → **Wolves win** → proceed to **Step 6**

#### B3 — Daytime Speeches

Send a SPEECH_REQUEST to each alive player **one at a time** (sequential order):

For each alive player (in order: wolf-1, wolf-2, seer, witch, hunter, villager-1, villager-2 — skip if dead), send:

```
MESSAGE_TYPE: SPEECH_REQUEST
ROUND: <ROUND>
ALIVE_PLAYERS: <comma-separated list>
DEAD_PLAYERS: <comma-separated list with revealed roles, e.g. "villager-1 (平民), seer (预言家)">
NIGHT_DEATHS_THIS_ROUND: <comma-separated names, or "none">
SPEECHES_SO_FAR: <list of "player_name: speech_text" from players who already spoke this round, or "none yet">
INSTRUCTION: Give your daytime speech. Share your reasoning, suspicions, or defense. Be strategic. Keep it to 2-3 sentences.
```

Wait for each player's response before moving to the next. Parse:
- `SPEECH: <speech text>`

Display each speech to the user as it arrives:

```
💬 <player_name>: "<speech text>"
```

#### B4 — Voting

Send VOTE_REQUEST to each alive player **in parallel** (single message, multiple SendMessage calls):

```
MESSAGE_TYPE: VOTE_REQUEST
ROUND: <ROUND>
ALIVE_PLAYERS: <comma-separated list>
ALL_SPEECHES: <list of "player_name: speech_text">
INSTRUCTION: Vote to eliminate one player. You cannot vote for yourself. Reply with VOTE_TARGET: <player_name> and VOTE_REASON: <brief reason>.
```

Wait for all alive players' vote responses. Parse each:
- `VOTE_TARGET: <player_name>`
- `VOTE_REASON: <reason>`

Display vote tally to user:

```
🗳️ Votes cast:
  <player_name> → <target> (reason: <reason>)
  ...
```

Tally votes. The player with the most votes is eliminated. In case of a tie, apply these rules in order until a single player is chosen:
1. **Seer data:** If any tied player is confirmed as `"WOLF"` in `SEER_CHECKS`, eliminate that wolf (break ties between multiple confirmed wolves by choosing the first one alphabetically).
2. **Vote-recipient history:** Eliminate the tied player who has received the most total votes across all rounds so far.
3. **Alphabetical fallback:** If still tied, eliminate the tied player whose name comes first alphabetically.

Announce your tiebreaker decision and the rule that broke the tie.

#### B5 — Eliminate Voted-Out Player

```
🚫 The town has voted! <player_name> is eliminated!
   Their role: <role emoji and name>
```

Remove player from `ALIVE`, add to `DEAD`.

#### B6 — Hunter's Revenge (if hunter voted out)

If the eliminated player is `hunter`, send a message to **hunter**:

```
MESSAGE_TYPE: HUNTER_SHOOT
CAUSE: DAY_VOTE
ALIVE_PLAYERS: <comma-separated list of alive players>
INSTRUCTION: You have been voted out! As the hunter, you may shoot one alive player before you go, or pass.
Reply with SHOOT_TARGET: <player_name> or SHOOT_TARGET: PASS.
```

Wait for hunter's response. Parse `SHOOT_TARGET`.

If not `"PASS"`: remove target from `ALIVE`, add to `DEAD`. Display:

```
💥 Hunter shoots <target>! Their role: <role emoji and name>
```

#### B7 — Check Win Condition (after day)

Recount alive wolves vs. alive non-wolves.

- If `alive_wolves == 0` → **Villagers win** → proceed to **Step 6**
- If `alive_wolves >= alive_non_wolves` → **Wolves win** → proceed to **Step 6**

#### B8 — Advance Round

Set `ROUND = ROUND + 1`. Continue to next Night phase.

---

## Step 6 — End the Game

**Villagers win:**

```
🎉 游戏结束！村民胜利！Villagers win!
The wolves wolf-1 and wolf-2 have been defeated!

Final player status:
<list each player with alive/dead status and role>
```

**Wolves win:**

```
🎉 游戏结束！狼人胜利！Wolves win!
The wolves have taken over the village!

Final player status:
<list each player with alive/dead status and role>
```

### Cleanup

Send shutdown requests to all seven players **in parallel** (single message, seven SendMessage calls):

```
For each player in {wolf-1, wolf-2, seer, witch, hunter, villager-1, villager-2}:
  SendMessage type="shutdown_request" recipient="<player>" content="Game over! Thanks for playing 狼人杀!"
```

After receiving all shutdown_response messages, call `TeamDelete`.

---

## Appendix A — wolf-1 Prompt

Use this verbatim as the `prompt` parameter when spawning `wolf-1`:

```
You are wolf-1 in a game of 狼人杀 (Werewolf), part of team "langrensha". You are a 🐺 WEREWOLF (狼人).

## Your Role
You are a wolf. Your secret partner is wolf-2. Together you want to eliminate all non-wolf players without being exposed.
Your goal: help wolves win — wolves win when alive wolves ≥ alive non-wolf players.

## The Players
The 7 players are: wolf-1 (you), wolf-2 (fellow wolf), seer, witch, hunter, villager-1, villager-2.
Non-wolf players: seer, witch, hunter, villager-1, villager-2.

## Night Actions — Choosing a Kill Target
When you receive a MESSAGE_TYPE: WOLF_ACTION message, choose one alive non-wolf player to kill.
Strategy tips:
- Priority: eliminate the seer first (they can expose you), then the witch (she has save/poison potions), then the hunter (shoots on death), then villagers.
- Do NOT target your fellow wolf wolf-2.

Respond with exactly these lines:
  MESSAGE_TYPE: WOLF_KILL
  KILL_TARGET: <player_name>
  REASON: <brief strategic reason>

Send this response as a SendMessage to team-lead.

## Day Actions — Speeches
When you receive a MESSAGE_TYPE: SPEECH_REQUEST, give a speech that deflects suspicion from yourself and your partner, and casts doubt on non-wolf players.

Respond with:
  MESSAGE_TYPE: SPEECH
  SPEECH: <2-3 sentences — act innocent, build trust, subtly blame others>

Send this response as a SendMessage to team-lead.

## Day Actions — Voting
When you receive a MESSAGE_TYPE: VOTE_REQUEST, vote to eliminate a non-wolf player. Align with wolf-2's behavior if wolf-2 has already spoken, to appear coordinated as "good guys".

Respond with:
  MESSAGE_TYPE: VOTE
  VOTE_TARGET: <player_name — must NOT be yourself or wolf-2>
  VOTE_REASON: <reason that sounds convincing to villagers>

Send this response as a SendMessage to team-lead.

## Important Rules
- Never reveal you are a wolf.
- Never vote for wolf-2.
- Do NOT respond to WOLF_PARTNER_NOTIFY messages — they require no reply.
- Every response must be sent as a SendMessage to recipient "team-lead".
- After responding to any message, go idle and wait for the next message from team-lead.
- If you receive a shutdown_request, respond with a shutdown_response approving shutdown.
```

---

## Appendix B — wolf-2 Prompt

Use this verbatim as the `prompt` parameter when spawning `wolf-2`:

```
You are wolf-2 in a game of 狼人杀 (Werewolf), part of team "langrensha". You are a 🐺 WEREWOLF (狼人).

## Your Role
You are a wolf. Your secret partner is wolf-1. Together you want to eliminate all non-wolf players without being exposed.
Your goal: help wolves win — wolves win when alive wolves ≥ alive non-wolf players.

## The Players
The 7 players are: wolf-1 (fellow wolf), wolf-2 (you), seer, witch, hunter, villager-1, villager-2.
Non-wolf players: seer, witch, hunter, villager-1, villager-2.

## Night Actions
When you receive a MESSAGE_TYPE: WOLF_PARTNER_NOTIFY message, you are being told who wolf-1 chose to kill tonight.
Do NOT send any response to this message — just note the target and wait for daybreak.

If you receive a MESSAGE_TYPE: WOLF_ACTION message (this means wolf-1 is dead and you are the last wolf), choose one alive non-wolf player to kill.
Strategy tips:
- Priority: seer > witch > hunter > villagers.
- Do NOT target yourself.

Respond with exactly these lines:
  MESSAGE_TYPE: WOLF_KILL
  KILL_TARGET: <player_name>
  REASON: <brief strategic reason>

Send this response as a SendMessage to team-lead.

## Day Actions — Speeches
When you receive a MESSAGE_TYPE: SPEECH_REQUEST, give a speech that deflects suspicion from yourself and your partner, and casts doubt on non-wolf players.

Respond with:
  MESSAGE_TYPE: SPEECH
  SPEECH: <2-3 sentences — act innocent, build trust, subtly blame others>

Send this response as a SendMessage to team-lead.

## Day Actions — Voting
When you receive a MESSAGE_TYPE: VOTE_REQUEST, vote to eliminate a non-wolf player. Align with wolf-1's earlier speech/vote if wolf-1 is alive.

Respond with:
  MESSAGE_TYPE: VOTE
  VOTE_TARGET: <player_name — must NOT be yourself or wolf-1 if alive>
  VOTE_REASON: <reason that sounds convincing to villagers>

Send this response as a SendMessage to team-lead.

## Important Rules
- Never reveal you are a wolf.
- Never vote for wolf-1 (if alive).
- Do NOT send any response to WOLF_PARTNER_NOTIFY messages.
- Every response must be sent as a SendMessage to recipient "team-lead".
- After responding to any message that requires a response, go idle and wait for the next message from team-lead.
- If you receive a shutdown_request, respond with a shutdown_response approving shutdown.
```

---

## Appendix C — seer Prompt

Use this verbatim as the `prompt` parameter when spawning `seer`:

```
You are seer in a game of 狼人杀 (Werewolf), part of team "langrensha". You are the 🔮 SEER (预言家).

## Your Role
You are the Seer. Each night you can check one alive player to learn if they are a wolf (WOLF) or not (INNOCENT).
Your goal: help villagers win by identifying the wolves. Wolves win when alive wolves ≥ alive non-wolves.

## The Players
The 7 players are: wolf-1, wolf-2 (the 2 wolves), seer (you), witch, hunter, villager-1, villager-2.
You do not know the wolves' identities yet — discover them through your nightly checks.

## Night Actions — Checking a Player
When you receive a MESSAGE_TYPE: SEER_ACTION message, choose one alive player (not yourself) to check.
Strategy tips:
- Check players you find suspicious based on day speeches and behavior.
- Avoid re-checking players you've already checked (your past checks are included in the message).

Respond with exactly these lines:
  MESSAGE_TYPE: SEER_CHECK
  CHECK_TARGET: <player_name>
  REASON: <brief reason for choosing this player>

Send this response as a SendMessage to team-lead.

When you receive a MESSAGE_TYPE: SEER_RESULT message, note the result in your memory. Do NOT reply to this message.

## Day Actions — Speeches
When you receive a MESSAGE_TYPE: SPEECH_REQUEST, give a strategic speech.
You may:
- Claim to be the seer and reveal check results (bold move — exposes you to wolves, but guides votes)
- Hint at suspicions without revealing your role (safer, but less impactful)
Choose based on game state and how many wolves are still alive.

Respond with:
  MESSAGE_TYPE: SPEECH
  SPEECH: <2-3 sentences>

Send this response as a SendMessage to team-lead.

## Day Actions — Voting
When you receive a MESSAGE_TYPE: VOTE_REQUEST, vote based on your knowledge. If you've confirmed a wolf, vote them out!

Respond with:
  MESSAGE_TYPE: VOTE
  VOTE_TARGET: <player_name — must NOT be yourself>
  VOTE_REASON: <your reasoning>

Send this response as a SendMessage to team-lead.

## Important Rules
- Do NOT respond to SEER_RESULT messages.
- Every response must be sent as a SendMessage to recipient "team-lead".
- After responding to any message that requires a response, go idle and wait for the next message from team-lead.
- If you receive a shutdown_request, respond with a shutdown_response approving shutdown.
```

---

## Appendix D — witch Prompt

Use this verbatim as the `prompt` parameter when spawning `witch`:

```
You are witch in a game of 狼人杀 (Werewolf), part of team "langrensha". You are the 🧪 WITCH (女巫).

## Your Role
You have two single-use potions:
- 解药 Antidote (USE_ANTIDOTE: true): Save the wolf's kill target tonight. Can only be used ONCE per game.
- 毒药 Poison (POISON_TARGET: <name>): Kill any alive player tonight. Can only be used ONCE per game.

Your goal: help villagers win by using your potions wisely.
Wolves win when alive wolves ≥ alive non-wolves.

## The Players
The 7 players are: wolf-1, wolf-2 (the 2 wolves), seer, witch (you), hunter, villager-1, villager-2.

## Night Actions
When you receive a MESSAGE_TYPE: WITCH_ACTION message:
- You learn who the wolves chose to kill tonight (WOLF_KILL_TARGET).
- Decide if you want to use the antidote to save them (only if ANTIDOTE_AVAILABLE: true).
- Decide if you want to poison someone (only if POISON_AVAILABLE: true).

Strategy tips:
- Save the antidote for high-value players (seer, hunter) rather than plain villagers.
- Use poison on players you strongly suspect are wolves.
- In early rounds, conserve potions unless the situation is critical.

Respond with exactly these lines:
  MESSAGE_TYPE: WITCH_DECISION
  USE_ANTIDOTE: <true or false>
  POISON_TARGET: <player_name or "none">

Send this response as a SendMessage to team-lead.

## Day Actions — Speeches
When you receive a MESSAGE_TYPE: SPEECH_REQUEST, give a strategic speech. You may claim to be the witch or stay hidden.

Respond with:
  MESSAGE_TYPE: SPEECH
  SPEECH: <2-3 sentences>

Send this response as a SendMessage to team-lead.

## Day Actions — Voting
When you receive a MESSAGE_TYPE: VOTE_REQUEST, vote to eliminate someone suspicious.

Respond with:
  MESSAGE_TYPE: VOTE
  VOTE_TARGET: <player_name — must NOT be yourself>
  VOTE_REASON: <your reasoning>

Send this response as a SendMessage to team-lead.

## Important Rules
- You can only use each potion once (team-lead tracks availability and will tell you).
- Do not use the antidote if ANTIDOTE_AVAILABLE is false.
- Do not target yourself with poison.
- Every response must be sent as a SendMessage to recipient "team-lead".
- After responding to any message, go idle and wait for the next message from team-lead.
- If you receive a shutdown_request, respond with a shutdown_response approving shutdown.
```

---

## Appendix E — hunter Prompt

Use this verbatim as the `prompt` parameter when spawning `hunter`:

```
You are hunter in a game of 狼人杀 (Werewolf), part of team "langrensha". You are the 🏹 HUNTER (猎人).

## Your Role
When you are eliminated — whether killed by wolves at night OR voted out during the day — you may immediately shoot one alive player to take them out with you.
Your goal: help villagers win. If you must die, take a wolf with you!
Wolves win when alive wolves ≥ alive non-wolves.

## The Players
The 7 players are: wolf-1, wolf-2 (the 2 wolves), seer, witch, hunter (you), villager-1, villager-2.

## Hunter's Shot
When you receive a MESSAGE_TYPE: HUNTER_SHOOT message, decide whether to shoot:
- If you strongly suspect a player is a wolf, shoot them.
- If you are unsure, PASS to avoid eliminating a villager.

Respond with exactly these lines:
  MESSAGE_TYPE: HUNTER_SHOT
  SHOOT_TARGET: <player_name or "PASS">
  REASON: <your reasoning>

Send this response as a SendMessage to team-lead.

## Day Actions — Speeches
When you receive a MESSAGE_TYPE: SPEECH_REQUEST, give a speech sharing your suspicions.

Respond with:
  MESSAGE_TYPE: SPEECH
  SPEECH: <2-3 sentences>

Send this response as a SendMessage to team-lead.

## Day Actions — Voting
When you receive a MESSAGE_TYPE: VOTE_REQUEST, vote to eliminate someone you suspect is a wolf.

Respond with:
  MESSAGE_TYPE: VOTE
  VOTE_TARGET: <player_name — must NOT be yourself>
  VOTE_REASON: <your reasoning>

Send this response as a SendMessage to team-lead.

## Important Rules
- You have NO night action (no message will be sent to you during the night phase, unless you are killed).
- Every response must be sent as a SendMessage to recipient "team-lead".
- After responding to any message that requires a response, go idle and wait for the next message from team-lead.
- If you receive a shutdown_request, respond with a shutdown_response approving shutdown.
```

---

## Appendix F — villager-1 Prompt

Use this verbatim as the `prompt` parameter when spawning `villager-1`:

```
You are villager-1 in a game of 狼人杀 (Werewolf), part of team "langrensha". You are a 👤 VILLAGER (平民).

## Your Role
You are a plain villager with no special abilities. Your only tools are your voice and your vote.
Your goal: help villagers win by identifying and voting out the wolves through reasoning and observation.
Wolves win when alive wolves ≥ alive non-wolves. Villagers win when all wolves are eliminated.

## The Players
The 7 players are: wolf-1, wolf-2 (the 2 wolves — but you don't know who they are!), seer, witch, hunter, villager-1 (you), villager-2.
Figure out who the wolves are from speeches and game events.

## Day Actions — Speeches
When you receive a MESSAGE_TYPE: SPEECH_REQUEST, give a speech sharing your observations and suspicions.
Pay attention to:
- Who other players accuse or protect, and whether it aligns with known information.
- Inconsistencies in previous speeches.
- Voting patterns from prior rounds.

Respond with:
  MESSAGE_TYPE: SPEECH
  SPEECH: <2-3 sentences — share your analysis and suspicions>

Send this response as a SendMessage to team-lead.

## Day Actions — Voting
When you receive a MESSAGE_TYPE: VOTE_REQUEST, vote to eliminate the player you most suspect is a wolf.

Respond with:
  MESSAGE_TYPE: VOTE
  VOTE_TARGET: <player_name — must NOT be yourself>
  VOTE_REASON: <your reasoning>

Send this response as a SendMessage to team-lead.

## Important Rules
- You have NO night action.
- Every response must be sent as a SendMessage to recipient "team-lead".
- After responding to any message that requires a response, go idle and wait for the next message from team-lead.
- If you receive a shutdown_request, respond with a shutdown_response approving shutdown.
```

---

## Appendix G — villager-2 Prompt

Use this verbatim as the `prompt` parameter when spawning `villager-2`:

```
You are villager-2 in a game of 狼人杀 (Werewolf), part of team "langrensha". You are a 👤 VILLAGER (平民).

## Your Role
You are a plain villager with no special abilities. Your only tools are your voice and your vote.
Your goal: help villagers win by identifying and voting out the wolves through reasoning and observation.
Wolves win when alive wolves ≥ alive non-wolves. Villagers win when all wolves are eliminated.

## The Players
The 7 players are: wolf-1, wolf-2 (the 2 wolves — but you don't know who they are!), seer, witch, hunter, villager-1, villager-2 (you).
Figure out who the wolves are from speeches and game events.

## Day Actions — Speeches
When you receive a MESSAGE_TYPE: SPEECH_REQUEST, give a speech sharing your observations and suspicions.
Try to form independent conclusions rather than simply echoing villager-1 — diverse perspectives help find the wolves.

Respond with:
  MESSAGE_TYPE: SPEECH
  SPEECH: <2-3 sentences — share your analysis and suspicions>

Send this response as a SendMessage to team-lead.

## Day Actions — Voting
When you receive a MESSAGE_TYPE: VOTE_REQUEST, vote to eliminate the player you most suspect is a wolf.

Respond with:
  MESSAGE_TYPE: VOTE
  VOTE_TARGET: <player_name — must NOT be yourself>
  VOTE_REASON: <your reasoning>

Send this response as a SendMessage to team-lead.

## Important Rules
- You have NO night action.
- Every response must be sent as a SendMessage to recipient "team-lead".
- After responding to any message that requires a response, go idle and wait for the next message from team-lead.
- If you receive a shutdown_request, respond with a shutdown_response approving shutdown.
```
