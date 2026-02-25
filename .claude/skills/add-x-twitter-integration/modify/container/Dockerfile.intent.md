# Intent: container/Dockerfile modifications

## What changed
Updated the Docker build context from `container/` to project root, and added workspace directory for X integration IPC results.

## Key sections

### Build context paths
- Changed: `COPY agent-runner/package*.json ./` → `COPY container/agent-runner/package*.json ./`
- Changed: `COPY agent-runner/ ./` → `COPY container/agent-runner/ ./`
- This is required because the build context is now the project root (to allow accessing `.claude/skills/` if needed)

### Workspace directories
- Changed: Added `/workspace/ipc/x_results` to the `mkdir -p` command
- This directory is where X integration results are written by the host and polled by the container's MCP server

## Invariants
- All existing system dependencies are unchanged
- Environment variables (AGENT_BROWSER_EXECUTABLE_PATH, PLAYWRIGHT_CHROMIUM_EXECUTABLE_PATH) unchanged
- Global npm installs unchanged
- TypeScript build step unchanged
- Entrypoint script unchanged
- User permissions unchanged
- All existing workspace directories preserved

## Must-keep
- The complete system dependency list (Chromium, fonts, libs)
- The agent-browser and claude-code global installs
- The entrypoint.sh creation with its stdin/stdout protocol
- The node user setup and permissions
- All existing /workspace/* directories
