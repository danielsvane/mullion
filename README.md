# mullion

A tmux sidebar that never moves.

A mullion is the vertical bar dividing a window into panes. This is one: a
permanent sidebar listing your projects, and a main view that swaps between them
without the sidebar ever losing its width, its scroll position, or its place.

It is about 763 lines of bash and no daemon.

```bash
./mn          # start + attach
./mn reload   # rebuild the keymap after editing mn, projects keep running
./mn stop     # tear both servers down
```

`M-q` detaches, it does not stop anything, so a plain `mn` afterwards reattaches
to the outer server you already had. After changing `mn` itself, use `mn reload`:
it rebuilds the outer server so the new keys and sidebar take effect, and leaves
the inner one alone so whatever is running in your project panes survives. It
will not run from inside mullion, because it has to kill the server you would be
typing through. Layout changes are the exception, existing sessions keep the
panes they were built with.

The left pane lists your projects and the task worktrees under them; `↑`/`↓` and
`Enter` (or a double-click, mouse is on) swap the right pane to that one. The
sidebar keeps its width across switches, reattaches, and terminal resizes.

`M-n` makes a new task worktree: a branch, a checkout, a port if the project
asks for one, whatever setup that project needs, and a coding agent already
holding the task you typed.

`M-i` opens a second sidebar on the right listing the project's ten newest open
GitHub issues, and `M-z` zooms the view over both sidebars. Both toggle back to
exactly the widths they had.

## Install

Requires **tmux 3.2+** (developed on 3.7) and **fzf 0.46+**, which is where the
`resize` event and `$FZF_COLUMNS` arrived (tested on 0.74).

[`gh`](https://cli.github.com) is optional, and only for the issues sidebar.

```bash
git clone https://github.com/danielsvane/mullion ~/projects/mullion
cd ~/projects/mullion
./mn                                # writes projects.conf from the template
$EDITOR projects.conf
./mn
ln -s "$PWD/mn" ~/.local/bin/mn     # optional
```

## Projects

`projects.conf` is one project per line, with an optional layout column:

```
~/projects/api        dev
~/projects/frontend   agent
```

- `dev` (the default) — `claude` on the left; `nvim` above a spare shell on the
  right.
- `agent` — `claude` on the left, a spare shell on the right.

Task worktrees always use `task` — a `claude` already holding the task you typed,
beside the project's dev server over a spare shell — regardless of the project's
column.

A layout is just a `layout_<name>` function in `mn` that splits the session's
one starting pane; add a function, name it in the column. Nothing is special
about `claude` or `nvim` — they are the commands the two shipped layouts happen
to send. `SIDEBAR_WIDTH` in `mn` sets the width.

## Worktrees

`M-n` prompts for a task and a branch name (pre-filled from the task, edit it as
you like), then makes a worktree under `~/.mullion/worktrees/<project>/<branch>`
and a session named `<project>/<branch>`. It shows up indented under its project
in the sidebar. The main checkout stays a top-level row and is never touched by
any of this — plenty of projects never need a worktree at all.

Removing one lives in the `M-Space` menu, behind a confirmation that tells you
how many uncommitted files you are about to destroy.

### Telling mn how to set a project up

Optional. Without it a worktree is just a branch and a checkout, which is all
some projects need. With it, drop a `project.sh` in either

- `<repo>/.mullion/project.sh` — committed, travels with the project
- `~/.config/mullion/<project>/project.sh` — personal, nothing in the repo

and give it up to three subcommands. All three are optional:

| | when | cwd |
|---|---|---|
| `setup` | once, when the worktree is created | the new worktree |
| `dev` | every time the worktree's session is built | the worktree |
| `teardown` | once, after the session is dead and before git deletes anything | the main checkout |

It is one file rather than three so that the facts they share — the database
names, here — are written once:

```bash
#!/usr/bin/env bash
set -euo pipefail
db="sofia_${MN_SLUG}"
dbs=("${db}_development" "${db}_development_cable" "$db")

case "$1" in
  setup)
    cp "$MN_REPO"/config/master.key config/
    sed "s/sofia_development/${db}_development/g" \
        config/database.yml.example > config/database.yml
    bundle check || bundle install
    bin/rails db:prepare
    printf 'PORT=%s\nHOST=0.0.0.0\n' "$MN_PORT" >> "$MN_ENV"
    ;;
  dev)      exec hivemind --port "$PORT" Procfile.dev ;;
  teardown) for d in "${dbs[@]}"; do drop_db "$d"; done ;;
esac
```

The environment, seven variables:

| | |
|---|---|
| `MN_REPO` | the main checkout |
| `MN_WT` | this worktree |
| `MN_BRANCH` | the branch |
| `MN_SLUG` | the branch as a safe identifier — `feat/x` → `feat_x`, bounded to 40 chars so what you append to it still fits MySQL's 64 |
| `MN_PORT` | a free port, allocated once and kept for the life of the branch. Mentioning it here is how a project asks for one; projects that never do get none, and no port badge in the sidebar |
| `MN_ENV` | append `KEY=value` lines here |
| `MN_TASK` | the task you typed |

`$MN_ENV` is the whole output contract. Whatever `setup` writes there becomes the
environment of `dev` — which is why the port never has to be spliced into a
command string, and why `dev` is a plain command rather than a shell one-liner.

### What the sidebar shows

The port mn allocated, and a mark when there is something to say:

```
sofia
    feat-clear-meadow  :3007
  ~ fix-invoice-round  :3008     ~ still running setup
  ! spike-new-nav      :3009     ! setup failed
herdr
```

None of this is pushed. It is read off `~/.local/state/mullion/<project>/<branch>/`
when the row is drawn, so there is nothing to publish and nothing to re-publish
after a restart.

### When setup fails

The worktree is left exactly as it is, marked `!`, with the failure on screen in
its dev pane and a shell sitting in the checkout. Fix whatever it was and run
`mn provision <project>/<branch>` in that pane.

### Teardown

`mn` owns the whole lifecycle, so the order is right by construction: the session
dies first (taking the dev server with it, since it is a pane process), then
`teardown` runs, then git removes the checkout and deletes the branch. An
unmerged branch is kept, and mn tells you the command to force it.

Delete a worktree behind mn's back and its databases would outlive it, so `mn`
sweeps for that on startup: any worktree in the state dir whose checkout is gone
gets its `teardown` run and its branch deleted. That sweep is why none of this
needs event hooks, locks, or re-fire guards.

## How it works

Two tmux **servers**, not two sessions:

- `mn-chrome` — one window, `[sidebar | view | issues]`, never rebuilt. That is
  why the sidebar cannot lose its width. The issues pane is optional and starts
  absent.
- `mn-work` — one session per project, and one per task worktree, named
  `<project>/<branch>`. The view pane is just a client attached to this server.

Picking in the sidebar runs `tmux -L mn-work switch-client -t <session>`. The
inner client changes session, the outer layout is untouched.

Two `set-hook`s (`client-attached`, `window-resized`) run `mn pin`, which
re-applies both sidebar widths, because tmux otherwise rescales panes
proportionally whenever a client attaches or the terminal resizes. `pin` holds
off while the window is zoomed: resizing a pane that zoom has hidden drops the
zoom, so without that guard a terminal resize would eject you from `M-z`.

Every outer pane is addressed by `#{pane_id}`, held in `@sb_pane`, `@view_pane`
and `@rsb_pane`. Indexes are not usable here: hiding a pane and putting it back
renumbers them without moving anything, so `ui:main.0` stops being the sidebar
and starts being the view.

## The issues sidebar

`M-i` toggles a right-hand pane listing the current project's ten newest open
issues, newest first. Enter opens the one under the cursor in a browser.

It needs [`gh`](https://cli.github.com) on `PATH`, authenticated, and an `origin`
remote pointing at github.com. A project without one shows `(no open issues)` and
costs no API call. `gh` resolves the repo from the checkout itself, so both the
SSH and HTTPS remote forms work.

The answer is cached under `~/.local/state/mullion/<project>/issues` for five
minutes, because walking the left sidebar re-renders this one on every row. A
failed fetch keeps the previous answer rather than blanking the pane. Worktrees
show their project's issues, since they share its remote.

Hiding it moves the pane to a detached window rather than killing it, so the
pane, its id and the process inside it all survive, and the outer window is
still never rebuilt.

## Keys

The outer server has **no prefix**. Its root key table is the app's keymap;
every key it does not bind falls through to the inner server and to the programs
in the panes. `C-b` is therefore the only prefix in the system, and it belongs to
the project you are working in.

| Key | Does |
|---|---|
| `M-Space` | command menu — including removing a worktree |
| `M-n` | new task worktree |
| `M-q` | detach |
| `M-p` | jump to any row, without leaving the view pane |
| `M-1`…`M-9` | jump to the *n*th row in the sidebar |
| `C-h/j/k/l` | move between the project's panes; at an edge, step out to a sidebar |
| `C-S-h` / `C-S-l` | narrow / widen the sidebar |
| `M-z` | zoom the view over both sidebars |
| `M-i` | show / hide the issues sidebar |
| `C-b` … | plain tmux, inside the project |

Making a worktree is one key; destroying one is only in the menu.

`C-hjkl` is delegated to the inner server (`mn nav h`) rather than handled
outright. It walks the project's own panes first and only steps out to a sidebar
when there is nothing further that way. Edges are detected explicitly, with
`#{pane_left}` and `#{pane_at_right}`, because tmux's own `select-pane -L` wraps
around instead of stopping.

`C-S-*` only exists because kitty can encode it. tmux ships no `extkeys` entry
for any terminal, so `mn` adds one (`xterm-kitty:extkeys`) alongside
`extended-keys on`. Attach from a different terminal and those two keys quietly
do nothing; add its `TERM` to the same line if it speaks CSI-u.

All of this is set on the `mn-chrome` server only. Your
`~/.config/tmux/tmux.conf` is untouched.

## Status bar

The outer status bar is the app's; the inner one is off, because it duplicated
the sidebar and redrew on every switch. The project name comes from a `@project`
user option that `switch_to` sets, so nothing is polled and nothing flickers.

The cost is that a project's *window* list is no longer visible. If you start
making windows inside a project with `C-b c`, drop the `work set -g status off`
line in `ensure_work`.

## License

0BSD.
