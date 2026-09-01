# mullion

A tmux sidebar that never moves.

A mullion is the vertical bar dividing a window into panes. This is one: a
permanent sidebar listing your projects, and a main view that swaps between them
without the sidebar ever losing its width, its scroll position, or its place.

It is about 220 lines of bash and no daemon.

```bash
./mn          # start + attach
./mn stop     # tear both servers down
```

The left pane lists your projects; `↑`/`↓` and `Enter` (or a double-click, mouse
is on) swap the right pane to that project. The sidebar keeps its width across
switches, reattaches, and terminal resizes.

## Install

Requires **tmux 3.2+** (developed on 3.7) and **fzf 0.34+**.

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

A layout is just a `layout_<name>` function in `mn` that splits the session's
one starting pane; add a function, name it in the column. Nothing is special
about `claude` or `nvim` — they are the commands the two shipped layouts happen
to send. `SIDEBAR_WIDTH` in `mn` sets the width.

## How it works

Two tmux **servers**, not two sessions:

- `mn-chrome` — one window, `[sidebar | view]`, never rebuilt. That is why the
  sidebar cannot lose its width.
- `mn-work` — one session per project. The view pane is just a client attached
  to this server.

Picking in the sidebar runs `tmux -L mn-work switch-client -t <session>`. The
inner client changes session, the outer layout is untouched.

Two `set-hook`s (`client-attached`, `window-resized`) re-pin the sidebar,
because tmux otherwise rescales panes proportionally whenever a client attaches
or the terminal resizes. They carry a literal width rather than calling back into
`mn`, so the common path stays inside tmux; `set_width` rewrites both hooks when
the width changes.

## Keys

The outer server has **no prefix**. Its root key table is the app's keymap;
every key it does not bind falls through to the inner server and to the programs
in the panes. `C-b` is therefore the only prefix in the system, and it belongs to
the project you are working in.

| Key | Does |
|---|---|
| `M-Space` | command menu |
| `M-q` | detach |
| `M-p` | switch project, without leaving the view pane |
| `M-1`…`M-9` | jump to the *n*th project in the sidebar |
| `C-h/j/k/l` | move between the project's panes; `C-h` at the left edge enters the sidebar |
| `C-S-h` / `C-S-l` | narrow / widen the sidebar |
| `C-b` … | plain tmux, inside the project |

`C-hjkl` is delegated to the inner server (`mn nav h`) rather than handled
outright, and the left edge is detected with `#{pane_left}` — tmux's own
`select-pane -L` wraps around instead of stopping.

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
