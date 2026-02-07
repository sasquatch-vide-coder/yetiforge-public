# YETIFORGE

Telegram bot framework that bridges messages to Claude Code CLI.

---

## Agent Behavior

### Working Style
- Orchestrator pattern: Chat agent stays responsive and conversational, delegates ALL work to the orchestrator pipeline
- For ALL real work (code changes, file operations, research, debugging, git operations, running commands), emit a `<YETIFORGE_ACTION>` block
- The only things the chat agent does directly: casual conversation, answering questions from knowledge, and formulating YETIFORGE_ACTION blocks
- Always commit and push changes when a feature is complete
- Update this CLAUDE.md when architecture changes
- Keep context windows small by using sub-agents for heavy lifting

## Tech Stack
- TypeScript ES modules, Node.js v22+
- grammY for Telegram
- Claude Code CLI spawned via child_process
- JSON file persistence in data/
- Fastify status/dashboard server (React + Vite + Tailwind)

## Commands
- `npm run dev` - Run with tsx (development)
- `npm run build` - Compile TypeScript
- `npm start` - Run compiled JS (production)
- `npm run build:client` - Build status page frontend
- `npm run build:all` - Build server + client

## Architecture
Telegram messages → grammY bot (always running) → `claude -p` spawned per message → response sent back.
Sessions are resumed via `--resume <sessionId>` for conversation continuity.

### Status Page
- **Server**: Fastify on port 3069 (`src/status/server.ts`), started alongside the bot
- **Client**: React + Vite + Tailwind in `status/client/`
- **Style**: Neo Brutalist design
- **Proxy**: Nginx reverse proxy on ports 80/443 with Let's Encrypt SSL
- **API Endpoints**:
  - `GET /api/status` - Service health, system info, projects
  - `GET /api/invocations` - Historical invocation data (cost, tokens, duration)
  - `GET /api/health` - Health check
- **Invocation logging**: Claude CLI results logged to `data/invocations.json` for historical metrics
- **Privacy**: No logs or session details exposed on the public dashboard

### Admin Panel
- **Auth**: JWT-based with optional TOTP MFA (`src/admin/auth.ts`)
- **Routes**: `/api/admin/*` endpoints (`src/admin/routes.ts`)
- **Frontend**: React pages at `/admin` (login) and `/admin/dashboard` (protected)
- **Panels**: Claude Code status, Telegram status, SSL/TLS management, Security (MFA + password)
- **Data**: Admin credentials stored in `data/admin.json`
- **First-time setup**: Visit `/admin` — creates admin account on first use
- **Config**: `ADMIN_JWT_SECRET` in `.env`

## Deployment

### As a systemd Service
```bash
sudo cp yetiforge.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable yetiforge
sudo systemctl start yetiforge
```

### Deploy Steps
```bash
npm install && npm run build
cd status/client && npm install && npm run build && cd ../..
sudo cp yetiforge.service /etc/systemd/system/yetiforge.service
sudo systemctl daemon-reload && sudo systemctl restart yetiforge
```

### Logs
```bash
sudo journalctl -u yetiforge -f
```

## GitHub
- **Repo**: https://github.com/sasquatch-vide-coder/yetiforge
