# AGENTS.md — orientation for coding agents

Read this before changing anything. `README.md` is the user-facing reference
(install, keys, layouts) — don't duplicate it here; update it when behaviour
changes.

## What this repo is

`mn` — about 745 lines of bash, no daemon, no dependencies beyond tmux, fzf and
git, plus `gh` if you want the issues sidebar. fzf must be 0.46+: the sidebars
lay their rows out from `$FZF_COLUMNS` and redraw on the `resize` event, both of
which landed in that release. It drives **two tmux servers**:

- `mn-chrome` — one window, `[sidebar | view | issues]`, created once and never
  rebuilt. The issues pane is optional, built on first `M-i` and hidden by
  moving it to a detached window.
- `mn-work` — one session per project in `projects.conf`, plus one per task
  worktree, named `<project>/<branch>`.

It also owns the full lifecycle of a task worktree: create the branch and
checkout, allocate a port, run the project's `setup`, and on removal kill the
session *before* running `teardown` and letting git delete anything.

The view pane is just a client attached to `mn-work`. Picking in the sidebar
runs `switch-client` on the inner server; the outer layout never moves. **That
is the entire point of the design** — a single-server sidebar is a pane inside
each session, so it loses its width on every switch. Don't "simplify" this to
one server.

## Invariants — do not break these

- **The outer window is never rebuilt.** Anything that recreates `ui:main`
  reintroduces the bug the project exists to avoid. The single exception is
  `mn reload`, which the user runs deliberately to pick up a change to the
  keymap or the sidebar (`ensure_chrome` returns early whenever `ui` exists, so
  nothing else would). It kills the whole outer server and leaves `mn-work`
  running. That is not a precedent for rebuilding `ui` from any code path the
  user did not ask for.
- **The outer server has no prefix.** Its root key table *is* the app keymap;
  every key not bound there falls through to the inner server and to the
  programs in the panes, so `C-b` stays the project's own. Every key you add at
  the outer layer is stolen from nvim and claude permanently — keep it inside
  `M-` and `C-S-`.
- **Check what already owns a key before proposing one.** Three layers claim
  keys before you do, and the terminal is the one people forget: kitty here maps
  `ctrl+c`, `ctrl+n`, `ctrl+v` and `ctrl+shift+e`, so those never reach tmux at
  all. Then the outer root table (`M-*`, `C-S-h/l`, `C-hjkl`), then fzf's own
  defaults inside a sidebar pane (`ctrl-a/b/g/n/p/q/y`, and `ctrl-r` is mn's
  reload). Plain `C-` keys are the pane programs' to spend, since the outer
  server only claims `M-` and `C-S-`. Verified inert in a live fzf and free at
  every layer: `ctrl-o` and `ctrl-t`.

- **`~/.config/tmux/tmux.conf` is loaded by both servers.** The user's
  `bind -n C-h select-pane -L` is why `C-hjkl` has to be re-bound on the outer
  server and delegated with `mn nav`: the outer server sees every key first.
- **`projects.conf` is gitignored** — it holds real local paths. Edit
  `projects.conf.example`; `projects()` copies it on first run.
- **Address panes by `#{pane_id}`, never by index.** Not a style preference:
  `break-pane` then `join-pane` renumbers indexes *without moving anything*, so
  after one `M-i` cycle `ui:main.0` can be the view rather than the sidebar, and
  `select-layout` then physically reorders the panes to match. The ids live in
  `@sb_pane`, `@view_pane` and `@rsb_pane`, set once when the window is built.
  Measured, not assumed: a rejoined leftmost pane reported `pane_index=1`.

- **Hiding a pane means `break-pane -d`, never killing it.** The pane, its id and
  the process inside it survive, widths come back exactly, and `ui:main` is not
  rebuilt. The alternative of shrinking to a sliver is a dead end: the minimum
  pane width is 1, so it still costs two columns with the border, and restoring
  drifted the *other* sidebar by a column.

- **Resizing a pane that zoom has hidden drops the zoom.** So `pin` returns early
  on `#{window_zoomed_flag}`, or the `window-resized` hook ejects the user from
  `M-z` every time the terminal changes size. Because the pin holds off while
  zoomed, and a window resized while zoomed comes back proportionally scaled,
  `zoom` re-pins on the way out.

- **A session name with a `/` in it is a task worktree; one without is a main
  checkout.** That is the only thing distinguishing them, and `rm` refuses to
  act on a name with no `/`. `tmux` allows `/` in a session name (`.` and `:`
  are the forbidden ones), and it sorts so that `proj`, `proj/a`, `proj/b` come
  out already grouped — the sidebar's nesting is that sort, not a tree.

- **Nothing pushes to the sidebar.** Rows are rendered from
  `~/.local/state/mullion/<project>/<branch>/meta` at draw time. The predecessor
  (`herdr-conductor`) pushed display tokens to an in-memory store and then
  needed two extra event hooks to re-push them after a restart. Anything that
  changes what a row says calls `sidebar_reload`; that is the whole mechanism.
  A resize is the one thing that does not need it, because fzf reloads itself on
  its own `resize` event.

- **A pane program calls `mn`; `mn` never calls back with a payload.** The
  sidebar and the issues pane are small programs living in a pane, and the whole
  contract between them and `mn` is: print `<what you see>\t<id>` on stdout, and
  run an `mn` subcommand when the user picks something. The only thing `mn` ever
  sends outward is "redraw yourself" (`sidebar_reload`, `issues_reload`), which
  carries nothing. Keep it that way. The moment `mn` broadcasts and panes
  subscribe you have `herdr-conductor` back: it pushed display state outward and
  then needed re-push hooks, locks and a dirty-file trick to order itself.
  Only 11 of `mn`'s lines mention fzf, and that is the property that makes the
  renderer replaceable.

- **A pane program renders and takes input. It never owns state.** The `meta`
  files are the truth and rows are built from them at draw time. A pane holding
  its own idea of what exists is two sources of truth, and it is what makes the
  next renderer swap expensive.

- **A project script's only output is `$MN_ENV`.** It writes `KEY=value`, mn
  sources that into the `dev` command's environment. Do *not* reintroduce a
  "compose a shell one-liner and inject it into a pane" step — that is what the
  `provision` subcommand exists to replace, and it is why nothing here has to
  quote a port into a command string.

- **Known gap:** a worktree created *before* its project had a provisioner has
  no `PORT` in its `meta`, so `dev` would run with an empty `$PORT`. Nothing
  re-allocates one. Fix it in `provision` (allocate on the spot when `PORT` is
  empty and a provisioner now exists) rather than by special-casing at a call
  site.

- **Worktree ordering is sequential, not event-driven.** `rm_worktree` kills the
  session, waits, runs `teardown`, then removes the checkout — in one process,
  in that order. `herdr-conductor` had to keep every checkout untracked-dirty to
  trick git into forcing its host to stop panes first. Do not add events, locks,
  marker files or re-fire guards back in; `reconcile` at startup is what covers
  worktrees deleted behind mn's back.

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

## Bash and fzf gotchas

- **`printf` precision counts BYTES, not display columns.** `%-22.22s` on a
  string starting with `…` (3 bytes) fits two fewer characters, which knocks the
  port column out of alignment on exactly the rows that have a status mark. The
  sidebar's marks (`~`, `!`) are ASCII for this reason.

- **fzf reserves 4 of the sidebar's columns** — 2 for its pointer gutter, 2 at
  the right edge. A row wider than the pane gets its tail replaced with `··`.
  `list_rows` pads to `w - 10` (name) `+ 1 + 5` (port).

- **A row's width comes from `$FZF_COLUMNS`, not `@sbwidth`.** fzf exports it to
  the commands it reloads from, so a row lays itself out for the pane as it is
  now. `@sbwidth` only records what `mn width` last asked for, and a divider
  dragged with the mouse never touches it, so reading it left the port badge
  stranded mid-pane. It is 0 on fzf's `start` event, hence the fallback to the
  pane's own `#{pane_width}` for the first draw.

- **`set -euo pipefail` turns a missing file into a truncated list.** `sed …
  file | head -1` on an absent file exits 2, pipefail propagates it, `set -e`
  kills `mn list` mid-stream and the sidebar silently loses every row after it.
  `meta` guards with `[ -f ]` for exactly this.

- **`set -e` does not fire on a failed `A && B`.** All the `[ -f x ] && { …; }`
  guards rely on that; verified, not assumed.

- **The issues pane is cached, and the cache is the contract.** `issue_fetch`
  writes `~/.local/state/mullion/<project>/issues` and nothing else reads `gh`.
  Walking the left sidebar calls `issues_reload` on every row, so an uncached
  render would be an API call per keystroke. A failed fetch keeps the previous
  file rather than truncating it.

- **`/dev/tcp` is a bash builtin**, so the free-port check needs no `nc` or
  python. It is a *connect* test, so it cannot see a port that is allocated but
  idle — `alloc_port` also skips every port already in a worktree's `meta`.

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
