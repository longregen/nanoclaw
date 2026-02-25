# Intent: container/agent-runner/src/ipc-mcp-stdio.ts modifications

## What changed
Added X (Twitter) integration MCP tools to the container's stdio MCP server. The agent can now call `x_post`, `x_like`, `x_reply`, `x_retweet`, and `x_quote` as MCP tools.

## Key sections

### Constants (top of file)
- Added: `X_RESULTS_DIR` constant pointing to `/workspace/ipc/x_results`
- All existing constants unchanged

### waitForXResult() function
- New async function that polls `X_RESULTS_DIR` for a result file matching a request ID
- Polls every 1 second with a 120-second timeout
- Returns `{ success, message }` from the result file or a timeout error
- Placed after `writeIpcFile` and before the MCP server creation

### X tool registrations (5 new server.tool() calls)
- `x_post`: Post a tweet (max 280 chars), main group only
- `x_like`: Like a tweet by URL, main group only
- `x_reply`: Reply to a tweet (URL + content), main group only
- `x_retweet`: Retweet by URL, main group only
- `x_quote`: Quote tweet (URL + comment), main group only
- Each tool writes an IPC task file to TASKS_DIR with a unique requestId, then polls for the result via waitForXResult
- All tools enforce `isMain` authorization check
- Tools are registered after the existing tools (register_group) and before the transport setup

### IPC flow
1. Tool writes task file to `/workspace/ipc/tasks/` with type `x_post` (etc.) and a `requestId`
2. Host IPC watcher picks up the file, routes to `handleXIpc()` in host.ts
3. Host spawns browser automation script, writes result to `/workspace/ipc/{groupFolder}/x_results/{requestId}.json`
4. MCP tool polls and returns the result

## Invariants
- All existing tools (send_message, schedule_task, list_tasks, pause_task, resume_task, cancel_task, register_group) are completely unchanged
- The `writeIpcFile` function is unchanged and reused by X tools
- Environment variable context (chatJid, groupFolder, isMain) is unchanged
- MCP server name and version unchanged
- Transport setup unchanged

## Must-keep
- All existing server.tool() registrations with their exact descriptions and schemas
- The send_message `sender` parameter (used by Telegram swarm)
- The schedule_task validation logic
- The writeIpcFile atomic write pattern (temp + rename)
- The final transport connection block
