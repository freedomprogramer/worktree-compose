---
name: worktree-compose
description: >
  Compose unified full-context workspaces from multiple independent repositories using git
  worktrees, with automatic dependency sharing and port conflict resolution. Use this skill
  when the user wants to work on a feature that spans multiple repos (backend + frontend +
  mobile), needs isolated branches for parallel feature development, mentions "worktree",
  "parallel development", "full-stack workspace", or asks to combine separate projects into
  one workspace. This skill solves two problems: (1) breaking down the silos between frontend,
  backend, and mobile development by giving code agents full cross-repo context in a single
  workspace root, and (2) enabling parallel coding on multiple features via git worktree
  isolation with zero-install dependency sharing and automatic dev server port allocation.
---

# Worktree Compose

## The Problem This Solves

Traditional development is siloed: frontend developers work in the frontend repo, backend
developers in the backend repo. API contracts are defined, implemented separately, and then
painfully debugged during integration. Code agents inherit this limitation — they can only
see the repo they're launched in.

This skill changes that by **composing multiple independent repos into a single workspace**,
so a code agent launched at the workspace root has full context across all projects. Combined
with git worktrees, this enables **parallel feature development** — multiple features can be
worked on simultaneously in isolated workspaces, each with its own branches across all repos.

```
Traditional (siloed):
  backend-repo/     ← agent 1 (no frontend context)
  frontend-repo/    ← agent 2 (no backend context)
  mobile-repo/      ← agent 3 (no API context)

Worktree Compose (unified):
  feature-workspace/
  ├── backend/      ← worktree of backend-repo
  ├── frontend/     ← worktree of frontend-repo
  └── mobile/       ← worktree of mobile-repo
                    ↑ one agent, full context
```

## Phase 1: Gather Information

When the user requests a new feature workspace, collect these inputs through conversation
or AskUserQuestion:

### 1.1 Required

- **Feature name** — used as branch name and workspace directory name
  (e.g., `shared-room-booking`, `fix-payment-flow`)

### 1.2 Optional (ask if not provided)

- **Additional projects** — other repos to include beyond the current one. User specifies:
  - The project path (e.g., `~/workspace/my-frontend`)
  - An optional label for the subdirectory (defaults to the directory name)

  Example conversation:
  ```
  User: "I need to add a booking feature. Also bring in the frontend
         at ~/workspace/frontend and mobile at ~/projects/mobile-app"
  ```

- **Worktree base directory** — where to create workspaces. If not specified, default to
  `<current-project-parent>/<current-project-name>-worktrees/`.

- **Base branch** — the branch to create the feature branch from. Defaults to the current
  branch of each project. The user may specify different base branches per project.

### 1.3 Result Structure Preview

Before creating anything, show the user what will be created:

```
# Single project:
<worktree-base>/<feature-name>/

# Multiple projects:
<worktree-base>/<feature-name>/
├── backend/        ← branch: <feature-name> (from main)
├── frontend/       ← branch: <feature-name> (from develop)
└── mobile/         ← branch: <feature-name> (from main)

Dev server ports:
  backend  :3010  (default 3000 + offset 10)
  frontend :5183  (default 5173 + offset 10)
```

Get user confirmation before proceeding.

## Phase 2: Analyze Each Project

For each project (current + additional), detect:

### 2.1 Git Status

```bash
cd <project-path>
git rev-parse --git-dir          # confirm it's a git repo
git branch --show-current        # current branch
git status --porcelain           # check for uncommitted changes
```

If uncommitted changes exist, warn the user: worktrees branch from HEAD, uncommitted
changes stay in the original working copy.

### 2.2 Package Ecosystem

Identify the ecosystem by checking for lock files:

| Indicator | Ecosystem | Dependency Dir |
|-----------|-----------|---------------|
| `pnpm-lock.yaml` | pnpm (Node) | `node_modules/` |
| `package-lock.json` | npm (Node) | `node_modules/` |
| `yarn.lock` | Yarn (Node) | `node_modules/` |
| `bun.lockb` | Bun (Node) | `node_modules/` |
| `Gemfile.lock` | Bundler (Ruby) | system gems |
| `requirements.txt` / `Pipfile.lock` / `poetry.lock` / `uv.lock` | Python | venv |
| `go.sum` | Go modules | global cache |
| `Cargo.lock` | Cargo (Rust) | `target/` |
| `composer.lock` | Composer (PHP) | `vendor/` |
| `Podfile.lock` | CocoaPods (iOS) | `Pods/` |
| `pubspec.lock` | pub (Flutter/Dart) | `.dart_tool/` |

### 2.3 Detect Dev Server Ports

Scan each project for its default dev server port:

```bash
# Node/Vite — check vite.config.*, package.json scripts, .env*
grep -r "port" vite.config.* 2>/dev/null
grep "PORT" .env .env.local .env.development 2>/dev/null

# Rails — check config/puma.rb, .env
grep "port" config/puma.rb 2>/dev/null

# Django — check manage.py or settings
grep "runserver" Makefile Procfile 2>/dev/null

# Spring Boot — check application.properties/yml
grep "server.port" src/main/resources/application.* 2>/dev/null

# General — check Procfile, docker-compose.yml, Makefile
grep -i "port" Procfile docker-compose.yml 2>/dev/null
```

Common defaults if not explicitly configured:

| Framework | Default Port |
|-----------|-------------|
| Vite | 5173 |
| Webpack Dev Server | 8080 |
| Create React App | 3000 |
| Next.js | 3000 |
| Nuxt | 3000 |
| Rails (Puma) | 3000 |
| Django | 8000 |
| Spring Boot | 8080 |
| Go (common) | 8080 |
| Flask | 5000 |
| Laravel | 8000 |
| Flutter Web | 随机 |

### 2.4 Config Files to Copy

Find untracked config files needed to run each project:

```bash
# Check .gitignore patterns, find matching files
# Common: .env, .env.local, config/database.yml, config/secrets.yml,
#         config/master.key, .npmrc, local.properties
```

## Phase 3: Create Worktrees

### 3.1 Create Directory Structure

```bash
mkdir -p <worktree-base>/<feature-name>
```

### 3.2 Create Each Worktree

For each project:

```bash
cd <project-path>

# Determine destination
# Single project:   <worktree-base>/<feature-name>/
# Multiple projects: <worktree-base>/<feature-name>/<label>/

if git show-ref --verify --quiet "refs/heads/<feature-name>" 2>/dev/null; then
  git worktree add <dest> <feature-name>
else
  git worktree add <dest> -b <feature-name>
fi
```

### 3.3 Copy Config Files

```bash
for cfg in <detected-config-files>; do
  if [[ -f "<project-path>/$cfg" && ! -f "<worktree-path>/$cfg" ]]; then
    cp "<project-path>/$cfg" "<worktree-path>/$cfg"
  fi
done
```

## Phase 4: Share Dependencies

Make each worktree immediately usable without running install commands.
Read `references/dependency-strategies.md` for detailed per-ecosystem strategies.

### General Principles

1. **Prefer symlinks** for dependency directories (node_modules, vendor, Pods).
2. **Rely on shared caches** when the package manager already has global caching
   (RVM gems, Go modules, Cargo registry, pnpm store) — nothing to do.
3. **Fall back to fresh install** if the main copy has no deps installed, or if the
   new branch changes dependency declarations.

### Quick Reference

| Ecosystem | Strategy | Fallback |
|-----------|----------|---------|
| pnpm | Symlink `node_modules/` | `pnpm install` (seconds) |
| npm | Symlink `node_modules/` | `npm ci` |
| Yarn | Symlink `node_modules/` | `yarn install --frozen-lockfile` |
| Bundler | Shared via RVM/rbenv, nothing to do | `bundle install` |
| pip/venv | Create new venv | `python -m venv .venv && pip install -r requirements.txt` |
| Poetry | Shared virtualenvs | `poetry install` |
| Go | Global module cache, nothing to do | `go mod download` |
| Cargo | Global registry, optionally symlink `target/` | `cargo build` |
| CocoaPods | Symlink `Pods/` | `pod install` |

When symlinking:

```bash
if [[ -d "<project-path>/node_modules" ]]; then
  ln -sf "<project-path>/node_modules" "<worktree-path>/node_modules"
fi
```

**Important**: Mention to the user that if the new branch changes dependency declarations,
they should break the symlink and run a fresh install.

## Phase 5: Resolve Port Conflicts

When a project already has a worktree running (or the main copy is running), assign unique
ports to avoid conflicts.

### 5.1 Calculate Port Offset

Use the worktree sequence number as an offset multiplier:

```
offset = worktree_index * 10

# Example: project default port is 3000
# main working copy:  3000 (offset 0)
# worktree 1:         3010 (offset 10)
# worktree 2:         3020 (offset 20)
```

To determine `worktree_index`, count existing worktrees **before** creating the new one:

```bash
cd <project-path>
git worktree list | wc -l   # includes main, so index = count - 1
# Calculate BEFORE creating the worktree, not after
```

### 5.2 Write Port Configuration

Create or update a `.env.local` (or equivalent) in each worktree with the assigned ports.

**Important**: Check if the port is hardcoded in config files (e.g., `vite.config.*`, `puma.rb`).
If so, warn the user that the `.env.local` port may not take effect and suggest either:
- Modifying the config to read from environment variables
- Using the CLI flag instead (e.g., `vite --port <N>`, `rails s -p <N>`)

**Node.js (Vite/CRA/Next.js):**
```bash
# Append to .env.local
echo "PORT=<new-port>" >> <worktree>/.env.local
echo "VITE_PORT=<new-port>" >> <worktree>/.env.local  # if Vite
```

**Rails:**
```bash
# Write to .env or create a .port file
echo "PORT=<new-port>" >> <worktree>/.env
# Or remind user: rails s -p <new-port>
```

**Django:**
```bash
echo "DJANGO_PORT=<new-port>" >> <worktree>/.env
# Remind user: python manage.py runserver 0.0.0.0:<new-port>
```

**Spring Boot:**
```bash
# Append to application-local.properties
echo "server.port=<new-port>" >> <worktree>/src/main/resources/application-local.properties
```

For frameworks where port can't be set via config file, include the port flag in the
launch command shown to the user.

### 5.3 Show Port Assignment Table

After creating worktrees, display the full port mapping:

```
Port allocation for workspace "booking-feature":
┌─────────────┬──────────────┬──────────┐
│ Project     │ Default Port │ Assigned │
├─────────────┼──────────────┼──────────┤
│ backend     │ 3000         │ 3010     │
│ frontend    │ 5173         │ 5183     │
│ mobile-api  │ 8080         │ 8090     │
└─────────────┴──────────────┴──────────┘
```

If the frontend needs to call the backend API, also update the API base URL in the
frontend's `.env.local`:

```bash
echo "VITE_API_URL=http://localhost:<backend-port>" >> <frontend-worktree>/.env.local
```

## Phase 6: Guide Parallel CLI Sessions

Detect available CLIs and present launch commands:

```bash
which claude 2>/dev/null && echo "claude"
which codex 2>/dev/null && echo "codex"
```

Output:

```
Workspace ready! Launch code agents in new terminals:

# Full-context agent (recommended for cross-repo features):
cd <worktree-base>/<feature> && <cli>

# Or per-project agents:
cd <worktree-base>/<feature>/<backend> && <cli>
cd <worktree-base>/<feature>/<frontend> && <cli>
```

Highlight the **full-context option** — launching the agent at the workspace root gives it
visibility into all projects, which is the primary value of composing repos together.

## Worktree Lifecycle

### List

```bash
cd <project-path> && git worktree list
```

### Remove

```bash
# Remove each project's worktree
cd <project-path> && git worktree remove <worktree-path> --force
cd <project-path> && git worktree prune

# Clean up workspace directory
rmdir <worktree-base>/<feature-name> 2>/dev/null
```

### Clean

```bash
cd <project-path> && git worktree prune -v
```

## Important Notes

- Always confirm the workspace structure and port assignments with the user before creating.
- Two worktrees cannot have the same branch checked out — warn if the branch exists elsewhere.
- Worktrees share the `.git` object store — disk usage is minimal.
- If a `worktree.sh` or `compose.sh` script exists in the project, prefer using it.
- The user may want different branch names per project — ask if needed.
