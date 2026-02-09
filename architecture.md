# Gastown UI — Architecture & Project Setup

> Technical architecture for building a modern web UI layer on top of
> [steveyegge/gastown](https://github.com/steveyegge/gastown), the multi-agent
> orchestration system for Claude Code.

---

## Table of Contents

1. [Existing Foundation](#1-existing-foundation)
2. [Problem Statement](#2-problem-statement)
3. [Target Architecture](#3-target-architecture)
4. [Technical Requirements](#4-technical-requirements)
5. [Frontend Architecture](#5-frontend-architecture)
6. [Backend Enhancements](#6-backend-enhancements)
7. [Data Model & API Contract](#7-data-model--api-contract)
8. [Real-Time Communication](#8-real-time-communication)
9. [Claude Code Setup & Configuration](#9-claude-code-setup--configuration)
10. [GitHub Gitflow & Repository Setup](#10-github-gitflow--repository-setup)
11. [CI/CD Pipeline](#11-cicd-pipeline)
12. [Security Considerations](#12-security-considerations)
13. [Testing Strategy](#13-testing-strategy)
14. [Deployment & Distribution](#14-deployment--distribution)

---

## 1. Existing Foundation

Gastown already ships a web dashboard (`gt dashboard --port 8080`). Understanding
what exists is critical before designing what comes next.

### What's Already Built

| Layer | Implementation | Location |
|-------|---------------|----------|
| HTTP Server | Go `net/http`, no framework | `internal/web/handler.go` |
| REST API | 12+ endpoints (convoy, mail, issues, PR, crew, commands) | `internal/web/api.go` |
| Frontend | Vanilla JS (62KB) + CSS (57KB) + HTMX | `internal/web/static/` |
| HTML Template | Single Go `html/template` (44KB), 15+ panels | `internal/web/templates/convoy.html` |
| Data Fetching | 14 parallel goroutine fetchers with timeouts | `internal/web/fetcher.go` |
| Command Exec | Whitelisted command palette (~40 commands) | `internal/web/commands.go` |
| Setup Wizard | Workspace creation/config UI | `internal/web/setup.go` |
| Terminal UI | Bubbletea-based convoy & feed TUIs | `internal/tui/` |
| Event System | JSONL audit log with structured events | `internal/events/` |

### Current Limitations

- **No WebSockets** — relies on HTMX 10-second polling (`hx-trigger="every 10s"`)
- **No authentication** — CORS set to `*`, no auth middleware
- **No frontend build system** — raw static files, no bundling or TypeScript
- **Monolithic template** — single 44KB HTML file with all 15+ panels
- **No real-time streaming** — agent output not streamed to browser
- **Data freshness** — fetcher timeouts of 2–15s per source, 30s option cache
- **Single-machine only** — dashboard shells out to `gt`, `bd`, `gh`, `tmux` locally

---

## 2. Problem Statement

Gastown's value scales with agent count (20–30+ concurrent Polecats), but the
current UI has scaling gaps:

1. **Visibility** — 10-second polling misses transient state; agent output is
   invisible from the dashboard
2. **Interactivity** — command palette covers basic ops but lacks drag-and-drop
   work assignment, inline editing, and contextual actions
3. **Extensibility** — monolithic template + vanilla JS makes feature additions
   costly and fragile
4. **Collaboration** — no multi-user support; single browser tab assumed
5. **Mobile/responsive** — CSS has breakpoints but the information density
   doesn't adapt well to small screens

### Design Goal

Build a **modern, component-based frontend** that connects to Gastown's existing
Go backend via an enhanced API layer with real-time event streaming — without
rewriting the backend.

---

## 3. Target Architecture

```
┌────────────────────────────────────────────────────────────┐
│                        Browser                             │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              React/Solid SPA (TypeScript)            │  │
│  │                                                      │  │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌──────────────┐  │  │
│  │  │Dashboard│ │ Kanban │ │Timeline│ │ Agent Detail  │  │  │
│  │  │  Grid   │ │ Board  │ │  View  │ │  + Live Tail │  │  │
│  │  └───┬────┘ └───┬────┘ └───┬────┘ └──────┬───────┘  │  │
│  │      │          │          │              │           │  │
│  │  ┌───▼──────────▼──────────▼──────────────▼───────┐  │  │
│  │  │         State Management (TanStack Query)      │  │  │
│  │  └───────────────────┬────────────────────────────┘  │  │
│  └──────────────────────┼───────────────────────────────┘  │
│                         │                                  │
│              WebSocket + REST                              │
└─────────────────────────┼──────────────────────────────────┘
                          │
┌─────────────────────────┼──────────────────────────────────┐
│                    Go Backend                              │
│                         │                                  │
│  ┌──────────────────────▼───────────────────────────────┐  │
│  │              HTTP/WS Router (net/http)                │  │
│  │                                                      │  │
│  │  ┌──────────┐  ┌──────────┐  ┌────────────────────┐  │  │
│  │  │ REST API │  │WebSocket │  │  Static File       │  │  │
│  │  │ (existing│  │  Hub     │  │  Server (embed.FS)  │  │  │
│  │  │ +enhanced)│ │ (new)    │  │  (new SPA assets)  │  │  │
│  │  └────┬─────┘  └────┬────┘  └────────────────────┘  │  │
│  │       │              │                               │  │
│  │  ┌────▼──────────────▼────────────────────────────┐  │  │
│  │  │          Data Layer                            │  │  │
│  │  │                                                │  │  │
│  │  │  ┌─────────┐ ┌────────┐ ┌────────┐ ┌───────┐  │  │  │
│  │  │  │Fetcher  │ │Events  │ │Config  │ │Tmux   │  │  │  │
│  │  │  │(existing)│ │(JSONL) │ │(JSON)  │ │Bridge │  │  │  │
│  │  │  └─────────┘ └────────┘ └────────┘ └───────┘  │  │  │
│  │  └────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Gastown Core (unchanged)                │  │
│  │  Mayor · Polecats · Hooks · Convoys · Beads · Mail   │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

### Key Principles

- **Additive, not rewrite** — enhance the existing Go backend; don't replace it
- **Gastown-native** — the UI reads the same git-backed state; no separate DB
- **Embeddable** — compiled SPA assets embedded via Go `embed.FS`, served by `gt dashboard`
- **Offline-capable** — works on localhost; no cloud dependency

---

## 4. Technical Requirements

### Runtime Dependencies

| Dependency | Version | Purpose |
|-----------|---------|---------|
| Go | >= 1.24 | Backend (matches Gastown) |
| Node.js | >= 20 LTS | Frontend build toolchain |
| pnpm | >= 9 | Frontend package management |
| Git | >= 2.25 | Worktree support (Gastown requirement) |
| tmux | >= 3.0 | Agent session management (Gastown requirement) |
| Beads | >= 0.47 | Issue tracking integration |

### Frontend Stack

| Technology | Purpose | Rationale |
|-----------|---------|-----------|
| **TypeScript 5.x** | Type safety | Catch errors at build time; essential for complex state |
| **React 19** or **Solid 1.9** | UI framework | Component model, ecosystem, hiring pool (React) / performance (Solid) |
| **TanStack Query** | Server state | Automatic refetching, caching, optimistic updates — already in this monorepo |
| **TanStack Router** | Client routing | Type-safe routing with search params |
| **Tailwind CSS 4** | Styling | Utility-first; matches dark terminal aesthetic easily |
| **Vite 6** | Build tool | Fast HMR, Go embed-friendly output |
| **Vitest** | Testing | Consistent with TanStack ecosystem |
| **dnd-kit** or **@tanstack/virtual** | Interaction | Drag-and-drop for kanban; virtualized lists for scale |

### Backend Additions (Go)

| Addition | Purpose |
|---------|---------|
| `gorilla/websocket` or `nhooyr.io/websocket` | WebSocket upgrade for real-time events |
| `fsnotify/fsnotify` | Watch hook/event files for changes (trigger WS pushes) |
| Structured JSON API versioning (`/api/v2/`) | Stable contract for the SPA |

---

## 5. Frontend Architecture

### Directory Structure

```
ui/
├── src/
│   ├── api/                    # API client & types
│   │   ├── client.ts           # HTTP + WebSocket client
│   │   ├── types.ts            # TypeScript types mirroring Go structs
│   │   └── hooks/              # TanStack Query hooks per resource
│   │       ├── useConvoys.ts
│   │       ├── useWorkers.ts
│   │       ├── useMail.ts
│   │       └── useEvents.ts
│   ├── components/
│   │   ├── layout/             # Shell, sidebar, header
│   │   ├── dashboard/          # Agent grid, summary cards, health
│   │   ├── kanban/             # Convoy kanban board
│   │   ├── timeline/           # Handoff timeline visualization
│   │   ├── agent-detail/       # Live agent output tailing
│   │   ├── mail/               # Inbox, compose, thread view
│   │   ├── command-palette/    # Cmd+K command palette
│   │   └── common/             # Badge, StatusDot, Card, etc.
│   ├── routes/                 # TanStack Router route definitions
│   ├── stores/                 # Client-only state (UI prefs, layout)
│   ├── lib/                    # Utilities (formatters, constants)
│   └── main.tsx                # Entry point
├── public/                     # Static assets (favicon, sounds)
├── index.html
├── vite.config.ts
├── tailwind.config.ts
├── tsconfig.json
└── package.json
```

### Core Views

| View | Route | Primary Data | Refresh |
|------|-------|-------------|---------|
| Dashboard | `/` | Workers, Convoys, Health, Sessions, Escalations | WebSocket push + 10s fallback poll |
| Kanban | `/kanban` | Convoys, Beads (issues), Workers | WebSocket push |
| Agent Detail | `/agents/:name` | Worker state, live output stream | WebSocket stream |
| Timeline | `/timeline` | Activity events, handoff records | Poll (historical) |
| Mail | `/mail` | Inbox, threads | WebSocket push for new mail |
| Settings | `/settings` | Config JSON files | On-demand |

### State Management Strategy

```
Server State (TanStack Query)         Client State (signals/stores)
─────────────────────────────         ─────────────────────────────
Convoys, Workers, Mail, Issues        Selected panel, layout prefs
Health, Sessions, Escalations         Command palette open/closed
Hooks, Merge Queue, Activity          Theme, sidebar collapsed
Config, Rigs, Dogs                    Notification preferences
```

- **Server state** is the source of truth. TanStack Query handles caching,
  refetching, and stale-while-revalidate.
- **WebSocket events** trigger targeted query invalidation (not full page
  refresh) — e.g., a `worker.stateChange` event invalidates only `useWorkers`.
- **Optimistic updates** for command execution (mark command as running
  immediately, reconcile on API response).

---

## 6. Backend Enhancements

### New Endpoints (API v2)

These extend the existing API without breaking v1 compatibility.

```
GET  /api/v2/convoys                  # Structured JSON (not CLI text parsing)
GET  /api/v2/convoys/:id              # Single convoy with beads
GET  /api/v2/workers                  # All workers with full state
GET  /api/v2/workers/:name            # Single worker detail
GET  /api/v2/workers/:name/output     # Buffered recent output (last N lines)
GET  /api/v2/events                   # Paginated event log
GET  /api/v2/events/stream            # SSE endpoint (fallback for WS)
GET  /api/v2/health                   # Structured health check
GET  /api/v2/rigs                     # Rig listing with stats
GET  /api/v2/hooks                    # Hook listing with metadata
GET  /api/v2/config                   # Read-only config snapshot
POST /api/v2/commands/execute         # Execute with structured response
WS   /api/v2/ws                       # WebSocket for real-time events
```

### WebSocket Event Protocol

```jsonc
// Client → Server (subscribe to event channels)
{ "type": "subscribe", "channels": ["workers", "convoys", "mail", "events"] }

// Server → Client (state change events)
{
  "type": "event",
  "channel": "workers",
  "action": "stateChange",
  "data": {
    "name": "polecat-alpha",
    "previousState": "working",
    "currentState": "stalled",
    "timestamp": "2025-01-15T10:30:00Z"
  }
}

// Server → Client (agent output streaming)
{
  "type": "stream",
  "channel": "output",
  "worker": "polecat-alpha",
  "line": "Running tests... 42/100 passed",
  "timestamp": "2025-01-15T10:30:01Z"
}
```

### Event Source Pipeline

```
JSONL Event Log ──► fsnotify watcher ──► Event Hub ──► WebSocket connections
                                              │
tmux pane output ──► periodic scraper ─────────┘
                     (every 2s)               │
                                              │
Hook file changes ──► fsnotify watcher ────────┘
```

The Go backend watches three sources:
1. **Event log** (`~/.events.jsonl`) — new lines trigger event broadcasts
2. **tmux panes** — periodic output scraping for live agent tailing
3. **Hook directories** — file changes signal state transitions

---

## 7. Data Model & API Contract

### TypeScript Types (mirroring Go structs)

```typescript
// Core entities from internal/web/templates.go
interface ConvoyRow {
  id: string
  title: string
  status: string
  workStatus: string
  progress: number          // 0-100
  completed: number
  total: number
  lastActivity: string
  trackedIssues: string[]
}

interface WorkerRow {
  name: string
  rig: string
  sessionId: string
  lastActivity: string
  statusHint: WorkerStatus  // "spinning" | "finished" | "questions" | "stale" | "stuck"
  issueId: string
  issueTitle: string
  workStatus: string
  agentType: string         // "polecat" | "refinery" | "witness"
}

interface MailRow {
  id: string
  from: string
  to: string
  subject: string
  timestamp: string
  priority: "normal" | "high" | "urgent"
  type: string
  read: boolean
}

interface HookRow {
  id: string
  title: string
  assignee: string
  agent: string
  age: string
  isStale: boolean
}

interface EscalationRow {
  id: string
  title: string
  severity: "low" | "medium" | "high" | "critical"
  escalatedBy: string
  age: string
  acked: boolean
}

interface RigRow {
  name: string
  gitUrl: string
  polecatCount: number
  crewCount: number
  hasWitness: boolean
  hasRefinery: boolean
}

interface SessionRow {
  name: string
  role: string
  rig: string
  worker: string
  activity: string
  isAlive: boolean
}

interface DashboardState {
  convoys: ConvoyRow[]
  workers: WorkerRow[]
  mergeQueue: MergeQueueData
  mail: MailRow[]
  rigs: RigRow[]
  dogs: DogData
  escalations: EscalationRow[]
  hooks: HookRow[]
  sessions: SessionRow[]
  issues: IssueRow[]
  activity: ActivityRow[]
  summary: SummaryData
  health: HealthData
  queues: QueueData
  mayor: MayorData
}
```

---

## 8. Real-Time Communication

### Strategy: WebSocket Primary, SSE Fallback, Polling Tertiary

```
Preference order:
1. WebSocket (/api/v2/ws)       ← full duplex, lowest latency
2. SSE (/api/v2/events/stream)  ← unidirectional, works through proxies
3. Polling (TanStack Query)     ← automatic 10s refetch as last resort
```

### Implementation in Go

```go
// internal/web/hub.go — WebSocket event hub
type Hub struct {
    clients    map[*Client]bool
    broadcast  chan Event
    register   chan *Client
    unregister chan *Client
}

type Client struct {
    conn       *websocket.Conn
    send       chan []byte
    channels   map[string]bool   // subscribed channels
}

type Event struct {
    Type      string      `json:"type"`
    Channel   string      `json:"channel"`
    Action    string      `json:"action"`
    Data      interface{} `json:"data"`
    Timestamp time.Time   `json:"timestamp"`
}
```

### Event Sources to Watch

| Source | Watch Mechanism | Events Emitted |
|--------|----------------|----------------|
| `~/.events.jsonl` | `fsnotify` on file | `events.*` (sling, spawn, kill, merge, etc.) |
| `{town}/mayor/rigs.json` | `fsnotify` | `rigs.changed` |
| Hook directories | `fsnotify` recursive | `hooks.created`, `hooks.completed` |
| tmux sessions | 2s poll via `tmux list-panes` | `workers.stateChange`, `workers.output` |
| Mail directories | `fsnotify` | `mail.received` |
| Convoy state files | `fsnotify` | `convoys.updated` |

---

## 9. Claude Code Setup & Configuration

### Repository-Level Configuration

#### `.claude/settings.json`

```jsonc
{
  "permissions": {
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Write",
      "Edit",
      "Bash(pnpm *)",
      "Bash(npm *)",
      "Bash(npx *)",
      "Bash(node *)",
      "Bash(go build *)",
      "Bash(go test *)",
      "Bash(go vet *)",
      "Bash(go run *)",
      "Bash(git *)",
      "Bash(vitest *)",
      "Bash(vite *)",
      "Bash(tsc *)",
      "Bash(eslint *)",
      "Bash(prettier *)",
      "Bash(golangci-lint *)"
    ],
    "deny": [
      "Bash(rm -rf /)",
      "Bash(gt * --force)",
      "Bash(go install *)"
    ]
  }
}
```

#### `CLAUDE.md` — Project Instructions for AI Agents

```markdown
# CLAUDE.md — Gastown UI

## Project Overview
This is the web UI layer for Gastown, a multi-agent orchestration system.
The UI is a TypeScript SPA that connects to Gastown's Go backend.

## Architecture
- Frontend: React + TypeScript + TanStack Query + Tailwind CSS
- Backend: Go net/http server (existing Gastown `internal/web/`)
- Build: Vite → embedded in Go binary via embed.FS
- Real-time: WebSocket for events, SSE fallback

## Working with the Codebase

### Frontend (ui/)
```bash
cd ui && pnpm install        # Install dependencies
pnpm dev                     # Dev server with HMR (port 5173)
pnpm build                   # Production build → ui/dist/
pnpm test                    # Run Vitest
pnpm lint                    # ESLint
pnpm typecheck               # tsc --noEmit
```

### Backend (Go)
```bash
go build ./cmd/gt            # Build CLI
go test ./internal/web/...   # Test web package
go test -race ./...          # Full test suite
golangci-lint run            # Lint
```

### Integration
```bash
cd ui && pnpm build          # Build SPA assets
go generate ./internal/web   # Embed assets (if using go:generate)
go build ./cmd/gt            # Build with embedded UI
./gt dashboard --port 8080   # Launch with new UI
```

## Conventions
- Go code follows golangci-lint rules (.golangci.yml)
- TypeScript uses strict mode
- API types in ui/src/api/types.ts must mirror Go structs
- All new API endpoints go under /api/v2/
- WebSocket events follow {type, channel, action, data} schema
- CSS uses Tailwind utilities; custom CSS only for complex animations
- Tests required for all new API endpoints and UI components

## Key Files
- `internal/web/api.go` — REST API handlers (existing v1 + new v2)
- `internal/web/hub.go` — WebSocket event hub (new)
- `internal/web/fetcher.go` — Data fetchers (existing, 14 sources)
- `ui/src/api/client.ts` — Frontend API client
- `ui/src/api/types.ts` — Shared TypeScript types
```

#### `.claude/hooks/` — Session Hooks

Hooks for automated setup when Claude Code starts a session:

**`.claude/hooks/session-start.sh`**
```bash
#!/bin/bash
# Ensure frontend dependencies are installed
if [ -d "ui" ] && [ ! -d "ui/node_modules" ]; then
  echo "Installing frontend dependencies..."
  cd ui && pnpm install
fi

# Ensure Go dependencies are available
go mod download 2>/dev/null

# Verify gt binary can build
go build -o /dev/null ./cmd/gt 2>/dev/null || echo "Warning: Go build failed"
```

#### `.claude/skills/` — Custom Skills

**`.claude/skills/build-and-test.md`**
```markdown
# Build & Test

Run the full build and test pipeline:
1. Build frontend: `cd ui && pnpm build`
2. Typecheck: `cd ui && pnpm typecheck`
3. Frontend tests: `cd ui && pnpm test`
4. Go tests: `go test ./internal/web/...`
5. Go lint: `golangci-lint run ./internal/web/...`
6. Full binary build: `go build ./cmd/gt`
```

### Multi-Agent Configuration with Gastown

When using Gastown itself to develop the UI (dogfooding), configure agents:

```bash
# Create a rig for the UI work
gt rig add gastown-ui https://github.com/user/gastown-ui-fork.git

# Configure specialized polecats
gt config agent set frontend-alpha "claude --model opus"
gt config agent set frontend-beta "claude --model opus"
gt config agent set backend-go "claude --model opus"
gt config agent set test-runner "claude --model sonnet"

# Create a convoy for the UI build
gt convoy create "Dashboard UI v2" gt-ui001 gt-ui002 gt-ui003

# Assign work
gt sling gt-ui001 gastown-ui   # "Build agent dashboard grid component"
gt sling gt-ui002 gastown-ui   # "Build kanban board with drag-and-drop"
gt sling gt-ui003 gastown-ui   # "Add WebSocket hub to Go backend"
```

---

## 10. GitHub Gitflow & Repository Setup

### Branching Strategy

This project uses a **modified Gitflow** model adapted for AI-assisted development
with Gastown's multi-agent workflow.

```
main (production)
  │
  ├── develop (integration branch)
  │     │
  │     ├── feature/dashboard-grid
  │     ├── feature/kanban-board
  │     ├── feature/websocket-hub
  │     ├── feature/agent-detail-view
  │     ├── feature/timeline-view
  │     └── feature/mail-ui
  │
  ├── release/v0.1.0
  │     └── (cherry-picks from develop, bugfixes only)
  │
  └── hotfix/critical-fix
        └── (merges to main AND develop)
```

### Branch Naming Convention

```
feature/<component>-<description>     # New features
fix/<issue-id>-<description>          # Bug fixes
refactor/<scope>-<description>        # Code restructuring
docs/<topic>                          # Documentation
ci/<change>                           # CI/CD changes
claude/<task-description>-<session>   # Gastown agent branches (auto-generated)
```

### Branch Protection Rules

#### `main`
- Require pull request reviews (1 reviewer minimum)
- Require status checks to pass (CI, lint, typecheck, tests)
- Require branches to be up to date before merging
- Require linear history (squash or rebase merges only)
- No force pushes
- No deletions

#### `develop`
- Require status checks to pass
- Allow merge commits (feature branches merge in)
- No force pushes

#### `claude/*`
- Auto-created by Gastown Polecats
- Must pass CI before merging to `develop`
- Auto-deleted after merge

### Repository Setup

```bash
# 1. Fork or clone Gastown
git clone https://github.com/steveyegge/gastown.git gastown-ui
cd gastown-ui

# 2. Set up branch structure
git checkout -b develop
git push -u origin develop

# 3. Set develop as default branch (GitHub UI or API)
gh repo edit --default-branch develop

# 4. Create branch protection rules
gh api repos/{owner}/{repo}/branches/main/protection -X PUT -f \
  required_status_checks='{"strict":true,"contexts":["ci","lint","test","typecheck"]}' \
  required_pull_request_reviews='{"required_approving_review_count":1}' \
  enforce_admins=true \
  restrictions=null

# 5. Set up labels for PRs
gh label create "frontend" --color "61dafb" --description "React/TypeScript UI changes"
gh label create "backend" --color "00add8" --description "Go backend changes"
gh label create "api" --color "ff6b6b" --description "API contract changes"
gh label create "real-time" --color "ffd93d" --description "WebSocket/SSE changes"
gh label create "agent-created" --color "a855f7" --description "Created by a Gastown agent"
```

### Commit Convention

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <description>

[optional body]

[optional footer]
```

| Type | Scope Examples | Usage |
|------|---------------|-------|
| `feat` | `dashboard`, `kanban`, `ws`, `api` | New features |
| `fix` | `worker-grid`, `mail`, `fetcher` | Bug fixes |
| `refactor` | `api-client`, `types` | Code restructuring |
| `test` | `hub`, `api`, `components` | Test additions |
| `docs` | `architecture`, `api` | Documentation |
| `build` | `vite`, `go`, `ci` | Build system changes |
| `style` | `tailwind`, `theme` | Visual/styling changes |

Examples:
```
feat(dashboard): add live agent output streaming panel
fix(ws): handle reconnection on network interruption
refactor(api): migrate fetcher responses to typed v2 format
test(kanban): add drag-and-drop reordering tests
build(vite): configure embed-friendly output for Go binary
```

### Pull Request Workflow

```
1. Agent/developer creates feature branch from develop
2. Work is done, tests pass locally
3. Push branch, open PR targeting develop
4. CI runs: lint, typecheck, unit tests, Go tests, build verification
5. Code review (human or Mayor review for agent PRs)
6. Squash merge to develop
7. When release-ready: create release/* branch from develop
8. Final testing on release branch
9. Merge release branch to main (tag + GitHub Release)
10. Merge main back to develop
```

### PR Template (`.github/PULL_REQUEST_TEMPLATE.md`)

```markdown
## Summary
<!-- What does this PR do? -->

## Changes
- [ ] Frontend (ui/)
- [ ] Backend (internal/web/)
- [ ] API contract (types)
- [ ] Tests

## Screenshots
<!-- For UI changes, include before/after screenshots -->

## Testing
- [ ] `pnpm test` passes
- [ ] `pnpm typecheck` passes
- [ ] `go test ./internal/web/...` passes
- [ ] Manual testing with `gt dashboard`

## Related
<!-- Link to Gastown bead/issue if applicable -->
Bead: gt-XXXXX
```

---

## 11. CI/CD Pipeline

### GitHub Actions Workflow

**`.github/workflows/ui-ci.yml`**

```yaml
name: UI CI

on:
  pull_request:
    paths:
      - 'ui/**'
      - 'internal/web/**'
      - '.github/workflows/ui-ci.yml'
  push:
    branches: [develop, main]

jobs:
  frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: 9
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
          cache-dependency-path: ui/pnpm-lock.yaml
      - run: cd ui && pnpm install --frozen-lockfile
      - run: cd ui && pnpm typecheck
      - run: cd ui && pnpm lint
      - run: cd ui && pnpm test -- --run
      - run: cd ui && pnpm build
      - uses: actions/upload-artifact@v4
        with:
          name: ui-dist
          path: ui/dist/

  backend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.24'
      - run: go test -race ./internal/web/...
      - run: golangci-lint run ./internal/web/...

  integration:
    needs: [frontend, backend]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.24'
      - uses: actions/download-artifact@v4
        with:
          name: ui-dist
          path: ui/dist/
      - run: go build ./cmd/gt
      - run: go test -tags=integration ./internal/web/...
```

---

## 12. Security Considerations

### Authentication (Phase 2)

The existing dashboard has no auth. For multi-user or remote access:

| Approach | Complexity | Use Case |
|----------|-----------|----------|
| No auth (localhost only) | None | Single developer, local only |
| Shared secret (Bearer token) | Low | Trusted network, simple remote access |
| OAuth2/OIDC (GitHub) | Medium | Team use, GitHub-authenticated access |
| mTLS (client certs) | Medium | Machine-to-machine, CI dashboards |

Recommended: **Start with localhost-only binding** (`127.0.0.1:8080`), add
optional Bearer token for remote access as a flag (`gt dashboard --token <secret>`).

### Existing Security Model (Preserve)

- Command whitelist in `commands.go` — do not expand without review
- Blocked pattern regex — maintain the blocklist
- Argument sanitization — strip shell metacharacters
- WebSocket commands must go through the same whitelist

### Content Security Policy

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self';
  style-src 'self' 'unsafe-inline';
  connect-src 'self' ws://localhost:* wss://localhost:*;
  img-src 'self' data:;
```

---

## 13. Testing Strategy

### Frontend Testing

| Layer | Tool | What to Test |
|-------|------|-------------|
| Unit | Vitest | Utility functions, formatters, state logic |
| Component | Vitest + Testing Library | Component rendering, interactions, props |
| Integration | Vitest + MSW | API client with mocked server responses |
| E2E | Playwright | Full dashboard flows, WebSocket behavior |

### Backend Testing

| Layer | Tool | What to Test |
|-------|------|-------------|
| Unit | Go `testing` | Handler functions, event parsing, hub logic |
| Integration | Go `testing` + `httptest` | API endpoints with test fetcher |
| E2E | `go-rod` (existing) | Browser-based dashboard testing |

### Test Data

Create a `testdata/` directory with fixture files:
- Mock JSONL event logs
- Sample convoy/worker/mail state
- Recorded tmux session output
- Config file snapshots

These fixtures allow testing without a running Gastown workspace.

---

## 14. Deployment & Distribution

### Embedding in the Go Binary

The SPA build output (`ui/dist/`) gets embedded into the Go binary:

```go
// internal/web/embed.go
package web

import "embed"

//go:embed ui/dist/*
var uiAssets embed.FS
```

Build pipeline:
```
pnpm build (ui/)  →  ui/dist/  →  go build (embeds dist/)  →  single gt binary
```

The `gt dashboard` command serves the embedded SPA. No separate Node.js process
needed at runtime.

### Distribution Channels (Existing)

Gastown already distributes via three channels. The UI is included automatically
once embedded:

1. **Homebrew** — `brew install steveyegge/gastown/gt`
2. **npm** — `npm install -g @gastown/gt` (downloads Go binary)
3. **Go install** — `go install github.com/steveyegge/gastown/cmd/gt@latest`

### Development Mode

For frontend development with HMR:

```bash
# Terminal 1: Go backend with API
go run ./cmd/gt dashboard --port 8080 --dev   # serves API only, no static files

# Terminal 2: Vite dev server
cd ui && pnpm dev                              # HMR on port 5173, proxies API to 8080
```

Vite proxy configuration:
```typescript
// vite.config.ts
export default defineConfig({
  server: {
    proxy: {
      '/api': 'http://localhost:8080',
      '/api/v2/ws': {
        target: 'ws://localhost:8080',
        ws: true,
      },
    },
  },
})
```

---

## Appendix: Implementation Phases

| Phase | Scope | Milestone |
|-------|-------|-----------|
| **P0** | Project scaffolding, Vite + React/Solid setup, API client, basic dashboard grid mirroring existing panels | "Feature parity in new stack" |
| **P1** | WebSocket hub in Go, real-time event push, live agent output streaming | "Real-time dashboard" |
| **P2** | Kanban board with drag-and-drop, convoy detail views | "Interactive work management" |
| **P3** | Timeline/handoff visualization, historical analysis | "Operational intelligence" |
| **P4** | Auth, multi-user, remote access, mobile responsive polish | "Team-ready" |
