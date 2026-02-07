# YetiForge

A Telegram bot framework powered by Claude Code CLI. Two-tier agent architecture — a fast chat agent for conversation and decision-making, and a powerful executor for planning and executing code tasks with a mandatory plan-approve-execute workflow.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Telegram User                            │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │      grammY Bot     │
                    │  auth + rate-limit  │
                    │    middleware        │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │     Chat Agent      │  ◄── Haiku (fast, 30s timeout)
                    │  personality layer  │
                    │  intent detection   │
                    └────┬──────────┬─────┘
                         │          │
                   Just chat    Work request
                         │          │
              ┌──────────▼──┐  ┌────▼───────────────────┐
              │   Respond   │  │  Executor — PLAN mode  │  ◄── Opus (read-only tools)
              │   directly  │  │  investigate + propose  │
              └─────────────┘  └────────────┬───────────┘
                                            │
                                   Plan shown to user
                                   "Approve / revise / cancel?"
                                            │
                               ┌────────────▼───────────┐
                               │ Executor — EXEC mode   │  ◄── Opus (full tool access)
                               │ edit, write, bash, etc  │
                               └────────────┬───────────┘
                                            │
                               ┌────────────▼───────────┐
                               │     Claude Code CLI    │
                               │   (spawned subprocess) │
                               └────────────────────────┘
```

### Two-Tier Agent System

| Agent | Model | Purpose | Tools | Timeout |
|-------|-------|---------|-------|---------|
| **Chat Agent** | Haiku | Fast responses, intent classification, memory detection. Decides if a message is casual chat or a work request. | None (conversation only) | 30s |
| **Executor** | Opus | Task planning and execution. Runs in two modes — PLAN (read-only investigation) and EXECUTE (full access). | PLAN: Read, Grep, Glob, WebFetch, WebSearch, Task. EXECUTE: all tools including Edit, Write, Bash. | 5–45 min (complexity-based) |

### Message Flow

1. **User sends message** in Telegram
2. **Auth middleware** validates against user allowlist
3. **Rate-limit middleware** ensures one request per chat at a time
4. **Chat Agent** (Haiku) receives message + memory context + any pending plan context
5. Chat Agent responds with text and optionally emits action blocks:
   - No action → chat response sent, done
   - `work_request` → enter plan phase
   - `approve_plan` → execute approved plan
   - `revise_plan` → re-plan with user feedback
   - `cancel_plan` → discard pending plan
6. **Executor PLAN mode** — read-only investigation, returns a proposed plan
7. **Plan presented to user** — "Approve, request changes, or cancel?"
8. **User approves** → plan consumed from store
9. **Executor EXECUTE mode** — full tool access, complexity-based timeout
10. **Progress panel** updates every 4 seconds with tool activity (reads, edits, writes, commands)
11. **Result delivered** — summary with duration and cost
12. **Queue check** — if tasks are queued, next one auto-starts
13. **Crash recovery** — result persisted to disk before Telegram delivery

## Features

### Task Queue
- Max 5 tasks per chat, persisted to disk (`data/task-queue.json`)
- If executor is busy, new work requests are queued instead of rejected
- After each execution completes, the next queued task auto-starts
- Queued tasks go through the full plan→approval cycle
- Survives restarts — `/resume queue` to continue processing

### Plan-Approve-Execute Workflow
- Every work request produces a read-only plan before any changes are made
- Plans stored in-memory per chat (PlanStore) — one pending plan at a time
- User can approve, request revisions (re-plan with feedback), or cancel
- Revision count tracked — previous plan + feedback included in re-planning prompt
- No code changes happen without explicit user approval

### Persistent Memory
- Auto-detected by Chat Agent via `<YETIFORGE_MEMORY>` blocks in responses
- Manual management via `/memory` commands
- Stored per-user in `data/memory.json`, max 50 notes
- Top 20 recent notes injected into agent context as `[MEMORY CONTEXT]`
- Deduplication prevents saving identical notes

### Cron Scheduling
- Standard cron expressions via `node-cron` (`"0 9 * * *"` = daily at 9am)
- Persisted to `data/cron-jobs.json`
- Jobs can be enabled/disabled, run immediately, or removed
- Execution routed through the same executor pipeline
- Results sent to Telegram with last-run status tracking

### Webhooks
- External systems trigger tasks via HTTP POST
- Each webhook has a unique 24-byte hex secret
- Optional payload injection into task context
- Persisted to `data/webhooks.json`
- Route: `POST /webhook/<webhook-id>`

### Crash Recovery
- **Active Task Tracker** — persists running tasks to `data/active-tasks.json` (sync writes). On restart, interrupted tasks detected and user notified.
- **Pending Responses** — results persisted to `data/pending-responses.json` before Telegram delivery. Recovered on startup.
- **Queue Persistence** — queued tasks survive restarts in `data/task-queue.json`.
- **Resume** — `/resume` shows interrupted tasks, `/resume <number>` continues with session ID, `/resume clear` dismisses.

### Stall Detection
- Monitors executor output silence duration
- Complexity-aware thresholds (trivial tasks fail faster)
- Three stages: warning → grace period (1.5x kill threshold) → hard abort
- User notified at each stage

### Session Management
- Per-user, per-scope (chat vs executor) conversation context
- Auto-rotation after 15 invocations to prevent unbounded context growth
- Persisted to `data/sessions.json`

### Progress Panels
- StreamFormatter renders live Telegram messages during execution
- Updates every 4 seconds with categorized tool activity
- Icons: `📂` read, `✏️` edit, `📝` write, `▶️` command, `⚠️` warning, `❌` error
- Smart truncation for Telegram's 4096-char limit

### Multi-Project Support
- Register multiple codebases with `/project add`
- Switch working directories on the fly
- Persisted to `data/projects.json`

### Git Operations
- `/git status` — working tree status
- `/git commit <message>` — stage and commit all changes
- `/git push [remote] [branch]` — push to remote
- `/git pr [title]` — create GitHub PR (requires `GITHUB_PAT`)

### Admin Dashboard
- **UI**: Neo Brutalist design — React + Vite + Tailwind CSS
- **Auth**: JWT + optional TOTP MFA, bcrypt password hashing, IP whitelist
- **Metrics**: Per-invocation token usage, cost, duration (SQLite backend)
- **Agent Config**: Change models, timeouts, stall thresholds per agent tier
- **Bot Config**: Edit bot name, view Telegram settings
- **Claude CLI**: Check version and installation status
- **Web Chat**: Talk to the bot from the browser
- **SSL/TLS**: Certificate status monitoring
- **Sessions**: View and revoke active admin sessions
- **Audit Logging**: Admin actions logged
- **Backups**: Data backup management
- **Config History**: Track configuration changes
- **Alerts**: Alert notification system

## Bot Commands

| Command | Description |
|---------|-------------|
| `/start` | Introduction and welcome message |
| `/help` | Show all available commands |
| `/status` | Session info, project, executor state, queue length, memory notes, cron jobs |
| `/reset` | Clear conversation history |
| `/cancel` | Abort current running task |
| `/model` | Show agent model configuration and timeouts |
| `/compact` | Summarize session history and clear old context |
| `/project list\|add\|switch\|remove` | Multi-project management |
| `/git status\|commit\|push\|pr` | Git operations |
| `/queue list\|cancel\|clear` | Task queue management |
| `/resume [number\|clear\|queue]` | Crash recovery — view, continue, or dismiss interrupted tasks |
| `/memory list\|add\|remove\|clear` | Persistent memory management |
| `/cron list\|add\|remove\|run\|enable\|disable` | Scheduled task management |
| `/webhook list\|create\|remove` | External webhook triggers |

## Tech Stack

| Category | Technology | Version |
|----------|-----------|---------|
| Runtime | Node.js | 22+ |
| Language | TypeScript | 5.7 |
| Bot Framework | grammY | 1.35 |
| Web Server | Fastify | 5.7 |
| Database | better-sqlite3 (metrics) | 12.6 |
| Auth | jsonwebtoken + bcrypt + otpauth | 9.0 / 6.0 / 9.5 |
| Scheduling | node-cron | 4.2 |
| Logging | Pino | 9.6 |
| Environment | dotenv | 16.4 |
| QR Codes | qrcode (TOTP setup) | 1.5 |
| Frontend | React + Vite + Tailwind CSS | — |
| AI Backend | Claude Code CLI | subprocess |
| Persistence | JSON files + SQLite | — |
| Deployment | systemd + Nginx | Ubuntu |

## Project Structure

```
src/
├── index.ts                  # App entry — manager init, startup, shutdown
├── bot.ts                    # Bot creation, middleware, command registration
├── config.ts                 # Environment variable loading
├── bot-config-manager.ts     # Bot name configuration
├── plan-store.ts             # In-memory pending plan storage
├── task-queue.ts             # Per-chat task queue (persistent)
├── memory-manager.ts         # Persistent user memory notes
├── cron-manager.ts           # Scheduled task management
├── webhook-manager.ts        # External trigger webhooks
├── active-task-tracker.ts    # Crash recovery — active task persistence
├── pending-responses.ts      # Crash recovery — unsent message persistence
│
├── agents/
│   ├── chat-agent.ts         # Tier 1: Haiku — conversation + intent detection
│   ├── executor.ts           # Tier 2: Opus — plan mode + execute mode
│   ├── prompts.ts            # System prompts for both tiers
│   ├── types.ts              # Shared interfaces (WorkRequest, ChatAction, ExecutorResult)
│   ├── agent-config.ts       # Model and timeout configuration
│   └── agent-registry.ts     # Central registry of active agents
│
├── handlers/
│   ├── message.ts            # Message pipeline — plan/execute workflows, queue processing
│   ├── commands.ts           # All slash command implementations
│   └── media.ts              # Image/photo handling (inactive)
│
├── claude/
│   ├── invoker.ts            # Claude CLI subprocess spawning + NDJSON parsing
│   └── session-manager.ts    # Per-user conversation context management
│
├── admin/
│   ├── auth.ts               # JWT + TOTP MFA authentication
│   ├── routes.ts             # Admin API endpoints
│   ├── audit-logger.ts       # Admin action logging
│   ├── alert-manager.ts      # Alert notifications
│   ├── backup-manager.ts     # Data backup management
│   ├── config-history.ts     # Configuration change tracking
│   ├── web-chat-store.ts     # Web chat message history
│   └── rate-limiter.ts       # Login rate limiting
│
├── status/
│   ├── server.ts             # Fastify server — routes, webhooks, static files
│   ├── database.ts           # SQLite persistence for invocation metrics
│   ├── invocation-logger.ts  # Per-invocation cost/token/duration logging
│   ├── metrics-collector.ts  # Real-time metric collection
│   └── agent-routes.ts       # Agent monitoring endpoints
│
├── middleware/
│   ├── auth.ts               # Telegram user allowlist
│   └── rate-limit.ts         # Per-chat mutual exclusion + executor busy tracking
│
├── projects/
│   └── project-manager.ts    # Multi-project directory management
│
└── utils/
    ├── logger.ts             # Pino logging setup
    ├── stream-formatter.ts   # Progress panel formatting for Telegram
    └── telegram.ts           # Typing indicators, message send/edit helpers

status/
└── client/                   # React admin dashboard (Vite + Tailwind)

data/                         # Persistent storage (JSON + SQLite)
docs/                         # Personality configuration
setup/                        # Installer modules (banner, preflight, deps, config, build, services, ssl, finalize)
```

## Setup

### Prerequisites

- **Node.js 22+**
- **Claude Code CLI** installed and authenticated — `npm install -g @anthropic-ai/claude-code && claude auth`
- **Telegram bot token** from [@BotFather](https://t.me/BotFather)

### One-Command Install

```bash
git clone https://github.com/sasquatch-vide-coder/yetiforge.git
cd yetiforge && bash install.sh
```

The installer runs an interactive wizard that handles Node.js, Nginx, `.env` configuration, building, systemd service setup, and optional SSL.

### Manual Installation

```bash
git clone https://github.com/sasquatch-vide-coder/yetiforge.git
cd yetiforge
npm install
cd status/client && npm install && cd ../..
```

### Configuration

Create `.env` from the template:

```bash
cp .env.example .env
```

Required variables:

```env
TELEGRAM_BOT_TOKEN=           # From @BotFather
ALLOWED_USER_IDS=123456789    # Comma-separated Telegram user IDs
DEFAULT_PROJECT_DIR=/path/to/projects
ADMIN_JWT_SECRET=             # openssl rand -hex 32
```

Optional variables:

```env
CLAUDE_CLI_PATH=claude        # Path to Claude CLI binary (default: claude)
CLAUDE_TIMEOUT_MS=300000      # 5 min default
DATA_DIR=./data               # Data storage directory
STATUS_PORT=3069              # Admin dashboard port
STATUS_HOST=your-domain.com   # Public hostname
WEBHOOK_HOST=your-domain.com  # Webhook callback domain
GITHUB_PAT=ghp_...           # GitHub token for /git pr
LOG_LEVEL=info                # Pino log level
```

### Build & Run

```bash
# Build everything (TypeScript + React dashboard)
npm run build:all

# Start
npm start

# Or dev mode with hot reload
npm run dev
```

### Deploy as a Service

```bash
sudo cp yetiforge.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable yetiforge
sudo systemctl start yetiforge

# Check logs
sudo journalctl -u yetiforge -f
```

### Reverse Proxy (Nginx)

The admin dashboard and webhook endpoints run on port 3069. Point Nginx at it for HTTPS:

```nginx
server {
    server_name your-domain.com;

    location / {
        proxy_pass http://127.0.0.1:3069;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## Data Persistence

All state is stored in the `data/` directory:

| File | Purpose |
|------|---------|
| `sessions.json` | Conversation context per user/scope |
| `projects.json` | Registered project directories |
| `memory.json` | Persistent user memory notes |
| `cron-jobs.json` | Scheduled task definitions |
| `webhooks.json` | Webhook configurations |
| `task-queue.json` | Queued tasks (survives restarts) |
| `active-tasks.json` | Running tasks for crash recovery |
| `pending-responses.json` | Unsent messages for crash recovery |
| `admin.json` | Admin credentials |
| `admin-sessions.json` | Active admin sessions |
| `agent-config.json` | Agent model/timeout configuration |
| `bot-config.json` | Bot name |
| `invocations.db` | SQLite — invocation metrics, costs, tokens |

## Startup Sequence

1. Load environment config
2. Initialize all managers (session, project, invocation, agent config, bot config, memory, cron, webhook, active task tracker, task queue, pending responses)
3. Load personality from `docs/personality.md`
4. Create Chat Agent (Haiku) and Executor (Opus)
5. Create bot with middleware and command handlers
6. Set up cron and webhook trigger handlers (route through executor pipeline)
7. Start Fastify server on `STATUS_PORT`
8. Start grammY bot
9. On startup callback:
   - Recover and deliver pending responses
   - Detect and notify about interrupted tasks
   - Detect and notify about queued tasks
   - Start all enabled cron jobs
10. Graceful shutdown on SIGINT/SIGTERM — stop cron, stop bot, close server, save all managers

## License

MIT

---

*Built with Claude Code.*
