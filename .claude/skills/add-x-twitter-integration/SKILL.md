---
name: add-x-twitter-integration
description: Add X (Twitter) integration to NanoClaw. Post tweets, like, reply, retweet, and quote via browser automation. Triggers on "add x", "add twitter", "x integration", "twitter integration".
---

# Add X (Twitter) Integration

This skill adds X (Twitter) integration to NanoClaw via browser automation. Uses the user's real Chrome browser to avoid API costs and bot detection.

> **Compatibility:** NanoClaw v1.0.0. Directory structure may change in future versions.

## Features

| Action | Tool | Description |
|--------|------|-------------|
| Post | `x_post` | Publish new tweets |
| Like | `x_like` | Like any tweet |
| Reply | `x_reply` | Reply to tweets |
| Retweet | `x_retweet` | Retweet without comment |
| Quote | `x_quote` | Quote tweet with comment |

## Phase 1: Pre-flight

### Check if already applied

Read `.nanoclaw/state.yaml`. If `x-twitter-integration` is in `applied_skills`, skip to Phase 3 (Setup). The code changes are already in place.

### Ask the user

Use `AskUserQuestion` to collect configuration:

AskUserQuestion: Which X actions do you want enabled?
(multi-select, default all)
- **Post** — Publish new tweets
- **Like** — Like any tweet
- **Reply** — Reply to tweets
- **Retweet** — Retweet without comment
- **Quote** — Quote tweet with comment

Store selection as `X_ENABLED_ACTIONS` env var (comma-separated, e.g. `post,like,reply,retweet,quote`).

AskUserQuestion: Should the agent ask permission before posting tweets, or post autonomously?
- **Ask permission** — Agent confirms with you before posting (sets `X_REQUIRE_CONFIRMATION=true`)
- **Post autonomously** — Agent posts without asking (sets `X_REQUIRE_CONFIRMATION=false`)

AskUserQuestion: Which groups should have access to X tools?
- **Main only** — Only the main group can use X tools (sets `X_MAIN_ONLY=true`, default)
- **All groups** — All registered groups can use X tools (sets `X_MAIN_ONLY=false`)

## Phase 2: Apply Code Changes

### Prerequisites

Ensure NanoClaw is installed and running, then install dependencies:

```bash
npm ls playwright dotenv-cli || npm install playwright dotenv-cli
```

Check `CHROME_PATH`:

```bash
# macOS:
mdfind "kMDItemCFBundleIdentifier == 'com.google.Chrome'" 2>/dev/null | head -1
# Linux:
which google-chrome || which chromium-browser
```

If Chrome is not at the default location, tell the user to add to `.env`:
```bash
CHROME_PATH=/path/to/Google Chrome
```

### Integration Points

To integrate this skill into NanoClaw, make the following modifications:

---

**1. Host side: `src/ipc.ts`**

Add import after other local imports:
```typescript
import { handleXIpc } from '../.claude/skills/add-x-twitter-integration/host.js';
```

Modify `processTaskIpc` function's switch statement default case:
```typescript
// Find:
default:
logger.warn({ type: data.type }, 'Unknown IPC task type');

// Replace with:
default:
const handled = await handleXIpc(data, sourceGroup, isMain, DATA_DIR);
if (!handled) {
    logger.warn({ type: data.type }, 'Unknown IPC task type');
}
```

---

**2. Container side: `container/agent-runner/src/ipc-mcp.ts`**

Add import after `cron-parser` import:
```typescript
// @ts-ignore - Copied during Docker build from .claude/skills/add-x-twitter-integration/
import { createXTools } from './skills/add-x-twitter-integration/agent.js';
```

Add to the end of tools array (before the closing `]`):
```typescript
    ...createXTools({ groupFolder, isMain })
```

---

**3. Build script: `container/build.sh`**

Change build context from `container/` to project root (required to access `.claude/skills/`):
```bash
# Find:
docker build -t "${IMAGE_NAME}:${TAG}" .

# Replace with:
cd "$SCRIPT_DIR/.."
docker build -t "${IMAGE_NAME}:${TAG}" -f container/Dockerfile .
```

---

**4. Dockerfile: `container/Dockerfile`**

First, update the build context paths (required to access `.claude/skills/` from project root):
```dockerfile
# Find:
COPY agent-runner/package*.json ./
...
COPY agent-runner/ ./

# Replace with:
COPY container/agent-runner/package*.json ./
...
COPY container/agent-runner/ ./
```

Then add COPY line after `COPY container/agent-runner/ ./` and before `RUN npm run build`:
```dockerfile
# Copy skill MCP tools
COPY .claude/skills/add-x-twitter-integration/agent.ts ./src/skills/add-x-twitter-integration/
```

---

### Configure environment

Add to `.env` based on user's choices:

```bash
X_ENABLED_ACTIONS=post,like,reply,retweet,quote
X_REQUIRE_CONFIRMATION=true
X_MAIN_ONLY=true
```

Sync to container environment:

```bash
mkdir -p data/env && cp .env data/env/env
```

### Validate code changes

```bash
npm run build
```

Build must be clean before proceeding.

## Phase 3: Setup

### Check Chrome Path

```bash
cat .env | grep CHROME_PATH
ls -la "$(grep CHROME_PATH .env | cut -d= -f2)" 2>/dev/null || \
echo "Chrome not found - update CHROME_PATH in .env"
```

### Run Authentication

```bash
npx dotenv -e .env -- npx tsx .claude/skills/add-x-twitter-integration/scripts/setup.ts
```

This opens Chrome for manual X login. Session saved to `data/x-browser-profile/`.

**Verify success:**
```bash
cat data/x-auth.json  # Should show {"authenticated": true, ...}
```

### Rebuild Container

```bash
./container/build.sh
```

**Verify success:**
```bash
./container/build.sh 2>&1 | grep -i "agent.ts"  # Should show COPY line
```

### Restart Service

```bash
npm run build
launchctl kickstart -k gui/$(id -u)/com.nanoclaw  # macOS
# Linux: systemctl --user restart nanoclaw
```

**Verify success:**
```bash
launchctl list | grep nanoclaw  # macOS — should show PID and exit code 0 or -
# Linux: systemctl --user status nanoclaw
```

## Phase 4: Verify

### Test via WhatsApp

Replace `@Assistant` with your configured trigger name (`ASSISTANT_NAME` in `.env`):

```
@Assistant post a tweet: Hello world!

@Assistant like this tweet https://x.com/user/status/123

@Assistant reply to https://x.com/user/status/123 with: Great post!

@Assistant retweet https://x.com/user/status/123

@Assistant quote https://x.com/user/status/123 with comment: Interesting
```

**Note:** If `X_MAIN_ONLY=true`, only the main group can use X tools. Other groups will receive an error.

### Check Authentication Status

```bash
cat data/x-auth.json 2>/dev/null && echo "Auth configured" || echo "Auth not configured"
ls -la data/x-browser-profile/ 2>/dev/null | head -5
```

### Re-authenticate (if expired)

```bash
npx dotenv -e .env -- npx tsx .claude/skills/add-x-twitter-integration/scripts/setup.ts
```

### Test Post (will actually post)

```bash
echo '{"content":"Test tweet - please ignore"}' | npx dotenv -e .env -- npx tsx .claude/skills/add-x-twitter-integration/scripts/post.ts
```

### Test Like

```bash
echo '{"tweetUrl":"https://x.com/user/status/123"}' | npx dotenv -e .env -- npx tsx .claude/skills/add-x-twitter-integration/scripts/like.ts
```

### Check logs if needed

```bash
grep -i "x_post\|x_like\|x_reply\|handleXIpc" logs/nanoclaw.log | tail -20
```

## Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `CHROME_PATH` | `/Applications/Google Chrome.app/Contents/MacOS/Google Chrome` | Chrome executable path |
| `X_ENABLED_ACTIONS` | `post,like,reply,retweet,quote` | Comma-separated list of enabled actions |
| `X_REQUIRE_CONFIRMATION` | `true` | Whether agent must ask before posting |
| `X_MAIN_ONLY` | `true` | Restrict X tools to main group only |
| `NANOCLAW_ROOT` | `process.cwd()` | Project root directory |
| `LOG_LEVEL` | `info` | Logging level (debug, info, warn, error) |

### Configuration File

Edit `.claude/skills/add-x-twitter-integration/lib/config.ts` to modify defaults:

```typescript
export const config = {
    // Browser viewport
    viewport: { width: 1280, height: 800 },

    // Timeouts (milliseconds)
    timeouts: {
        navigation: 30000,    // Page navigation
        elementWait: 5000,    // Wait for element
        afterClick: 1000,     // Delay after click
        afterFill: 1000,      // Delay after form fill
        afterSubmit: 3000,    // Delay after submit
        pageLoad: 3000,       // Initial page load
    },

    // Tweet limits
    limits: {
        tweetMaxLength: 280,
    },
};
```

### Data Directories

Paths relative to project root:

| Path | Purpose | Git |
|------|---------|-----|
| `data/x-browser-profile/` | Chrome profile with X session | Ignored |
| `data/x-auth.json` | Auth state marker | Ignored |
| `logs/nanoclaw.log` | Service logs (contains X operation logs) | Ignored |

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Container (Linux VM)                                       │
│  └── agent.ts → MCP tool definitions (x_post, etc.)    │
│      └── Writes IPC request to /workspace/ipc/tasks/       │
└──────────────────────┬──────────────────────────────────────┘
                       │ IPC (file system)
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  Host                                                       │
│  └── src/ipc.ts → processTaskIpc()                         │
│      └── host.ts → handleXIpc()                         │
│          └── spawn subprocess → scripts/*.ts               │
│              └── Playwright → Chrome → X Website           │
└─────────────────────────────────────────────────────────────┘
```

### Why This Design?

- **API is expensive** - X official API requires paid subscription ($100+/month) for posting
- **Bot browsers get blocked** - X detects and bans headless browsers and common automation fingerprints
- **Must use user's real browser** - Reuses the user's actual Chrome on Host with real browser fingerprint to avoid detection
- **One-time authorization** - User logs in manually once, session persists in Chrome profile for future use

### File Structure

```
.claude/skills/add-x-twitter-integration/
├── SKILL.md          # This documentation
├── host.ts           # Host-side IPC handler
├── agent.ts          # Container-side MCP tool definitions
├── lib/
│   ├── config.ts     # Centralized configuration
│   └── browser.ts    # Playwright utilities
└── scripts/
    ├── setup.ts      # Interactive login
    ├── post.ts       # Post tweet
    ├── like.ts       # Like tweet
    ├── reply.ts      # Reply to tweet
    ├── retweet.ts    # Retweet
    └── quote.ts      # Quote tweet
```

## Troubleshooting

### Authentication Expired

```bash
npx dotenv -e .env -- npx tsx .claude/skills/add-x-twitter-integration/scripts/setup.ts
launchctl kickstart -k gui/$(id -u)/com.nanoclaw  # macOS
# Linux: systemctl --user restart nanoclaw
```

### Browser Lock Files

If Chrome fails to launch:

```bash
rm -f data/x-browser-profile/SingletonLock
rm -f data/x-browser-profile/SingletonSocket
rm -f data/x-browser-profile/SingletonCookie
```

### Check Logs

```bash
# Host logs (relative to project root)
grep -i "x_post\|x_like\|x_reply\|handleXIpc" logs/nanoclaw.log | tail -20

# Script errors
grep -i "error\|failed" logs/nanoclaw.log | tail -20
```

### Script Timeout

Default timeout is 2 minutes (120s). Increase in `host.ts`:

```typescript
const timer = setTimeout(() => {
  proc.kill('SIGTERM');
  resolve({ success: false, message: 'Script timed out (120s)' });
}, 120000);  // ← Increase this value
```

### X UI Selector Changes

If X updates their UI, selectors in scripts may break. Current selectors:

| Element | Selector |
|---------|----------|
| Tweet input | `[data-testid="tweetTextarea_0"]` |
| Post button | `[data-testid="tweetButtonInline"]` |
| Reply button | `[data-testid="reply"]` |
| Like | `[data-testid="like"]` |
| Unlike | `[data-testid="unlike"]` |
| Retweet | `[data-testid="retweet"]` |
| Unretweet | `[data-testid="unretweet"]` |
| Confirm retweet | `[data-testid="retweetConfirm"]` |
| Modal dialog | `[role="dialog"][aria-modal="true"]` |
| Modal submit | `[data-testid="tweetButton"]` |

### Container Build Issues

If MCP tools not found in container:

```bash
# Verify build copies skill
./container/build.sh 2>&1 | grep -i skill

# Check container has the file
docker run nanoclaw-agent ls -la /app/src/skills/
```

## Security

- `data/x-browser-profile/` - Contains X session cookies (in `.gitignore`)
- `data/x-auth.json` - Auth state marker (in `.gitignore`)
- Only main group can use X tools by default (enforced in `agent.ts` and `host.ts`, controlled by `X_MAIN_ONLY`)
- Scripts run as subprocesses with limited environment

## Removal

To remove X integration:

1. Remove import of `handleXIpc` from `src/ipc.ts` and revert the switch default case
2. Remove `createXTools` import and spread from `container/agent-runner/src/ipc-mcp.ts`
3. Remove the `COPY .claude/skills/add-x-twitter-integration/agent.ts` line from `container/Dockerfile`
4. Revert build context changes in `container/build.sh` and `container/Dockerfile` (if no other skills need them)
5. Remove `X_ENABLED_ACTIONS`, `X_REQUIRE_CONFIRMATION`, `X_MAIN_ONLY` from `.env` and `data/env/env`
6. Delete auth data: `rm -rf data/x-browser-profile data/x-auth.json`
7. Rebuild: `npm run build && ./container/build.sh && launchctl kickstart -k gui/$(id -u)/com.nanoclaw` (macOS) or `npm run build && ./container/build.sh && systemctl --user restart nanoclaw` (Linux)
