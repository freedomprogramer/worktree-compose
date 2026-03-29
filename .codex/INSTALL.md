# Codex CLI Installation

## Install

```bash
git clone https://github.com/freedomprogramer/worktree-compose.git ~/.codex/worktree-compose
ln -sf ~/.codex/worktree-compose/skills/worktree-compose ~/.agents/skills/worktree-compose
```

## Update

```bash
cd ~/.codex/worktree-compose && git pull
```

## Uninstall

```bash
rm ~/.agents/skills/worktree-compose
rm -rf ~/.codex/worktree-compose
```
