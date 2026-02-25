# Intent: container/build.sh modifications

## What changed
Changed the Docker build context from `container/` to the project root directory.

## Key sections

### Build context change
- Changed: `cd "$SCRIPT_DIR"` (container/) → `cd "$SCRIPT_DIR/.."` (project root)
- Changed: `${CONTAINER_RUNTIME} build -t "${IMAGE_NAME}:${TAG}" .` → `${CONTAINER_RUNTIME} build -t "${IMAGE_NAME}:${TAG}" -f container/Dockerfile .`
- This allows the Dockerfile to access `.claude/skills/` and other project-root files during build

## Invariants
- SCRIPT_DIR calculation unchanged
- IMAGE_NAME and TAG defaults unchanged
- CONTAINER_RUNTIME default unchanged
- Test command unchanged
- set -e error handling unchanged

## Must-keep
- The SCRIPT_DIR portability pattern
- CONTAINER_RUNTIME environment variable support
- TAG argument support
- The test command in output
