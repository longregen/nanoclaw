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

Run the skills engine to apply this skill's code package. The package files are in this directory alongside this SKILL.md.

### Initialize skills system (if needed)

If `.nanoclaw/` directory doesn't exist yet:

```bash
npx tsx scripts/apply-skill.ts --init
```

### Apply the skill

```bash
npx tsx scripts/apply-skill.ts .claude/skills/add-x-twitter-integration
```

This deterministically:
- Three-way merges X IPC handler into `src/ipc.ts` (adds `handleXIpc` import and switch default routing)
- Three-way merges X MCP tools into `container/agent-runner/src/ipc-mcp-stdio.ts` (registers `x_post`, `x_like`, `x_reply`, `x_retweet`, `x_quote` tools)
- Three-way merges build context change into `container/build.sh` (project root context)
- Three-way merges Dockerfile changes into `container/Dockerfile` (build context paths, x_results workspace directory)
- Installs `playwright` and `dotenv-cli` npm dependencies
- Updates `.env.example` with `CHROME_PATH`, `X_ENABLED_ACTIONS`, `X_REQUIRE_CONFIRMATION`, `X_MAIN_ONLY`
- Records the application in `.nanoclaw/state.yaml`

If the apply reports merge conflicts, read the intent files:
- `modify/src/ipc.ts.intent.md` — what changed and invariants for ipc.ts
- `modify/container/agent-runner/src/ipc-mcp-stdio.ts.intent.md` — what changed for the MCP server
- `modify/container/Dockerfile.intent.md` — what changed for the Dockerfile
- `modify/container/build.sh.intent.md` — what changed for the build script

### Configure environment

Add to `.env` based on user's choices:

```bash
CHROME_PATH=/path/to/Google Chrome
X_ENABLED_ACTIONS=post,like,reply,retweet,quote
X_REQUIRE_CONFIRMATION=true
X_MAIN_ONLY=true
```

Check `CHROME_PATH`:

```bash
# macOS:
mdfind "kMDItemCFBundleIdentifier == 'com.google.Chrome'" 2>/dev/null | head -1
# Linux:
which google-chrome || which chromium-browser
```

Sync to container environment:

```bash
mkdir -p data/env && cp .env data/env/env
```

### Validate code changes

```bash
npm test
npm run build
```

All tests must pass and build must be clean before proceeding.

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
│  └── ipc-mcp-stdio.ts → MCP tools (x_post, etc.)          │
│      └── Writes IPC task to /workspace/ipc/tasks/          │
│      └── Polls /workspace/ipc/x_results/ for response      │
└──────────────────────┬──────────────────────────────────────┘
                       │ IPC (file system)
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  Host                                                       │
│  └── src/ipc.ts → processTaskIpc()                         │
│      └── host.ts → handleXIpc()                            │
│          └── spawn subprocess → scripts/*.ts               │
│              └── Playwright → Chrome → X Website           │
│          └── Write result → ipc/{group}/x_results/         │
└─────────────────────────────────────────────────────────────┘
```

### Why This Design?

- **API is expensive** - X official API requires paid subscription ($100+/month) for posting
- **Bot browsers get blocked** - X detects and bans headless browsers and common automation fingerprints
- **Must use user's real browser** - Reuses the user's actual Chrome on Host with real browser fingerprint to avoid detection
- **One-time authorization** - User logs in manually once, session persists in Chrome profile for future use
- **MCP integration** - X tools are registered as MCP tools in the container's stdio server, available as `mcp__nanoclaw__x_post`, etc.

### File Structure

```
.claude/skills/add-x-twitter-integration/
├── manifest.yaml     # Skill package manifest
├── SKILL.md          # This documentation
├── host.ts           # Host-side IPC handler
├── agent.ts          # Reference: tool definitions (Agent SDK format)
├── lib/
│   ├── config.ts     # Centralized configuration
│   └── browser.ts    # Playwright utilities
├── scripts/
│   ├── setup.ts      # Interactive login
│   ├── post.ts       # Post tweet
│   ├── like.ts       # Like tweet
│   ├── reply.ts      # Reply to tweet
│   ├── retweet.ts    # Retweet
│   └── quote.ts      # Quote tweet
└── modify/
    ├── src/
    │   ├── ipc.ts                # Host IPC with X handler
    │   └── ipc.ts.intent.md      # Intent documentation
    └── container/
        ├── agent-runner/src/
        │   ├── ipc-mcp-stdio.ts          # MCP server with X tools
        │   └── ipc-mcp-stdio.ts.intent.md
        ├── Dockerfile            # Container with x_results dir
        ├── Dockerfile.intent.md
        ├── build.sh              # Build from project root
        └── build.sh.intent.md
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

If X MCP tools not available in container:

```bash
# Rebuild container (ensure ipc-mcp-stdio.ts has X tool registrations)
./container/build.sh

# Verify tools registered by checking MCP server source
grep -c "x_post" container/agent-runner/src/ipc-mcp-stdio.ts
```

## Security

- `data/x-browser-profile/` - Contains X session cookies (in `.gitignore`)
- `data/x-auth.json` - Auth state marker (in `.gitignore`)
- Only main group can use X tools by default (enforced in MCP tools and `host.ts`, controlled by `X_MAIN_ONLY`)
- Scripts run as subprocesses with limited environment

## Removal

To remove X integration:

1. Revert `src/ipc.ts`: remove `handleXIpc` import and revert the switch default case to just log unknown types
2. Revert `container/agent-runner/src/ipc-mcp-stdio.ts`: remove `X_RESULTS_DIR` constant, `waitForXResult` function, and all `x_*` tool registrations
3. Revert `container/Dockerfile`: remove `/workspace/ipc/x_results` from mkdir, revert build context paths if no other skills need them
4. Revert `container/build.sh`: revert to `cd "$SCRIPT_DIR"` and `build -t "${IMAGE_NAME}:${TAG}" .` if no other skills need project root context
5. Remove `CHROME_PATH`, `X_ENABLED_ACTIONS`, `X_REQUIRE_CONFIRMATION`, `X_MAIN_ONLY` from `.env` and `data/env/env`
6. Delete auth data: `rm -rf data/x-browser-profile data/x-auth.json`
7. Remove `x-twitter-integration` from `.nanoclaw/state.yaml` applied_skills
8. Rebuild: `npm run build && ./container/build.sh && launchctl kickstart -k gui/$(id -u)/com.nanoclaw` (macOS) or `npm run build && ./container/build.sh && systemctl --user restart nanoclaw` (Linux)
