---
name: add-telegram
description: Add Telegram as a channel. Can replace WhatsApp entirely or run alongside it. Also configurable as a control-only channel (triggers actions) or passive channel (receives notifications only).
---

# Add Telegram Channel

This skill adds Telegram support to NanoClaw using the skills engine for deterministic code changes, then walks through interactive setup.

## Phase 1: Pre-flight

### Check if already applied

Read `.nanoclaw/state.yaml`. If `telegram` is in `applied_skills`, skip to Phase 3 (Setup). The code changes are already in place.

### Ask the user

Use `AskUserQuestion` to collect configuration:

AskUserQuestion: Should Telegram replace WhatsApp or run alongside it?
- **Replace WhatsApp** - Telegram will be the only channel (sets TELEGRAM_ONLY=true)
- **Alongside** - Both Telegram and WhatsApp channels active

AskUserQuestion: Do you have a Telegram bot token, or do you need to create one?

If they have one, collect it now. If not, we'll create one in Phase 3.

## Phase 2: Apply Code Changes

Run the skills engine to apply this skill's code package. The package files are in this directory alongside this SKILL.md.

### Initialize skills system (if needed)

If `.nanoclaw/` directory doesn't exist yet:

```bash
npx tsx scripts/apply-skill.ts --init
```

Or call `initSkillsSystem()` from `skills-engine/migrate.ts`.

### Apply the skill

```bash
npx tsx scripts/apply-skill.ts .claude/skills/add-telegram
```

This deterministically:
- Adds `src/channels/telegram.ts` (TelegramChannel class implementing Channel interface)
- Adds `src/channels/telegram.test.ts` (46 unit tests)
- Three-way merges Telegram support into `src/index.ts` (multi-channel support, findChannel routing)
- Three-way merges Telegram config into `src/config.ts` (TELEGRAM_BOT_TOKEN, TELEGRAM_ONLY exports)
- Three-way merges updated routing tests into `src/routing.test.ts`
- Installs the `grammy` npm dependency
- Updates `.env.example` with `TELEGRAM_BOT_TOKEN` and `TELEGRAM_ONLY`
- Records the application in `.nanoclaw/state.yaml`

If the apply reports merge conflicts, read the intent files:
- `modify/src/index.ts.intent.md` — what changed and invariants for index.ts
- `modify/src/config.ts.intent.md` — what changed for config.ts

### Validate code changes

```bash
npm test
npm run build
```

All tests must pass (including the new telegram tests) and build must be clean before proceeding.

## Phase 3: Setup

### Create Telegram Bot (if needed)

If the user doesn't have a bot token, tell them:

> I need you to create a Telegram bot:
>
> 1. Open Telegram and search for `@BotFather`
> 2. Send `/newbot` and follow prompts:
>    - Bot name: Something friendly (e.g., "Andy Assistant")
>    - Bot username: Must end with "bot" (e.g., "andy_ai_bot")
> 3. Copy the bot token (looks like `123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11`)

Wait for the user to provide the token.

### Configure environment

Add to `.env`:

```bash
TELEGRAM_BOT_TOKEN=<their-token>
```

If they chose to replace WhatsApp:

```bash
TELEGRAM_ONLY=true
```

Sync to container environment:

```bash
mkdir -p data/env && cp .env data/env/env
```

The container reads environment from `data/env/env`, not `.env` directly.

### Disable Group Privacy (for group chats)

Tell the user:

> **Important for group chats**: By default, Telegram bots only see @mentions and commands in groups. To let the bot see all messages:
>
> 1. Open Telegram and search for `@BotFather`
> 2. Send `/mybots` and select your bot
> 3. Go to **Bot Settings** > **Group Privacy** > **Turn off**
>
> This is optional if you only want trigger-based responses via @mentioning the bot.

### Build and restart

```bash
npm run build
launchctl kickstart -k gui/$(id -u)/com.nanoclaw  # macOS
# Linux: systemctl --user restart nanoclaw
```

## Phase 4: Registration

### Get Chat ID

Tell the user:

> 1. Open your bot in Telegram (search for its username)
> 2. Send `/chatid` — it will reply with the chat ID
> 3. For groups: add the bot to the group first, then send `/chatid` in the group

Wait for the user to provide the chat ID (format: `tg:123456789` or `tg:-1001234567890`).

### Register the chat

Use the IPC register flow or register directly. The chat ID, name, and folder name are needed.

For a main chat (responds to all messages, uses the `main` folder):

```typescript
registerGroup("tg:<chat-id>", {
  name: "<chat-name>",
  folder: "main",
  trigger: `@${ASSISTANT_NAME}`,
  added_at: new Date().toISOString(),
  requiresTrigger: false,
});
```

For additional chats (trigger-only):

```typescript
registerGroup("tg:<chat-id>", {
  name: "<chat-name>",
  folder: "<folder-name>",
  trigger: `@${ASSISTANT_NAME}`,
  added_at: new Date().toISOString(),
  requiresTrigger: true,
});
```

## Phase 5: Verify

### Test the connection

Tell the user:

> Send a message to your registered Telegram chat:
> - For main chat: Any message works
> - For non-main: `@Andy hello` or @mention the bot
>
> The bot should respond within a few seconds.

### Check logs if needed

```bash
tail -f logs/nanoclaw.log
```

## Troubleshooting

### Bot not responding

Check:
1. `TELEGRAM_BOT_TOKEN` is set in `.env` AND synced to `data/env/env`
2. Chat is registered in SQLite (check with: `sqlite3 store/messages.db "SELECT * FROM registered_groups WHERE jid LIKE 'tg:%'"`)
3. For non-main chats: message includes trigger pattern
4. Service is running: `launchctl list | grep nanoclaw` (macOS) or `systemctl --user status nanoclaw` (Linux)

### Bot only responds to @mentions in groups

Group Privacy is enabled (default). Fix:
1. `@BotFather` > `/mybots` > select bot > **Bot Settings** > **Group Privacy** > **Turn off**
2. Remove and re-add the bot to the group (required for the change to take effect)

### Getting chat ID

If `/chatid` doesn't work:
- Verify token: `curl -s "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/getMe"`
- Check bot is started: `tail -f logs/nanoclaw.log`

## After Setup

If running `npm run dev` while the service is active:
```bash
# macOS:
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist
npm run dev
# When done testing:
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist
# Linux:
# systemctl --user stop nanoclaw
# npm run dev
# systemctl --user start nanoclaw
```

## Agent Swarms (Teams)

After completing the Telegram setup, use `AskUserQuestion`:

AskUserQuestion: Would you like to add Agent Swarm support? Without it, Agent Teams still work — they just operate behind the scenes. With Swarm support, each subagent appears as a different bot in the Telegram group so you can see who's saying what and have interactive team sessions.
- **Yes** — Set up pool bots for visible agent identities
- **No** — Skip, teams will work behind the scenes

If they say no, skip to "After Setup".

If they say yes, continue with the swarm setup steps below.

### How It Works

- The **main bot** receives messages and sends lead agent responses (already set up above)
- **Pool bots** are send-only — each gets a Grammy `Api` instance (no polling)
- When a subagent calls `send_message` with a `sender` parameter, the host assigns a pool bot and renames it to match the sender's role
- Messages appear in Telegram from different bot identities

```
Subagent calls send_message(text: "Found 3 results", sender: "Researcher")
  → MCP writes IPC file with sender field
  → Host IPC watcher picks it up
  → Assigns pool bot #2 to "Researcher" (round-robin, stable per-group)
  → Renames pool bot #2 to "Researcher" via setMyName
  → Sends message via pool bot #2's Api instance
  → Appears in Telegram from "Researcher" bot
```

### Swarm Step 1: Create Pool Bots

Tell the user:

> I need you to create 3-5 Telegram bots to use as the agent pool. These will be renamed dynamically to match agent roles.
>
> 1. Open Telegram and search for `@BotFather`
> 2. Send `/newbot` for each bot:
>    - Give them any placeholder name (e.g., "Bot 1", "Bot 2")
>    - Usernames like `myproject_swarm_1_bot`, `myproject_swarm_2_bot`, etc.
> 3. Copy all the tokens
> 4. Add all bots to your Telegram group(s) where you want agent teams

Wait for user to provide the tokens.

### Swarm Step 2: Disable Group Privacy for Pool Bots

Tell the user:

> **Important**: Each pool bot needs Group Privacy disabled so it can send messages in groups.
>
> For each pool bot in `@BotFather`:
> 1. Send `/mybots` and select the bot
> 2. Go to **Bot Settings** > **Group Privacy** > **Turn off**
>
> Then add all pool bots to your Telegram group(s).

### Swarm Step 3: Update Configuration

Read `src/config.ts` and add the bot pool config near the other Telegram exports:

```typescript
export const TELEGRAM_BOT_POOL = (process.env.TELEGRAM_BOT_POOL || '')
  .split(',')
  .map((t) => t.trim())
  .filter(Boolean);
```

### Swarm Step 4: Add Bot Pool to Telegram Module

Read `src/telegram.ts` and add the following:

1. **Update imports** — add `Api` to the Grammy import:

```typescript
import { Api, Bot } from 'grammy';
```

2. **Add pool state** after the existing `let bot` declaration:

```typescript
// Bot pool for agent teams: send-only Api instances (no polling)
const poolApis: Api[] = [];
// Maps "{groupFolder}:{senderName}" → pool Api index for stable assignment
const senderBotMap = new Map<string, number>();
let nextPoolIndex = 0;
```

3. **Add pool functions** — place these before the `isTelegramConnected` function:

```typescript
/**
 * Initialize send-only Api instances for the bot pool.
 * Each pool bot can send messages but doesn't poll for updates.
 */
export async function initBotPool(tokens: string[]): Promise<void> {
  for (const token of tokens) {
    try {
      const api = new Api(token);
      const me = await api.getMe();
      poolApis.push(api);
      logger.info(
        { username: me.username, id: me.id, poolSize: poolApis.length },
        'Pool bot initialized',
      );
    } catch (err) {
      logger.error({ err }, 'Failed to initialize pool bot');
    }
  }
  if (poolApis.length > 0) {
    logger.info({ count: poolApis.length }, 'Telegram bot pool ready');
  }
}

/**
 * Send a message via a pool bot assigned to the given sender name.
 * Assigns bots round-robin on first use; subsequent messages from the
 * same sender in the same group always use the same bot.
 * On first assignment, renames the bot to match the sender's role.
 */
export async function sendPoolMessage(
  chatId: string,
  text: string,
  sender: string,
  groupFolder: string,
): Promise<void> {
  if (poolApis.length === 0) {
    // No pool bots — fall back to main bot
    await sendTelegramMessage(chatId, text);
    return;
  }

  const key = `${groupFolder}:${sender}`;
  let idx = senderBotMap.get(key);
  if (idx === undefined) {
    idx = nextPoolIndex % poolApis.length;
    nextPoolIndex++;
    senderBotMap.set(key, idx);
    // Rename the bot to match the sender's role, then wait for Telegram to propagate
    try {
      await poolApis[idx].setMyName(sender);
      await new Promise((r) => setTimeout(r, 2000));
      logger.info({ sender, groupFolder, poolIndex: idx }, 'Assigned and renamed pool bot');
    } catch (err) {
      logger.warn({ sender, err }, 'Failed to rename pool bot (sending anyway)');
    }
  }

  const api = poolApis[idx];
  try {
    const numericId = chatId.replace(/^tg:/, '');
    const MAX_LENGTH = 4096;
    if (text.length <= MAX_LENGTH) {
      await api.sendMessage(numericId, text);
    } else {
      for (let i = 0; i < text.length; i += MAX_LENGTH) {
        await api.sendMessage(numericId, text.slice(i, i + MAX_LENGTH));
      }
    }
    logger.info({ chatId, sender, poolIndex: idx, length: text.length }, 'Pool message sent');
  } catch (err) {
    logger.error({ chatId, sender, err }, 'Failed to send pool message');
  }
}
```

### Swarm Step 5: Add sender Parameter to MCP Tool

Read `container/agent-runner/src/ipc-mcp-stdio.ts` and update the `send_message` tool to accept an optional `sender` parameter:

Change the tool's schema from:
```typescript
{ text: z.string().describe('The message text to send') },
```

To:
```typescript
{
  text: z.string().describe('The message text to send'),
  sender: z.string().optional().describe('Your role/identity name (e.g. "Researcher"). When set, messages appear from a dedicated bot in Telegram.'),
},
```

And update the handler to include `sender` in the IPC data:

```typescript
async (args) => {
    const data: Record<string, string | undefined> = {
      type: 'message',
      chatJid,
      text: args.text,
      sender: args.sender || undefined,
      groupFolder,
      timestamp: new Date().toISOString(),
    };

    writeIpcFile(MESSAGES_DIR, data);

    return { content: [{ type: 'text' as const, text: 'Message sent.' }] };
  },
```

### Swarm Step 6: Update Host IPC Routing

Read `src/ipc.ts` and make these changes:

1. **Add imports** — add `sendPoolMessage` and `initBotPool` from the Telegram module, and `TELEGRAM_BOT_POOL` from config.

2. **Update IPC message routing** — in `src/ipc.ts`, find where the `sendMessage` dependency is called to deliver IPC messages (inside `processIpcFiles`). Wrap it to route Telegram swarm messages through the bot pool:

```typescript
if (data.sender && data.chatJid.startsWith('tg:')) {
  await sendPoolMessage(
    data.chatJid,
    data.text,
    data.sender,
    sourceGroup,
  );
} else {
  await deps.sendMessage(data.chatJid, data.text);
}
```

3. **Initialize pool in `main()` in `src/index.ts`** — after creating the Telegram channel, add:

```typescript
if (TELEGRAM_BOT_POOL.length > 0) {
  await initBotPool(TELEGRAM_BOT_POOL);
}
```

### Swarm Step 7: Update CLAUDE.md Files

#### 7a. Add global message formatting rules

Read `groups/global/CLAUDE.md` and add a Message Formatting section:

```markdown
## Message Formatting

NEVER use markdown. Only use WhatsApp/Telegram formatting:
- *single asterisks* for bold (NEVER **double asterisks**)
- _underscores_ for italic
- • bullet points
- ```triple backticks``` for code

No ## headings. No [links](url). No **double stars**.
```

#### 7b. Update existing group CLAUDE.md headings

In any group CLAUDE.md that has a "WhatsApp Formatting" section (e.g. `groups/main/CLAUDE.md`), rename the heading to reflect multi-channel support:

```
## WhatsApp Formatting (and other messaging apps)
```

#### 7c. Add Agent Teams instructions to Telegram groups

For each Telegram group that will use agent teams, create or update its `groups/{folder}/CLAUDE.md` with these instructions. Read the existing CLAUDE.md first (or `groups/global/CLAUDE.md` as a base) and add the Agent Teams section:

```markdown
## Agent Teams

When creating a team to tackle a complex task, follow these rules:

### CRITICAL: Follow the user's prompt exactly

Create *exactly* the team the user asked for — same number of agents, same roles, same names. Do NOT add extra agents, rename roles, or use generic names like "Researcher 1". If the user says "a marine biologist, a physicist, and Alexander Hamilton", create exactly those three agents with those exact names.

### Team member instructions

Each team member MUST be instructed to:

1. *Share progress in the group* via `mcp__nanoclaw__send_message` with a `sender` parameter matching their exact role/character name (e.g., `sender: "Marine Biologist"` or `sender: "Alexander Hamilton"`). This makes their messages appear from a dedicated bot in the Telegram group.
2. *Also communicate with teammates* via `SendMessage` as normal for coordination.
3. Keep group messages *short* — 2-4 sentences max per message. Break longer content into multiple `send_message` calls. No walls of text.
4. Use the `sender` parameter consistently — always the same name so the bot identity stays stable.
5. NEVER use markdown formatting. Use ONLY WhatsApp/Telegram formatting: single *asterisks* for bold (NOT **double**), _underscores_ for italic, • for bullets, ```backticks``` for code. No ## headings, no [links](url), no **double asterisks**.

### Example team creation prompt

When creating a teammate, include instructions like:

\```
You are the Marine Biologist. When you have findings or updates for the user, send them to the group using mcp__nanoclaw__send_message with sender set to "Marine Biologist". Keep each message short (2-4 sentences max). Use emojis for strong reactions. ONLY use single *asterisks* for bold (never **double**), _underscores_ for italic, • for bullets. No markdown. Also communicate with teammates via SendMessage.
\```

### Lead agent behavior

As the lead agent who created the team:

- You do NOT need to react to or relay every teammate message. The user sees those directly from the teammate bots.
- Send your own messages only to comment, share thoughts, synthesize, or direct the team.
- When processing an internal update from a teammate that doesn't need a user-facing response, wrap your *entire* output in `<internal>` tags.
- Focus on high-level coordination and the final synthesis.
```

### Swarm Step 8: Update Environment

Add pool tokens to `.env`:

```bash
TELEGRAM_BOT_POOL=TOKEN1,TOKEN2,TOKEN3,...
```

**Important**: Sync to all required locations:

```bash
cp .env data/env/env
```

Also add `TELEGRAM_BOT_POOL` to the launchd plist (`~/Library/LaunchAgents/com.nanoclaw.plist`) in the `EnvironmentVariables` dict if using launchd.

### Swarm Step 9: Rebuild and Test

```bash
npm run build
./container/build.sh  # Required — MCP tool changed
# macOS:
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist
# Linux:
# systemctl --user restart nanoclaw
```

Must use `unload/load` (macOS) or `restart` (Linux) because the service env vars changed.

Tell the user:

> Send a message in your Telegram group asking for a multi-agent task, e.g.:
> "Assemble a team of a researcher and a coder to build me a hello world app"
>
> You should see:
> - The lead agent (main bot) acknowledging and creating the team
> - Each subagent messaging from a different bot, renamed to their role
> - Short, scannable messages from each agent
>
> Check logs: `tail -f logs/nanoclaw.log | grep -i pool`

### Swarm Architecture Notes

- Pool bots use Grammy's `Api` class — lightweight, no polling, just send
- Bot names are set via `setMyName` — changes are global to the bot, not per-chat
- A 2-second delay after `setMyName` allows Telegram to propagate the name change before the first message
- Sender→bot mapping is stable within a group (keyed as `{groupFolder}:{senderName}`)
- Mapping resets on service restart — pool bots get reassigned fresh
- If pool runs out, bots are reused (round-robin wraps)

### Swarm Troubleshooting

**Pool bots not sending messages:**
1. Verify tokens: `curl -s "https://api.telegram.org/botTOKEN/getMe"`
2. Check pool initialized: `grep "Pool bot" logs/nanoclaw.log`
3. Ensure all pool bots are members of the Telegram group
4. Check Group Privacy is disabled for each pool bot

**Bot names not updating:**
Telegram caches bot names client-side. The 2-second delay after `setMyName` helps, but users may need to restart their Telegram client to see updated names immediately.

**Subagents not using send_message:**
Check the group's `CLAUDE.md` has the Agent Teams instructions. The lead agent reads this when creating teammates and must include the `send_message` + `sender` instructions in each teammate's prompt.

### Swarm Removal

To remove Agent Swarm support while keeping basic Telegram:

1. Remove `TELEGRAM_BOT_POOL` from `src/config.ts`
2. Remove pool code from `src/telegram.ts` (`poolApis`, `senderBotMap`, `initBotPool`, `sendPoolMessage`)
3. Remove pool routing from IPC handler in `src/index.ts` (revert to plain `sendMessage`)
4. Remove `initBotPool` call from `main()`
5. Remove `sender` param from MCP tool in `container/agent-runner/src/ipc-mcp-stdio.ts`
6. Remove Agent Teams section from group CLAUDE.md files
7. Remove `TELEGRAM_BOT_POOL` from `.env`, `data/env/env`, and launchd plist/systemd unit
8. Rebuild: `npm run build && ./container/build.sh && launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist && launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist` (macOS) or `npm run build && ./container/build.sh && systemctl --user restart nanoclaw` (Linux)

## Removal

To remove Telegram integration:

1. Delete `src/channels/telegram.ts`
2. Remove `TelegramChannel` import and creation from `src/index.ts`
3. Remove `channels` array and revert to using `whatsapp` directly in `processGroupMessages`, scheduler deps, and IPC deps
4. Revert `getAvailableGroups()` filter to only include `@g.us` chats
5. Remove Telegram config (`TELEGRAM_BOT_TOKEN`, `TELEGRAM_ONLY`) from `src/config.ts`
6. Remove Telegram registrations from SQLite: `sqlite3 store/messages.db "DELETE FROM registered_groups WHERE jid LIKE 'tg:%'"`
7. Uninstall: `npm uninstall grammy`
8. Rebuild: `npm run build && launchctl kickstart -k gui/$(id -u)/com.nanoclaw` (macOS) or `npm run build && systemctl --user restart nanoclaw` (Linux)
