---
name: firstmate-test-env
description: Spin up a live strength testing environment in a worktree with a herdr pane layout. Use when the captain says "spin up a testing environment", "test env", "worktree env", "spin up an env", or asks for a runnable environment/pane setup for the strength app.
---

# Firstmate test env

Spin up a fully working strength dev environment in an isolated worktree, with a
two-tab herdr layout: tab 1 = crewmate + terminal-browser; tab 2 = web + server.
Proven end-to-end (2026-10-21/22); follow the order exactly.

## 1. Acquire the worktree

```bash
treehouse get strength -b <branch-name>
```

- Always pass a branch. A detached-HEAD lease produces the slug `main`, which
  collides with the primary checkout's portless route.
- The worktree lands at `~/.treehouse/strength-dc4529/<slot>/strength`.

## 2. Provision

Poll `~/.cache/fm-treehouse-provision/<md5 of worktree path>.status` until
`done`; treat `failed` as a block (log sits next to it). A transient race
(「database system is shutting down」 during the seed step) is fixed by
re-running `pnpm worktree:init` once — then the status file may read failed
while the worktree is actually fine; verify by booting.

After MR !3271 (merge-seed fix), `worktree:init` preserves the primary
checkout's real values (LD_SDK_KEY, AXLE_URL) and overrides only
worktree-specific keys (ports, DB/zero URLs, cookie domain).

## 3. Serve (always through portless)

```bash
pnpm nx serve:worktree web      # portless: <branch>.web.strength.localhost
pnpm nx serve:worktree server   # portless: <branch>.api.strength.localhost
```

Never hand-set ports, never run bare `nx serve` in a worktree, never hand-edit
the seeded env files. If a serve fails with 「already registered by a running
process」, a previous server still holds the route — find it with
`lsof -nP -iTCP -sTCP:LISTEN` and kill that PID before re-running.

## 4. Herdr layout

```bash
herdr workspace create --label <label> --cwd <worktree-path>
herdr agent start <crew-name> --kind pi --pane <root-pane>
herdr pane split --pane <root-pane> --direction right --cwd <worktree-path> --no-focus
herdr pane run <browser-pane> "terminal-browser open https://<branch>.web.strength.localhost"
herdr tab create --workspace <ws> --cwd <worktree-path> --label servers
herdr pane run <tab2-root> "pnpm nx serve:worktree web"
herdr pane split --pane <tab2-root> --direction right --cwd <worktree-path> --no-focus
herdr pane run <server-pane> "pnpm nx serve:worktree server"
```

Tab 1: crewmate left, terminal-browser right. Tab 2: web left, server right.

## 5. Verify before reporting done

1. `curl -sk -o /dev/null -w '%{http_code}' https://<branch>.web.strength.localhost/` → 200/302.
2. `curl -sk https://<branch>.api.strength.localhost/health` → 200.
3. `terminal-browser action -- eval "document.title"` → "Teamworks Strength" (not
   chrome-error, not an error boundary).
4. LocoSync must not be stuck `initializing` — check the browser console.

## Known residual quirks

- The portless proxy (runs as root) mints route certs at registration; if a
  route's TLS fails with `SSL_ERROR_SYSCALL` while `curl -v` shows the cert ok
  for curl but the browser errors, the proxy needs a restart:
  `sudo portless proxy stop && portless proxy start` (captain runs it).
- The worktree server's Datadog log-streaming 401 (「invalid SDK key」) is
  harmless noise in worktrees — the flag provider falls back to local defaults.
- If `AXLE_URL is Required` crashes the server boot, the worktree env predates
  the merge-seed fix: append the AXLE values from `.env.sample`'s uat lines
  rather than hand-copying from the primary.

## Teardown

Ask the captain, then: close the workspace (`herdr workspace close`), and
return the lease (`treehouse return <path>` — firstmate's policy denies this
from agent sessions; the captain runs it).
