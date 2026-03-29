# Dependency Sharing Strategies by Ecosystem

Detailed strategies for sharing dependencies between main working copy and worktrees.

## Table of Contents
- [Node.js (pnpm)](#nodejs-pnpm)
- [Node.js (npm)](#nodejs-npm)
- [Node.js (Yarn)](#nodejs-yarn)
- [Ruby (Bundler)](#ruby-bundler)
- [Python (pip/venv)](#python-pipvenv)
- [Python (Poetry)](#python-poetry)
- [Go](#go)
- [Rust (Cargo)](#rust-cargo)
- [PHP (Composer)](#php-composer)

---

## Node.js (pnpm)

pnpm uses a global content-addressable store. All packages are stored once in
`~/.local/share/pnpm/store/` (or `~/Library/pnpm/store/` on macOS) and hard-linked
into each project's `node_modules/`.

**Strategy: Symlink node_modules/**

```bash
ln -sf "<main>/node_modules" "<worktree>/node_modules"
```

This works because pnpm's node_modules is essentially a tree of symlinks already.
If the worktree's `package.json` diverges, break the symlink and run `pnpm install`
(typically completes in seconds since packages are in the global store).

**Caveat**: If the project uses `.npmrc` with `shamefully-hoist=true` or custom
settings, ensure `.npmrc` is also present in the worktree.

## Node.js (npm)

npm downloads packages into a local `node_modules/` directory.

**Strategy: Symlink node_modules/**

```bash
ln -sf "<main>/node_modules" "<worktree>/node_modules"
```

**Fallback**: `npm ci` (clean install from lock file, faster than `npm install`).

**Caveat**: Native addons compiled with node-gyp may reference absolute paths.
If you see errors about missing `.node` files after symlinking, run `npm rebuild`
in the worktree or do a fresh `npm ci`.

## Node.js (Yarn)

Yarn v1 works like npm. Yarn v2+ (Berry) uses Plug'n'Play with a `.yarn/cache/`
directory and no `node_modules/`.

**Yarn v1 Strategy: Symlink node_modules/**

```bash
ln -sf "<main>/node_modules" "<worktree>/node_modules"
```

**Yarn Berry Strategy**: The `.yarn/cache/` is typically committed to git, so it's
already present in the worktree. If not:

```bash
ln -sf "<main>/.yarn/cache" "<worktree>/.yarn/cache"
```

Then run `yarn install` (instant with cache present).

## Ruby (Bundler)

Bundler installs gems to a shared location by default:
- With RVM: `~/.rvm/gems/<ruby-version>/`
- With rbenv: `~/.rbenv/versions/<ruby-version>/lib/ruby/gems/`
- System Ruby: system gem path

**Strategy: Nothing to do**

Gems are already shared globally. As long as the worktree uses the same Ruby version,
all gems are immediately available. No symlinks needed.

**Exception**: If the project uses `bundle install --path vendor/bundle` (local gem
installation), you'd need to either symlink `vendor/bundle` or run `bundle install`
in the worktree. Check `bundle config path` to determine this.

**Config files to copy**: `database.yml`, `secrets.yml`, `master.key`,
`credentials.yml.enc`, and any other untracked config in `config/`.

## Python (pip/venv)

Python virtual environments contain hardcoded paths and cannot be symlinked.

**Strategy: Create new venv, install from cache**

```bash
cd "<worktree>"
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt  # uses pip cache, typically fast
```

pip caches downloaded wheels in `~/.cache/pip/` (Linux) or
`~/Library/Caches/pip/` (macOS), so reinstalling is faster than it looks.

**Alternative**: If using `uv` (fast Python package installer):
```bash
uv venv .venv
uv pip install -r requirements.txt  # very fast
```

## Python (Poetry)

Poetry stores virtualenvs in `~/.cache/pypoetry/virtualenvs/` by default,
keyed by project name + Python version.

**Strategy: Shared virtualenvs (usually automatic)**

If the worktree has the same `pyproject.toml` name and Python version,
Poetry may reuse the same virtualenv. Otherwise:

```bash
cd "<worktree>"
poetry install  # uses cached packages, relatively fast
```

If `poetry config virtualenvs.in-project true` is set, the venv is local
and you'll need a fresh install (same as pip/venv strategy).

## Go

Go modules are cached globally in `$GOPATH/pkg/mod/` (default `~/go/pkg/mod/`).

**Strategy: Nothing to do**

The module cache is shared across all projects. The worktree will use cached
modules immediately. The `go build` may need to compile, but downloading is skipped.

**Optional**: Symlink the `vendor/` directory if the project vendors dependencies:
```bash
ln -sf "<main>/vendor" "<worktree>/vendor"
```

## Rust (Cargo)

Cargo stores the package registry and source cache in `~/.cargo/registry/`.
Build artifacts go into a local `target/` directory.

**Strategy: Shared registry (automatic) + optional target symlink**

The registry cache is global — no action needed for dependency downloads.
For build artifacts:

```bash
# Optional: share compiled dependencies (saves compile time)
ln -sf "<main>/target" "<worktree>/target"
```

**Caveat**: Sharing `target/` can cause issues if the two worktrees are on
very different branches. In that case, let each worktree have its own `target/`.

## PHP (Composer)

Composer downloads packages to a global cache in `~/.cache/composer/` (Linux)
or `~/Library/Caches/composer/` (macOS), and installs them to local `vendor/`.

**Strategy: Symlink vendor/**

```bash
ln -sf "<main>/vendor" "<worktree>/vendor"
```

**Fallback**: `composer install` (fast with cached packages).

**Caveat**: If the project uses `composer dump-autoload` with classmap optimization,
the autoload files in `vendor/composer/` contain absolute paths. Run
`composer dump-autoload` in the worktree after symlinking.
