# Intent: src/ipc.ts modifications

## What changed
Added X (Twitter) integration IPC handler to route `x_*` task types from containers to the host-side browser automation scripts.

## Key sections

### Imports (top of file)
- Added: `handleXIpc` from `../.claude/skills/add-x-twitter-integration/host.js`
- All existing imports remain unchanged

### processTaskIpc() switch statement
- Changed: `default` case now calls `handleXIpc()` before logging unknown type
- The handler returns `true` if the message was an X type (x_post, x_like, etc.), `false` otherwise
- Only logs "Unknown IPC task type" if handleXIpc did not handle the message
- The default case is wrapped in a block (`{ }`) to allow the `const handled` declaration

## Invariants
- All existing switch cases (schedule_task, pause_task, resume_task, cancel_task, refresh_groups, register_group) are completely unchanged
- Message processing (processIpcFiles) is unchanged
- Authorization logic for messages is unchanged
- IPC watcher lifecycle is unchanged
- The handleXIpc function performs its own authorization check (main group only)

## Must-keep
- All existing IpcDeps interface members
- All existing switch cases and their authorization logic
- The ipcWatcherRunning guard
- Error handling and error directory logic
- The processIpcFiles polling loop
