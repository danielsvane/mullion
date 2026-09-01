# AGENTS.md — orientation for coding agents

Read this before changing anything. `README.md` is the user-facing reference
(install, keys, layouts) — don't duplicate it here; update it when behaviour
changes.

## What this repo is

`mn` — about 230 lines of bash, no daemon, no dependencies beyond tmux and fzf.
It drives **two tmux servers**:

- `mn-chrome` — one window, `[sidebar | view]`, created once and never rebuilt.
- `mn-work` — one session per project, listed in `projects.conf`.

The view pane is just a client attached to `mn-work`. Picking in the sidebar
runs `switch-client` on the inner server; the outer layout never moves. **That
is the entire point of the design** — a single-server sidebar is a pane inside
each session, so it loses its width on every switch. Don't "simplify" this to
one server.

## Invariants — do not break these

- **The outer window is never rebuilt.** Anything that recreates `ui:main`
  reintroduces the bug the project exists to avoid.
- **The outer server has no prefix.** Its root key table *is* the app keymap;
  every key not bound there falls through to the inner server and to the
  programs in the panes, so `C-b` stays the project's own. Every key you add at
  the outer layer is stolen from nvim and claude permanently — keep it inside
  `M-` and `C-S-`.
- **`~/.config/tmux/tmux.conf` is loaded by both servers.** The user's
  `bind -n C-h select-pane -L` is why `C-hjkl` has to be re-bound on the outer
  server and delegated with `mn nav`: the outer server sees every key first.
- **`projects.conf` is gitignored** — it holds real local paths. Edit
  `projects.conf.example`; `projects()` copies it on first run.
- **Address panes by `#{pane_id}`, never by index.** Index math breaks on
  layouts with more than two panes and on a user's `pane-base-index`.

## tmux gotchas (each of these cost real time)

- **`select-pane -L` wraps at the edge.** Detect the left edge with
  `#{pane_left}` *before* moving, or `C-h` on the leftmost pane jumps to the
  rightmost one instead of entering the sidebar.
- **tmux rescales panes proportionally** when a client attaches and when the
  terminal resizes, which overrides any build-time `resize-pane`. The sidebar
  needs hooks on **`client-attached` and `window-resized`**. `client-resized`
  does *not* hold it — measured drift to 62 columns on a resize to 260.
- **`run-shell` must be `-b`.** Without it the tmux server blocks while `mn`
  calls back into that same server, and deadlocks.
- **`extended-keys on` alone is inert.** tmux ships no `extkeys` terminal
  feature for any terminal — the defaults are only `xterm*:clipboard:ccolour:
  cstyle:focus:title`, `screen*:title`, `rxvt*:ignorefkeys`. Hence the explicit
  `set -as terminal-features 'xterm-kitty:extkeys'`. Other terminals need their
  own entry or `C-S-*` silently does nothing.
- **Detached sessions are built at 80x24.** Percentage splits only take their
  intended proportions once a client attaches; don't chase the numbers before
  then.
- **`mn` cannot run without a tty** — it ends in `tmux attach`. Everything
  before that still executes, so running it from a script pre-builds both
  servers and only the attach fails.

## Testing

`send-keys` into a pane **bypasses root-table bindings**, so you cannot test a
keybinding that way. Run `mn` inside a third tmux server and send keys to *that*:

```bash
mkdir -p /tmp/t && cp mn /tmp/t/mn
sed -i 's/^CHROME=mn-chrome/CHROME=t-chrome/; s/^WORK=mn-work/WORK=t-work/' /tmp/t/mn
tmux -L probe new-session -d -x 180 -y 45 /tmp/t/mn
tmux -L probe send-keys -t 0 M-3
tmux -L t-chrome show -gv @project          # assert
tmux -L probe kill-server; tmux -L t-chrome kill-server; tmux -L t-work kill-server
```

**Never test against `mn-chrome` / `mn-work`.** Those are the user's live
session, with running agents in the panes.

Extended keys do not survive that harness (the inner `TERM` has no `extkeys`,
so tmux never requests them). Inject the raw CSI-u bytes instead — this is
Ctrl+Shift+L:

```bash
tmux -L probe send-keys -t 0 -H 1b 5b 31 30 38 3b 36 75
```

## Conventions

Commits are authored in the user's voice: no AI attribution trailer or footer,
imperative human-styled subject, minimal diffs. Don't add explanatory comments
unless asked.
