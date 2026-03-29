# worktree-compose

Compose unified full-context workspaces from multiple independent repositories using git worktrees.

## The Problem

Traditional development is siloed — frontend, backend, and mobile teams work in separate repos, communicate through API specs, and debug integration issues in painful back-and-forth cycles. Code agents inherit this limitation: they can only see the repo they're launched in, missing the full picture.

## The Solution

**worktree-compose** creates a unified workspace that combines multiple repos into one directory. A code agent launched at the workspace root can see all projects — backend API routes, frontend components, mobile screens — and implement cross-cutting features in one shot.

Combined with git worktrees, you can work on **multiple features in parallel**, each in its own isolated workspace with its own branches, shared dependencies, and dedicated dev server ports.

```
Traditional (siloed):                    worktree-compose (unified):

  backend/    ← agent 1 (blind)          feature-workspace/
  frontend/   ← agent 2 (blind)          ├── backend/     ← worktree
  mobile/     ← agent 3 (blind)          ├── frontend/    ← worktree
                                          └── mobile/      ← worktree
                                                ↑ one agent, full context
```

## Features

- **Multi-repo composition** — bring any number of independent repos into one workspace
- **Git worktree isolation** — each feature gets its own branches, no interference
- **Zero-install dependency sharing** — symlinks node_modules, leverages shared gem/module caches
- **Port conflict resolution** — auto-assigns unique dev server ports per workspace
- **Any code agent** — works with Claude Code, Codex CLI, or any CLI-based agent

## Supported Ecosystems

Node.js (pnpm/npm/Yarn/Bun) | Ruby (Bundler) | Python (pip/Poetry/uv) | Go | Rust (Cargo) | PHP (Composer) | iOS (CocoaPods) | Flutter (pub)

## Install

### Claude Code

```bash
claude skill add --from github:freedomprogramer/worktree-compose
```

### Manual

```bash
cp -r . ~/.claude/skills/worktree-compose/
```

## Usage

### Single project

```
> I need to fix a pagination bug

# Creates an isolated worktree:
# my-project-worktrees/fix-pagination/
```

### Multi-repo workspace

```
> I want to add a booking feature.
> Also bring in the frontend at ~/workspace/web-app
> and mobile at ~/projects/mobile-app.

# Creates:
# my-project-worktrees/booking-feature/
#   ├── my-project/    (backend)
#   ├── web-app/       (frontend, port 5183)
#   └── mobile-app/    (mobile)
#
# Port allocation:
#   backend  :3010 (default 3000 + 10)
#   frontend :5183 (default 5173 + 10)
```

### Launch agents

```bash
# Full-context (recommended):
cd my-project-worktrees/booking-feature && claude

# Or per-project:
cd my-project-worktrees/booking-feature/my-project && claude    # terminal 1
cd my-project-worktrees/booking-feature/web-app && codex        # terminal 2
```

## License

MIT
