# mullion

A tmux sidebar that never moves.

A mullion is the vertical bar dividing a window into panes. This is one: a
permanent sidebar listing your projects, and a main view that swaps between them
without the sidebar ever losing its width, its scroll position, or its place.

It is about 1520 lines of bash and no daemon.

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

The left pane lists your projects and the task worktrees under them. Both
sidebars are lists rather than prompts, so they take the keys you would expect:
`j`/`k` or `↑`/`↓` to move, `g`/`G` for the ends, `Enter` (or a double-click,
mouse is on) to swap the right pane to that one, and `/` when you do want to
filter, until Esc. The sidebar keeps its width across switches, reattaches, and
terminal resizes.

`M-n` makes a new task worktree: a branch, a checkout, a port if the project
asks for one, whatever setup that project needs, and a coding agent already
holding the task you typed.

`M-i` opens a second sidebar on the right listing the project's ten newest open
GitHub issues, `Enter` on one of them opens it in a popup you can start a
worktree from, `w` starts that worktree without the popup, `n` files a new
issue, and `M-z` zooms the view over both sidebars. Under the issues, if the
checkout names a Basecamp project, a pane lists the cards assigned to you there,
and `w` on one starts a worktree whose agent turns the card into a GitHub issue.
Both sidebars toggle back to exactly the widths they had.

## Install

Requires **tmux 3.2+** (developed on 3.7) and **fzf 0.74+**, which is where the
`result-final` event arrived, the one that keeps the sidebar's footer in step
with its rows; the rest of it wants `--gutter` from 0.66, the `resize` event and
`$FZF_COLUMNS` from 0.46, `--no-input` from 0.59, and `--footer` and the
`bg-transform-*` actions from 0.63 (tested on 0.74.3).

[`gh`](https://cli.github.com) is optional, and only for the issues sidebar, the
pull request a worktree row shows, and making a worktree from one. That last one
wants `gh` 2.98+, the release `pr checkout --worktree` shipped in.

The `basecamp` CLI (37signals' own, 0.10 here) is optional too, and only for the
pane under the issues. It has to be logged in, and a checkout opts in with
`basecamp config project`; see the basecamp section below.

```bash
git clone https://github.com/danielsvane/mullion ~/projects/mullion
cd ~/projects/mullion
./mn                                # writes projects.conf from the template
$EDITOR projects.conf
./mn
ln -s "$PWD/mn" ~/.local/bin/mn     # optional
```

## Projects

`projects.conf` is one project per line, just the path:

```
~/projects/api
~/projects/frontend
```

Each line becomes an inner tmux session named after the directory, holding one
shell in the checkout. What else it holds is the project's own business: give it
a [`project.sh`](#telling-mn-how-to-set-a-project-up) with a `layout` phase and
split the panes there. `mn` sends no `claude` and no `nvim` of its own to a main
checkout. `SIDEBAR_WIDTH` in `mn` sets the sidebar's width.

`hide project…` in the `M-Space` menu takes a project back off the list. It
comments the line out of `projects.conf` and kills the session, and that is all
it does. The checkout is yours (`mn` never made it and never deletes it), so
nothing on disk is touched. Uncomment the line to bring the project back. A
project with task worktrees still under it is refused, and told which ones:
those have a branch and a `teardown`, so they stay the menu's
`remove worktree…`.

## Worktrees

`M-n` prompts for a task, then for a title and a branch name, and makes a
worktree under `~/.mullion/worktrees/<project>/<branch>` with a session named
`<project>/<branch>`. The title and the branch are named from the task by
claude (one haiku call, five seconds or so, while the dialog says `naming…`),
and both are offered for editing before anything is made. Without `claude`,
offline, or past ten seconds, the title is the task line and the branch a slug
of it, which is what the dialog used to offer. The worktree shows up indented
under its project in the sidebar, headed by the title. The branch starts at the
main checkout's last commit, so the prompt says how many uncommitted files are
staying behind. The main checkout stays a top-level row and is never touched by
any of this — plenty of projects never need a worktree at all.

`w` in the issues popup is that same dialog with the task already filled in as
`#24 <the issue title>`. The number stays out of the title and the branch, so
the row says what the work is; it lives in the task line, which is where it
matters.

That `#24` is not decoration. A task that starts with `#<number>` makes the
agent open holding the issue — its title, its URL and its whole description,
fetched when the pane starts, so an edit made since you created the worktree is
included. Type one by hand into `M-n` and you get the same thing. A pull request
number works too, since github numbers both from one sequence. The task field
stays a single line because it is the agent's opening prompt, and the title and
branch are named from it, so the issue travels this way rather than in the
field. Without `gh`, or on a number that is neither, the agent just gets the
line you typed.

`bc#<id>` is the same thing for a Basecamp card, and it is what `w` in the
basecamp pane fills in. The agent opens holding the card as the CLI renders it,
its comments, its attachments downloaded into the worktree's state directory,
and `qualify.md` from the directory `mn` runs from, which says what to do with a
report written by a non-technical user: read the screenshots and the code, ask
what is unclear, file an English GitHub issue with `gh issue create`, comment the
issue's URL back on the card, and stop there until you ask for the fix. Edit
`qualify.md` to change those instructions. Without the CLI the agent gets the
line alone, like an issue number without `gh`.

Removing one lives in the `M-Space` menu, behind a confirmation that tells you
how many uncommitted files you are about to destroy.

### From a pull request

`M-Space` then `P` lists the project's open pull requests in a popup, up to a
hundred of them, newest first: the number, whether it is a draft, who opened it,
the title. Type to filter, which is what you want on a repo with sixty of them
open, then `Enter`. The one field you fill in is the name the row is headed with,
pre-filled with the pull request's title.

The branch is the pull request's own and is not offered for editing: the badge in
the sidebar is looked up by branch name, and a push has to land on the ref the PR
is for. `gh` does the checkout, so the fetch, the local branch and its tracking
config are its business rather than mn's, and a branch you already have locally
is reused and brought up to date instead of refused. The row shows its
`#9465 draft` badge as soon as the sync behind it lands.

The agent opens with no task here, where `M-n`'s opens holding the one you typed.
You pulled the branch in to get at work that already exists, so there is nothing
to instruct it with, and the number is on the row already. It is still briefed
on where it is, like every agent `mn` starts. See below.

The list is fetched when you open the popup, not cached like the issues pane, on
the same grounds: one keypress by one person can afford half a second, and
nothing here is redrawing a row.

Removing it is `M-Space` then `x`, like any other worktree. That deletes the
local branch and nothing else, so the pull request and the branch on the remote
are left alone.

### What the agent is told

Every agent `mn` starts is briefed before it is seeded: `briefing.md`, from the
directory `mn` runs from, is appended to claude's system prompt under one line
naming the session it is in. The file says what mullion is, that the branch
started at the main checkout's last commit, and how to hand work off:

```bash
mn new <project> "<task>" <branch> "<title>"
```

That is `M-n` with nothing to ask. The worktree is made, its agent starts
holding the task, the row appears in the sidebar, and your view stays where it
is, since the agent that ran it is the one you were talking to. So an agent that
finds a second problem while fixing the first can file it and hand it over
rather than fix it on the wrong branch; the branch and the title are its to
name, and the number stays out of both. Edit `briefing.md` to change what agents
are told. A checkout without the file briefs nobody.

A project's own agent is briefed the same way when its `layout` phase starts
claude through `$MN_AGENT` rather than by name. See below.

### Telling mn how to set a project up

Optional, and the only place a project says anything about itself: what its
panes are, and what a worktree needs before it can run. Without it a project
session is one shell in the checkout and a worktree is just a branch and a
checkout, which is all some projects need. With it, drop a `project.sh` in
either

- `<repo>/.mullion/project.sh` — committed, travels with the project
- `~/.config/mullion/<project>/project.sh` — personal, nothing in the repo

and give it up to five subcommands. All five are optional:

| | when | cwd |
|---|---|---|
| `layout` | every time the project's own session is built | the checkout |
| `layout-task` | every time a worktree's session is built | the worktree |
| `setup` | once, when the worktree is created | the new worktree |
| `dev` | every time the worktree's session is built | the worktree |
| `teardown` | once, after the session is dead and before git deletes anything | the main checkout |

It is one file rather than five so that the facts they share — the database
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

#### Laying the panes out

The session already exists and holds one pane. Split it however you like:
`$MN_SERVER` is the inner tmux server's socket name and `$MN_SESSION` the
session, so a layout is real tmux commands rather than a format `mn` has to
parse. This one is the agent on the left, `nvim` above a spare shell on the
right, which is what `mn` used to do for every project whether it wanted it or
not. `$MN_AGENT` is `claude` started through `mn`, so it arrives briefed on
where it is; send `claude` by name and it is not:

```bash
layout)
  tm() { tmux -L "$MN_SERVER" "$@"; }
  main=$(tm list-panes -t "$MN_SESSION" -F '#{pane_id}')
  right=$(tm split-window -h -t "$main" -c "$MN_REPO" -P -F '#{pane_id}')
  tm split-window -v -l 25% -t "$right" -c "$MN_REPO"
  tm send-keys -t "$main" "$MN_AGENT" C-m
  tm send-keys -t "$right" nvim C-m
  tm select-pane -t "$main"
  ;;
```

Three things worth knowing:

- **Pass `-c`.** A split run from outside the session starts in `mn`'s own
  directory, not the session's.
- **Split before you send.** A detached session is 80x24 until you first look at
  it, so a command started before the splits redraws at a size it will not keep.
- **Address panes by `#{pane_id}`** — the `%12` that `-P -F` hands back — not by
  index.

A worktree's `layout-task` is the same job with two commands to place.
`$MN_AGENT` starts the coding agent, already holding the task the worktree was
made for; `$MN_PROVISION` runs `setup` the first time and then `dev`, in a pane
where you can read their output and which is left standing in the checkout if
`setup` fails. Both arrive as command lines to send, so the quoting stays
`mn`'s:

```bash
layout-task)
  tm() { tmux -L "$MN_SERVER" "$@"; }
  main=$(tm list-panes -t "$MN_SESSION" -F '#{pane_id}')
  right=$(tm split-window -h -l 45% -t "$main" -c "$MN_WT" -P -F '#{pane_id}')
  tm send-keys -t "$right" "$MN_PROVISION" C-m
  tm send-keys -t "$main"  "$MN_AGENT" C-m
  tm select-pane -t "$main"
  ;;
```

Leave `layout-task` out and `mn` lays a worktree out itself, much like that but
with a spare shell under the dev server. Leave `layout` out and the project's
session stays the one shell it started as.

#### The environment

| | | in |
|---|---|---|
| `MN_REPO` | the main checkout | all |
| `MN_WT` | the checkout this session sits in — the worktree, or the main checkout in `layout` | all |
| `MN_BRANCH` | the branch | all, empty in `layout` |
| `MN_SLUG` | the branch as a safe identifier — `feat/x` → `feat_x`, bounded to 40 chars so what you append to it still fits MySQL's 64 | all, empty in `layout` |
| `MN_PORT` | a free port, allocated once and kept for the life of the branch. Mentioning it here is how a project asks for one; projects that never do get none, and no port badge in the sidebar | all, empty in `layout` |
| `MN_TASK` | the task you typed | all, empty in `layout` |
| `MN_SERVER` | the inner tmux server's socket name | the two layouts |
| `MN_SESSION` | the session your splits target | the two layouts |
| `MN_AGENT` | the command line that starts the agent, briefed on where it is and, in a worktree, holding the task | the two layouts |
| `MN_PROVISION` | the command line that runs `setup`, then `dev` | `layout-task` |
| `MN_ENV` | append `KEY=value` lines here | `setup`, `dev`, `teardown` |

A name a phase has no value for is set empty rather than left unset, so a script
with `set -u` and a line or two at the top level does not trip over one it does
not need.

`$MN_ENV` is the whole output contract. Whatever `setup` writes there becomes the
environment of `dev` — which is why the port never has to be spliced into a
command string, and why `dev` is a plain command rather than a shell one-liner.

### What the sidebar shows

Every row is three lines. A project is its name and the pull request for the
branch it has checked out, unless that is the repo's default branch; a worktree
is the task you made it for and, under it, its own pull request and the port mn
allocated. The last line is what the session is running: how many processes are
alive in its panes, and what they are holding. A rule separates one project's
rows from the next one's, and a blue bar down the left of one of them is the
session the view is on.

```
 * herdr
   -
   6 processes                   288M
─────────────────────────────────────
 ? sofia
   #9466 open
   9 processes                   231M
 ? Clear the meadow before the fros…
   #9412 open                   :3007
   11 processes                  283M
▎* Round the invoice at the end of…
▎  #9470 draft                   3008
▎  10 processes                  1.4G
 ! Spike the new navigation
   -                             3009
   8 processes                   164M
─────────────────────────────────────
 44 processes                    2.3G
```

The bar is where you are. The cursor is the quieter of the two: walking the list
only tints the row under it a shade lighter, and Enter is what moves the bar. The
rule stays out of that tint, since it belongs to neither of the rows it sits
between.

The task reads as the worktree's headline, in the same colour as a project name
without the bold. Nothing indents under a project, so a worktree's mark and its
project's line up in the same column and the rule is what does the grouping.

Each sidebar's first row names it, `sessions` on the left and `issues` on the
right. It sits in border grey until that pane has the keyboard, when it turns
blue and grows the same bar the active session has. If neither is lit, the keys
are going to the view.

The headline is the title, named from the task alongside the branch, so the row
says what the work is rather than what git calls it. The two columns before it
are the one thing about that row worth knowing from across the room:

| | |
|---|---|
| `?` | the agent is waiting for you — a permission prompt, or anything else it has put on screen |
| `+` | the agent finished a turn while you were looking somewhere else |
| `*` | the agent is working |
| `~` | setup is still running |
| `!` | setup failed |

The marks queue in that order, except that `!` outranks all of them, because its
pane is holding output you have to go and read. Blank means an agent you are
already caught up with, one that has not been asked for anything yet, or no
agent in that session at all. The row is coloured to match, amber and orange and
the blue of a live port, but the mark is what carries the meaning. The PR spells its state out for the
same reason — `open`, `draft`, `merged`, `closed`, or `-` for a branch with no PR
yet. A project sitting on its default branch gets `-` as well. A repo old enough
has some fork-era pull request made from `master`, and `#263 closed` from 2016
tells you nothing about the checkout. See [Colours](#colours).

A project's row takes the same two columns, but only the agent half of them: mn
provisions a worktree and never a main checkout, so `~` and `!` belong to a
worktree's row alone.

The third line is what the session is costing you. A pane running claude is
rarely one process: the shell, claude itself, and one more for every MCP server
it starts, so a session with an agent in it and nothing else sits at a floor of
about six, and the number above that is the editor, the dev server and whatever
those have forked.

The megabytes beside it are the proportional set size, which is the kernel's own
answer to "whose memory is this": a page mapped by four claude processes
counts a quarter towards each, so the rows sum to what is really resident
rather than counting the same shared page nine times. Claude is nearly all of it — 150 to
300MB per session against a megabyte or two for everything else — so the two
numbers say different things, and the count is the one that moves when you start
a dev server. Anything that puts itself in a session of its own is in neither.
Both are as fresh as the last draw, like the port badge, so `ctrl-o` is what
asks again.

At the bottom of the sidebar, under a rule, the same two numbers for every
session at once.

Agent state comes from Claude Code itself, which keeps a file per live session
under `~/.claude/sessions` saying whether it is idle, working or waiting on you,
and which tmux session it is in. That name is exactly what a row is keyed by, so
an agent you left in a main checkout marks its project's row the same way. mn
reads those as it draws a row, the same way it reads the port. Nothing is
installed in your claude config and no hook has to fire. A session running
something other than claude simply has no file, so its rows stay blank.

`+` is the one state claude does not publish, and it is idle with two more
questions asked of the same file. The file records when the status last changed,
so an idle session whose last change came long after it opened has been busy and
come back: it finished a turn. And tmux knows when the view was last on that
session, because switching is what every pick does. Later than that and the turn
finished while you were somewhere else, which is the thing the desktop
notification tells you about a session you are in. The row you are on never
wears it, since you are looking at it — but leave one without replying and it
goes back to being a turn you have not answered.

Since nothing announces a prompt appearing, the outer status bar doubles as the
clock: tmux re-runs a command in it every ten seconds while you have mullion
open, and the sidebar redraws only if one of those words actually changed. Move
the keyboard between panes or press `C-r` and you get the same refresh sooner.

The port keeps its colon for as long as something is answering on it, and takes
the same blue as an open PR. `:3007` is a server you can open, `3007` is a
number held for the branch with nothing on it. mn reads the kernel's table of
listening sockets as it draws the row, so no dev server has to report in.
Nothing announces one starting either, so the badge catches up the next time the
keyboard moves between panes, or on `C-r`. A project's row has no port badge: mn
allocates one per worktree, and runs the project's script for a worktree only.

None of this is pushed. It is read off `~/.local/state/mullion/<project>/<branch>/`
when the row is drawn — plus the kernel's socket table for the port and
`~/.claude/sessions` for the agent — so there is nothing to publish and nothing
to re-publish after a restart. The PR is the one thing that cannot come off disk: `gh` answers
for it in the background when mn starts and whenever you switch sessions, at
most once every five minutes per project, into
`~/.local/state/mullion/<project>/prs` — one line per branch, the main
checkout's alongside its worktrees'. Drawing a row never waits on github, and
a sync that fails offline keeps the badges it had.

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

## Colours

GitHub Dark, colourblind flavour. The `C_*` block at the top of `mn` is the
whole theme, and every colour in the program comes from it:

| | |
|---|---|
| `C_INSET` `#010409` | sidebars and the status bar, a step darker than the terminal, so the view is what you are working in |
| `C_RAISED` `#151b23` | a popup, a step lighter, so it reads as a card over the top |
| `C_ROW` `#21262d` | the row the cursor is on |
| `C_LINE` `#3d444d` | pane borders and the rules inside a popup |
| `C_TEXT` `#f0f6fc` | a project name, a worktree's task, an issue title |
| `C_MUTED` `#9198a1` | an issue body, a closed pull request |
| `C_DIM` `#6e7681` | an idle port, a working agent, hints, separators |
| `C_KEY` `#4493f8` | the session you are in, and any key you can press |
| `C_SETUP` `#d29922` | setup still running, or an agent waiting on you |
| `C_BROKEN` `#f0883e` | setup failed |
| `C_PR_OPEN` `#58a6ff` | an open pull request |
| `C_PR_DONE` `#b7bdc8` | a merged one |
| `C_LIVE` `#58a6ff` | a port with a server answering on it, and an agent that has finished a turn |

Two rules hold it together. In this flavour success is blue and danger is
orange, so nothing puts its meaning in a red/green pair, and every coloured
state keeps a mark that survives a greyscale screenshot. And the accent is spent
only on the session you are in and on keys you can press, which is why a focused
pane border is the brighter of two greys instead of blue, and why the sidebar
cursor is a tint rather than a second blue thing.

Change a value and every surface follows: tmux reads the hex directly, fzf takes
it in `--color`, and `sgr()` turns it into an escape for the parts `printf`
writes. It wants a terminal that speaks 24-bit colour, which tmux detects from
terminfo (kitty, and most others since about 2018).

## How it works

Two tmux **servers**, not two sessions:

- `mn-chrome` — one window, `[sidebar | view | issues over basecamp]`, never
  rebuilt. That is why the sidebar cannot lose its width. The two right-hand
  panes are optional and start absent.
- `mn-work` — one session per project, and one per task worktree, named
  `<project>/<branch>`. The view pane is just a client attached to this server.

Picking in the sidebar runs `tmux -L mn-work switch-client -t <session>`. The
inner client changes session, the outer layout is untouched.

Two `set-hook`s (`client-attached`, `window-resized`) run `mn pin`, which
re-applies both sidebar widths, because tmux otherwise rescales panes
proportionally whenever a client attaches or the terminal resizes. `pin` holds
off while the window is zoomed: resizing a pane that zoom has hidden drops the
zoom, so without that guard a terminal resize would eject you from `M-z`.

Every outer pane is addressed by `#{pane_id}`, held in `@sb_pane`, `@view_pane`,
`@rsb_pane` and `@bc_pane`. Indexes are not usable here: hiding a pane and putting it back
renumbers them without moving anything, so `ui:main.0` stops being the sidebar
and starts being the view.

## The issues sidebar

`M-i` toggles the right-hand sidebar, whose upper pane lists the current
project's ten newest open issues, newest first. Opening it moves the keyboard there, since reading an
issue is what you pressed the key for; closing it hands the keyboard back to the
view. `Enter` opens the issue under the cursor in a popup:

```
  #24  Rather confusing chat from meshcore

  It seems like people are replying to names/messages I cant see? Is
  there some info in the chat window we are not showing?

  w  new worktree      o  open in browser      q  close
```

`w` starts a task worktree seeded from the issue, `o` opens it on github, and
any other key closes. The list comes from the cache; the description is a live
`gh issue view`, on the grounds that one keypress can afford half a second and
drawing a row cannot.

`n` files one instead, in a popup of the same shape:

```
  new issue in mullion

  title: Port badge stays dim after a restart
  body:  ctrl-d on an empty line ends it, ctrl-c abandons
  The badge only refreshes on a draw, so a server that comes
  up later reads idle until something else redraws the row.
```

The title is one line and an empty one abandons; the body is every line after it
until `ctrl-d` on an empty one, and may be empty. Press `ctrl-d` on a line with
text on it and the terminal hands that text over rather than ending the input,
so it takes one more press to get out; the line is kept either way. `gh` prints
its own errors into the popup, so a rejected issue says why before it waits for
a key. A filed one deletes the cache and fzf refetches as its next action, so
the new row is on screen as the popup closes.

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

### Basecamp cards

Under the issues sits a second pane, `basecamp`, listing the cards and to-dos
assigned to you on the Basecamp project this checkout belongs to: the title over
the column the card sits in, which on a bug board is its severity. Which project
that is comes from the Basecamp CLI's own repo config, so in the main checkout:

```bash
basecamp config project --project "DBP master"   # writes .basecamp/config.json
```

Add `.basecamp/` to the repo's or your global gitignore. The CLI resolves its own
`--project` from the same file, so its commands work from the checkout without
one. A checkout with no such file shows `(no basecamp project)` and costs no
call.

`Enter` opens the card in a popup, its description with the HTML taken off and
each attachment named in brackets; `o` opens it in the browser through
`xdg-open`; `w` starts a worktree whose task is `bc#<id> <title>`, so its agent
opens with the card, its comments, its screenshots and `qualify.md`, as described
above next to `#<number>`. `w` inside the popup is the same key. There is no
`n`: cards are written on the Basecamp side.

Assignments are yours across every Basecamp project, so `basecamp assignments`
runs once for all of them and its answer is cached under
`~/.local/state/mullion/basecamp` for the same five minutes as the issues, and
each project's pane filters that one file. The issues pane is as tall as its
list can get and this pane takes the rest of the column, on every resize. `C-j`
and `C-k` move between the two, and `h` steps into the view from either.

## Keys

The outer server has **no prefix**. Its root key table is the app's keymap;
every key it does not bind falls through to the inner server and to the programs
in the panes. `C-b` is therefore the only prefix in the system, and it belongs to
the project you are working in.

| Key | Does |
|---|---|
| `M-Space` | command menu: a worktree from a pull request, removing a worktree, hiding a project |
| `M-n` | new task worktree |
| `M-w` | close the active pane in the project |
| `M-q` | detach |
| `M-p` | jump to any row, without leaving the view pane |
| `M-1`…`M-9` | jump to the *n*th row in the sidebar |
| `C-h/j/k/l` | move between the project's panes; at an edge, step out to a sidebar, and `C-j`/`C-k` there move between the two right-hand panes |
| `C-S-h` / `C-S-l` | narrow / widen the sidebar |
| `M-z` | zoom the view over both sidebars |
| `M-i` | show / hide the right-hand sidebar, issues over basecamp |
| `C-b` … | plain tmux, inside the project |

Making a worktree is one key. Destroying one is `x` on its row in the sidebar,
or the menu; either way it counts the files it is about to destroy and waits for
a `y`.

`M-w` is `C-b x` and its `y` in one key: it closes the project's active pane
without asking. It stops at the session's last pane, since closing that one
would take the view pane with it. The view *is* a client attached to that
session, and tmux detaches a client whose session is destroyed. Getting rid of a
whole session is `x`, which removes the worktree behind it too.

Inside a sidebar the keys belong to the list, since both panes run with fzf's
input line hidden: `j`/`k`, `g`/`G`, `Enter`, `/` to filter and Esc to stop,
`C-r` to redraw, and `l` (left sidebar) or `h` (issues, basecamp) to step into the view.
The cursor is a bar in the accent colour over a highlighted row.

The rest of a sidebar's keys act on the row under the cursor, so the common
things do not go through the menu:

| Pane | Key | Does |
|---|---|---|
| left | `n` | new task worktree in that row's project |
| left | `x` | remove that worktree (a project row says so and stops) |
| left | `X` | hide that project |
| issues | `w` | task worktree seeded from that issue |
| issues | `o` | open that issue on github |
| issues | `n` | file a new issue |
| basecamp | `w` | task worktree whose agent turns that card into a GitHub issue |
| basecamp | `o` | open that card in the browser |

`w` takes the title from the same cache the row was drawn from, so it costs no
API call; it is the `w` in the issue popup without having to open the issue.

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
the sidebar and redrew on every switch.

It holds the keymap and nothing else. Which keys it names follows the pane you
are in, and in the left sidebar the row you are on as well, so `x` appears over
a worktree and not over a project. Both of those are tmux formats rather than
anything mn polls: a status format's `#{pane_id}` is the *active* pane, and a
row is a worktree exactly when its id has a `/` in it. Only the cursor has to be
reported, which the sidebar does on fzf's `focus` event, at about 2ms a move.

The cost is that a project's *window* list is no longer visible. If you start
making windows inside a project with `C-b c`, drop the `work set -g status off`
line in `ensure_work`.

## License

0BSD.
