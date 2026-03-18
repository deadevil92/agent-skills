---
name: nanoclaw-dev
description: >
  Expert NanoClaw developer skill. Use this whenever the user wants to build, modify, extend, or customize their NanoClaw installation — including adding new messaging channels, integrations, scheduled tasks, memory behaviors, container features, IPC tools, or any other capability. Trigger this skill any time the user says things like "add X to NanoClaw", "make NanoClaw do Y", "build a NanoClaw skill for Z", "customize my NanoClaw", or asks how to implement anything inside the NanoClaw codebase. This skill gives Claude deep knowledge of NanoClaw's architecture so it can build correctly without guessing.
---

# NanoClaw Developer Skill

You are an expert NanoClaw developer. When the user asks you to build or modify something, read this skill fully, then implement it correctly using the patterns below. Don't guess — NanoClaw has specific conventions and deviating from them breaks things.

## What is NanoClaw?

NanoClaw is a lightweight personal AI agent (~3,900 lines, 15 source files) that:
- Runs Claude agents in **isolated Linux containers** (Docker or Apple Container)
- Connects to messaging apps (WhatsApp, Telegram, Slack, Discord, Gmail)
- Has per-group memory via `CLAUDE.md` files
- Supports scheduled tasks (cron, interval, one-shot)
- Uses **no config files** — behavior changes = code changes

GitHub: https://github.com/qwibitai/nanoclaw  
Full spec: `docs/SPEC.md` in the repo (read it if you need deeper detail on any subsystem)

---

## Architecture at a Glance

```
Channels → SQLite → Message Loop → Group Queue → Container (Claude Agent SDK) → Response
```

**Single Node.js process** on the host. Each agent runs in its own isolated container. IPC between container and host happens via filesystem JSON files.

### Key source files

| File | Role |
|------|------|
| `src/index.ts` | Orchestrator: state, message loop, agent invocation |
| `src/channels/registry.ts` | Channel factory registry — channels self-register here |
| `src/channels/index.ts` | Barrel imports that trigger channel registration |
| `src/types.ts` | `Channel` interface, `ChannelOpts`, message types |
| `src/router.ts` | Routes outbound messages to the right channel |
| `src/ipc.ts` | Processes IPC requests from containers |
| `src/db.ts` | All SQLite queries (messages, groups, sessions, tasks) |
| `src/group-queue.ts` | Per-group FIFO queue with concurrency control |
| `src/container-runner.ts` | Spawns agent containers with mounts |
| `src/task-scheduler.ts` | Runs scheduled tasks when due |
| `src/config.ts` | Constants (ASSISTANT_NAME, paths, timeouts) |
| `container/agent-runner/src/index.ts` | Entry point *inside* the container |
| `container/agent-runner/src/ipc-mcp-stdio.ts` | MCP server inside container for host IPC |

### Folder conventions

```
nanoclaw/
├── src/                        # Host process source
├── container/                  # Code that runs inside containers
│   ├── Dockerfile
│   └── agent-runner/src/
├── .claude/skills/<name>/      # Claude Code skills (slash commands)
├── groups/
│   ├── CLAUDE.md               # Global memory (all groups read this)
│   └── {channel}_{group}/      # Per-group: CLAUDE.md, logs/, agent files
├── store/messages.db           # SQLite
└── data/
    ├── sessions/               # Per-group .claude/ dirs with JSONL transcripts
    ├── env/env                 # Auth env vars for containers
    └── ipc/                    # Container IPC files
```

---

## Core Patterns — Read Before Building Anything

### 1. Adding a New Messaging Channel

Channels self-register at startup. Every channel is a Claude Code skill that adds a TypeScript file.

**Step 1:** Create `src/channels/<name>.ts` implementing the `Channel` interface:

```typescript
import { registerChannel, ChannelOpts } from './registry.js';
import type { Channel } from '../types.js';

export class MyChannel implements Channel {
  name = 'mychannel';

  constructor(private opts: ChannelOpts) {}

  async connect(): Promise<void> { /* connect to API */ }
  async sendMessage(jid: string, text: string): Promise<void> { /* send */ }
  isConnected(): boolean { return /* ... */; }
  ownsJid(jid: string): boolean { return jid.endsWith('@mychannel'); }
  async disconnect(): Promise<void> { /* cleanup */ }

  // Optional:
  async setTyping?(jid: string, isTyping: boolean): Promise<void> {}
  async syncGroups?(force: boolean): Promise<void> {}
}

registerChannel('mychannel', (opts: ChannelOpts) => {
  if (!process.env.MYCHANNEL_TOKEN) return null; // missing credentials → skip
  return new MyChannel(opts);
});
```

**Step 2:** Add import to `src/channels/index.ts`:
```typescript
import './mychannel.js';
```

**Step 3:** Add env var to `.env` and `data/env/env` (or let the skill prompt the user).

**JID convention:** Use a unique suffix per channel (e.g. `@s.whatsapp.net`, `@telegram`, `@slack`) so `ownsJid()` can route correctly.

**Incoming messages:** Call `opts.onMessage({ jid, text, sender, timestamp, ... })` when a message arrives.

### 2. Adding a New Slash Command (Claude Code Skill)

Skills live in `.claude/skills/<command-name>/SKILL.md`. They are Claude Code instructions, not runtime code.

```
.claude/skills/my-command/
└── SKILL.md        # Instructions for Claude Code when user runs /my-command
```

The SKILL.md tells Claude Code what to do step by step — install deps, write files, update registry, prompt user for env vars, restart the service. See existing skills (`/add-telegram`, `/setup`, `/debug`) as templates.

**Skill SKILL.md structure:**
```markdown
# /my-command

Brief description of what this skill does.

## Prerequisites
- NanoClaw installed and running
- [Any other requirements]

## Steps

### 1. Check prerequisites
[What to verify first]

### 2. Install dependencies
```bash
npm install some-package
```

### 3. Create the channel file
[Write src/channels/mychannel.ts with the pattern above]

### 4. Register the channel
[Add import to src/channels/index.ts]

### 5. Configure credentials
[Prompt user, write to .env]

### 6. Register groups
[SQL or orchestrator call to add group to SQLite]

### 7. Rebuild and restart
```bash
npm run build && npm start
```

### 8. Verify
[Test message or log check]
```

### 3. Modifying the Orchestrator / Message Loop

`src/index.ts` is the heart of NanoClaw. Key things to know:

- **Startup order:** container runtime → SQLite init → load state → connect channels → start scheduler + IPC watcher + message loop
- **Message processing:** `processGroupMessages(group)` is called per group; it fetches unprocessed messages from SQLite, builds the prompt (with catch-up context), and calls `container-runner.ts`
- **Trigger pattern:** `TRIGGER_PATTERN` from `src/config.ts` — messages not matching it are stored but not processed
- **Per-group concurrency:** controlled by `GroupQueue` — default max 5 concurrent containers

When adding new behavior to the orchestrator, look for the right hook:
- New startup step → add to the startup sequence after channel connect
- New per-message behavior → modify `processGroupMessages` or the prompt builder
- New background loop → add alongside the scheduler loop

### 4. Working with SQLite (`src/db.ts`)

Tables: `messages`, `registered_groups`, `sessions`, `scheduled_tasks`, `task_run_logs`, `router_state`

Add new queries directly to `db.ts`. Use `better-sqlite3` (synchronous API — no async/await needed):

```typescript
// Example query addition
export function getMyThing(groupFolder: string): MyThing | undefined {
  return db.prepare('SELECT * FROM my_table WHERE group_folder = ?').get(groupFolder) as MyThing | undefined;
}
```

If you add a new table, add the `CREATE TABLE IF NOT EXISTS` statement to the `initializeDatabase()` function.

### 5. IPC — Container ↔ Host Communication

Containers communicate with the host by writing JSON files to `data/ipc/`. The host polls and processes them via `src/ipc.ts`.

**To add a new IPC tool** (a new capability agents can call from inside containers):

1. Add a handler in `src/ipc.ts` — the switch on `request.tool`
2. Expose it as an MCP tool in `container/agent-runner/src/ipc-mcp-stdio.ts`

The MCP tool inside the container calls the host via filesystem IPC. The host executes the action (e.g. sending a message, writing to DB) and writes the response back.

### 6. Scheduled Tasks

Tasks are stored in SQLite `scheduled_tasks` and run by `src/task-scheduler.ts`. Each task runs as a full agent in its group's container.

**Schedule types:**
- `cron` — standard cron expression (e.g. `"0 9 * * 1"` for Mondays 9am)
- `interval` — milliseconds (e.g. `3600000` for hourly)
- `once` — ISO timestamp

Agents create tasks via the `mcp__nanoclaw__schedule_task` MCP tool. You can also insert directly into SQLite if needed.

### 7. Container Mounts

Each container gets these mounts by default:
- `groups/{name}/` → `/workspace/group` (group's working dir)
- `groups/global/` → `/workspace/global` (non-main groups only)
- `data/sessions/{group}/.claude/` → `/home/node/.claude/` (session continuity)
- `data/env/env` → `/workspace/env-dir/env` (auth credentials)

Additional mounts are stored in `registered_groups.container_config` as JSON:
```json
{
  "additionalMounts": [
    { "hostPath": "~/my-folder", "containerPath": "myfolder", "readonly": false }
  ],
  "timeout": 600000
}
```
Additional mounts appear at `/workspace/extra/{containerPath}` inside the container.

**Important:** Paths must be absolute on the host for mounts to work. Use `path.resolve()`.

**Readonly mounts:** Use `--mount "type=bind,source=...,target=...,readonly"` syntax, not `:ro` suffix (compatibility issues).

### 8. Memory System

| File | Location | Who reads it | Who writes it |
|------|----------|--------------|---------------|
| Global memory | `groups/CLAUDE.md` | All groups | Main group only |
| Group memory | `groups/{name}/CLAUDE.md` | That group | That group |
| Agent files | `groups/{name}/*.md` | That group | That group |

The Claude Agent SDK with `settingSources: ['project']` auto-loads CLAUDE.md files from the working directory and parent. Agents write memory by using file tools inside the container.

### 9. Config Constants (`src/config.ts`)

```typescript
export const ASSISTANT_NAME = process.env.ASSISTANT_NAME || 'Andy';
export const POLL_INTERVAL = 2000;                    // message poll ms
export const SCHEDULER_POLL_INTERVAL = 60000;         // task check ms
export const CONTAINER_TIMEOUT = 1800000;             // 30min
export const IDLE_TIMEOUT = 1800000;                  // keep-alive after last result
export const MAX_CONCURRENT_CONTAINERS = 5;
export const TRIGGER_PATTERN = new RegExp(`^@${ASSISTANT_NAME}\\b`, 'i');
```

To add a new config value, add it to `src/config.ts` and reference via import. Prefer env vars for things users might want to change.

---

## Build & Restart

After any source change:
```bash
npm run build    # TypeScript compile → dist/
npm start        # or restart the launchd service
```

For macOS launchd service:
```bash
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist
```

---

## Implementation Checklist

Before writing any code, ask yourself:

1. **Is this a new channel?** → Follow the Channel pattern (interface + registerChannel + barrel import)
2. **Is this a new slash command?** → Create `.claude/skills/<name>/SKILL.md`
3. **Does it need new DB state?** → Add to `db.ts`, add table init if needed
4. **Does it need container access?** → Use IPC pattern (ipc.ts + ipc-mcp-stdio.ts)
5. **Does it touch startup?** → Respect the startup order in index.ts
6. **Does it add credentials?** → Write to `.env` AND `data/env/env` (containers read the latter)
7. **Does it add dependencies?** → `npm install <pkg>` and commit package.json

---

## Common Mistakes to Avoid

- **Don't use relative paths for container mounts** — always `path.resolve()`
- **Don't add features to the core** — new capabilities go in skills (`.claude/skills/`)
- **Don't forget `data/env/env`** — containers don't read `.env` directly
- **Don't use async better-sqlite3** — it's synchronous by design
- **Don't hardcode the assistant name** — always use `ASSISTANT_NAME` from config
- **Don't break existing channels** — new channel files must return `null` from factory if credentials are missing
- **Readonly mounts:** use `--mount` syntax, not `:ro`

---

## Reference Files

For deeper detail on specific subsystems, read these files in the NanoClaw repo:
- `docs/SPEC.md` — Full architecture specification (785 lines, very detailed)
- `docs/SECURITY.md` — Security model and threat model
- `src/types.ts` — All TypeScript interfaces
- Existing skill examples: `.claude/skills/add-telegram/`, `.claude/skills/setup/`
