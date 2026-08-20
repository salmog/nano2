# nano2 — NanoClaw + Claude Code Integration

Personal NanoClaw agent setup running on macOS, powered by Claude Code (claude-sonnet-4-6, 1M context).

---

## Project Integration: avri-nanoclaw + Claude Code + pnpm

### What it is

NanoClaw is a personal Claude assistant host. It runs as a macOS launchd service, spawns per-session Docker containers for each agent group, and routes messages from channels (CLI, Matrix, WhatsApp, Slack, etc.) through to Claude via the Anthropic Agent SDK.

**Stack:**
- **Host**: Node.js + pnpm (TypeScript, compiled via `tsc`)
- **Agent containers**: Bun runtime (separate package tree under `container/agent-runner/`)
- **Communication**: Two SQLite DBs per session (`inbound.db` / `outbound.db`) — no IPC or stdin piping
- **Credentials**: OneCLI gateway (secrets injected at request time, never in env vars)

### Key pnpm Commands (run from nanoclaw-main/)

```bash
# Development
pnpm run dev          # Start host with hot reload (tsx watch)
pnpm run build        # Compile TypeScript (src/ → dist/)
pnpm test             # Run host tests (vitest)

# Container
./container/build.sh  # Rebuild agent container image (nanoclaw-agent:latest)

# Admin CLI
pnpm exec ncl groups list              # List agent groups
pnpm exec ncl groups config get --id <id>    # Show container config for a group
pnpm exec ncl groups config update --id <id> --model claude-sonnet-5  # Change model
pnpm exec ncl groups restart --id <id>       # Restart agent containers for a group
pnpm exec ncl sessions list            # List active sessions
pnpm exec ncl roles list               # List user roles (owner/admin)

# Ad-hoc DB queries
pnpm exec tsx scripts/q.ts data/v2.db "<SQL>"   # Query central DB
```

### Agent Container (Bun — separate tree)

```bash
cd container/agent-runner
bun install          # Install container deps (NOT pnpm install)
bun test             # Run container tests
bun run typecheck    # Type-check container TypeScript
```

> **Important:** `container/agent-runner/` is a Bun workspace, not a pnpm workspace. Never run `pnpm install` there.

### Service Management (macOS)

```bash
launchctl load   ~/Library/LaunchAgents/com.nanoclaw.plist   # Start
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist   # Stop
launchctl kickstart -k gui/$(id -u)/com.nanoclaw             # Restart
```

### SSH / Git Setup (this repo)

Remote: `git@github-nano2:salmog/nano2.git`
Key: `~/.ssh/nano2` via `Host github-nano2` alias in `~/.ssh/config`

```
Host github-nano2
    HostName github.com
    User git
    IdentityFile ~/.ssh/nano2
    IdentitiesOnly yes
```

---

## How to Uninstall NanoClaw Completely (macOS)

### Option A — Built-in uninstall script (recommended)

```bash
cd /path/to/nanoclaw-main
bash uninstall.sh --dry-run   # Preview what will be removed
bash uninstall.sh             # Run interactively (confirms per group)
bash uninstall.sh --yes       # Skip all prompts
```

Removes: launchd service, Docker containers + image, `data/`, `logs/`, `groups/`, OneCLI agents for this install. Other NanoClaw installs and the shared OneCLI app are untouched.

### Option B — Manual steps

**1. Stop and remove launchd services:**
```bash
launchctl unload ~/Library/LaunchAgents/com.nanoclaw-v2-*.plist
launchctl unload ~/Library/LaunchAgents/com.nanoclaw-matrix-tunnel.plist
rm ~/Library/LaunchAgents/com.nanoclaw*.plist
```

**2. Remove Docker containers and image:**
```bash
docker ps -a --filter "name=nanoclaw" -q | xargs docker rm -f
docker images nanoclaw-agent -q | xargs docker rmi -f
```

**3. Delete the install directory and all data:**
```bash
rm -rf /path/to/nanoclaw-main
```

**4. (Optional) Remove OneCLI agents:**
```bash
onecli agents list
onecli agents delete --id <agent-id>   # repeat for each nanoclaw agent
```

**5. (Optional) Remove Claude Code project memory:**
```bash
rm -rf ~/.claude/projects/<nanoclaw-project-slug>/
```

---

## Logs & Troubleshooting

| What | Where |
|------|-------|
| Host errors | `logs/nanoclaw.error.log` |
| Full host log | `logs/nanoclaw.log` |
| Setup logs | `logs/setup.log`, `logs/setup-steps/*.log` |
| Session DBs | `data/v2-sessions/<group>/<session>/inbound.db` + `outbound.db` |

> Container logs are lost after exit (`--rm` flag). If an agent silently fails, check `outbound.db` first.
