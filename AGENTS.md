# AGENTS.md — orientation for coding agents

Read this before changing anything. `README.md` is the user-facing reference
(install, keys, layouts) — don't duplicate it here; update it when behaviour
changes.

## What this repo is

`mn` — about 1600 lines of bash, no daemon, no dependencies beyond tmux, fzf and
git, plus `gh` if you want the issues sidebar or a worktree from a pull request
(that one needs `gh` 2.98+, where `pr checkout --worktree` landed). fzf must be
0.46+: the sidebars
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
  every layer were `ctrl-o` and `ctrl-t`; `ctrl-o` is now spent on the redraw both
  sidebars answer, so `ctrl-t` is the one left. Inside a sidebar pane the plain
  letters are spent too, because `--no-input` hides fzf's input line and hands
  every printable key to the bindings: `j`, `k`, `g`, `G`, `/` and Esc in both
  panes, plus `n`, `x`, `X` and `l` in the left one and `n`, `w`, `o` and `h` in
  the issues one. The uppercase half of the alphabet is nearly all still free,
  and `n`/`x`/`X` are deliberately the same letters in the sidebar as in the
  `M-Space` menu, since the menu is where you learn them.

- **`~/.config/tmux/tmux.conf` is loaded by both servers.** The user's
  `bind -n C-h select-pane -L` is why `C-hjkl` has to be re-bound on the outer
  server and delegated with `mn nav`: the outer server sees every key first.
- **`projects.conf` is gitignored** — it holds real local paths. Edit
  `projects.conf.example`; `projects()` copies it on first run. `hide_project`
  is the only thing in `mn` that writes to it, and it *comments* the line rather
  than cutting it, because `projects()` already skips a `#`, so the path
  survives to be put back by hand. A line is *only* a path now — the layout
  column is gone, and `projects()`'s readers still drop a trailing field so an
  old conf that kept one names a project rather than an error. It refuses a
  project that still has task worktrees under it, which is what keeps every
  `<project>/<branch>` row
  resolvable through `repo_path`; and it switches the view off the session before
  killing it, since the view pane *is* a client attached to that session and the
  outer window is never rebuilt.
- **A session's panes belong to the project, not to `mn`.** `project.sh` gets a
  `layout` phase for the main checkout and a `layout-task` phase for a worktree,
  and runs tmux against `$MN_SERVER` and `$MN_SESSION` itself — the same
  argument as `provision`'s, that the control flow belongs in real bash rather
  than in a string `mn` composes. So `mn` sends no `claude` and no `nvim` of its
  own any more: `layout_dev` and `layout_agent` are gone with the config column
  that selected them, a project with no `layout` phase keeps the one shell
  `new-session` made, and `layout_task` survives as the *default* a worktree
  gets when its project lays none out. The two commands a worktree's panes need
  are handed over as `$MN_AGENT` and `$MN_PROVISION`, ready to `send-keys`, so
  the quoting of `$SELF` stays in one place. Every `MN_*` name a phase has no
  value for is exported *empty*, because a project script runs its top level
  before it reaches the arm — sofia's computes a database name from `$MN_SLUG`
  there, and under `set -u` an unset one would kill the layout.
  Whether a project defines a phase is a `grep` for `<phase>)`, because a `case`
  arm that is not there exits 0 having done nothing — running the hook cannot
  tell you whether it did, and the fallback has to know. Same
  declaration-by-grep as `wants_port`, with the same failure mode: the word in a
  comment costs you the default, which is visible on screen rather than silent.

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
  out already grouped — the sidebar's grouping is that sort, not a tree, and the
  rule between groups is drawn on the row where `%%/*` changes. The
  branch can hold a `/` of its own, since a PR's head branch is whoever's
  `name/thing` and is taken verbatim: so the project is `%%/*` and the branch is
  `#*/`, never the other way round, and only `dirslug` flattens it for a path.
  Measured end to end on `sofia/aske/page-header-breadcrumbs`.

- **Nothing pushes to the sidebar.** Rows are rendered from
  `~/.local/state/mullion/<project>/<branch>/meta` at draw time. The predecessor
  (`herdr-conductor`) pushed display tokens to an in-memory store and then
  needed two extra event hooks to re-push them after a restart. Anything that
  changes what a row says calls `sidebar_reload`; that is the whole mechanism.
  A resize is the one thing that does not need it, because fzf reloads itself on
  its own `resize` event.

- **Agent state is read, not received.** Claude Code writes a file per live
  session under `~/.claude/sessions` (`$CC_SESSIONS`) holding a `status` of
  exactly `idle | busy | waiting` and a `tmux` field whose first field is the
  inner session's own name — which is a sidebar row's id, a main checkout's
  included, since its session is named after the project. So `agent_states`
  answers `" <session>=<status> "` from one awk over the lot and `list_rows`
  glob-matches it, exactly as it does the ports; the mark cost no
  hook, no daemon and nothing installed in the user's claude config. This is why
  Orca's approach was not copied: it POSTs every claude hook event to a local
  daemon, and hooks cannot see an interrupt — **Esc mid-turn fires no `Stop`, no
  event at all**, so a hook-driven row sticks on "working" until the next
  prompt, where the file was back to `idle` within 3s of the same Esc. All
  measured against 2.1.259 with all 14 events logged.
  Four things to keep: the keys are read one at a time rather than in one
  pattern, because it is minified JSON whose key order is nobody's promise and
  an unparseable file has to cost a blank mark rather than a broken row; a pid
  with no `/proc` entry is dropped, because `kill -9` leaves a file behind with
  a frozen `busy` in it where SIGHUP (what `kill-session` sends) makes claude
  clean it up; `waiting` is asked for before `busy`, so a session holding two
  agents needs no merge rule; and it stays out of anything that draws a row that
  is not this file read.
  The fourth state, `done`, is mn's own reading of `idle` rather than anything
  claude publishes, and it is answered by the same awk: `statusUpdatedAt` later
  than `startedAt` means the session has been busy and come back (a session
  still sitting where it opened has the two 40ms apart, measured), and
  `session_last_attached` on the *inner* server is when the view was last on
  that row — `switch-client` updates it, measured on a probe, and a session
  never attached reads back empty, which is 0 and exactly right. A row whose
  turn finished later than that is `done`; the session with `session_attached`
  is never, because you are looking at it. That is the whole reason no `seen`
  file exists: tmux already keeps the timestamp, so the fourth state stays a
  read at draw time like the other three. Two traps in the awk: the inner
  server's list is the first file and needs a **sentinel line**, because
  `NR == FNR` cannot tell an empty first file from the start of the second (a
  dead `mn-work` would otherwise make every session line a claude file); and
  both timestamps are `+ 0`'d, because awk compares a `substr` against a sum as
  *strings*. `claude agents --json` is the documented interface for
  the same data and is a 128ms fork of a 216MB binary, so it can never be
  anywhere near a draw — if the file's shape ever changes, the fallback is a
  blank column, not a cache.

- **A pane program calls `mn`; `mn` never calls back with a payload.** The
  sidebar and the issues pane are small programs living in a pane, and the whole
  contract between them and `mn` is: print `<what you see>\t<id>` on stdout, and
  run an `mn` subcommand when the user picks something. The only thing `mn` ever
  sends outward is "redraw yourself" (`sidebar_reload`, `issues_reload`), which
  carries nothing. Keep it that way. The one thing that travels the *other* way
  is the left sidebar's `focus` binding, which puts the id of the row under the
  cursor into `@hover` so the status bar can name the keys that apply to it. That
  is still the pane talking outward and it still carries only an id, and it is a
  tmux client invocation with no shell in it: measured at 2.2ms a move, against
  5ms for fifty moves with no binding, so a held `j` stays well ahead of the key
  repeat. Nothing reads `@hover` but a status format. The moment `mn`
  broadcasts and panes subscribe you have `herdr-conductor` back: it pushed
  display state outward and then needed re-push hooks, locks and a dirty-file
  trick to order itself. Only 25 of `mn`'s lines mention fzf, and that is the
  property that makes the renderer replaceable.

- **The session the view is on gets the accent; the cursor only tints its row.**
  Those are two different facts and the loud one is the session, not the cursor:
  being *on* a row is a thing you are in the middle of doing, and being in a
  session is a thing that is true. So `list_rows` draws the accent itself, a
  `C_KEY` bar in column 0 of both of an item's lines, which fzf cannot do because
  it has no idea what `@project` says. It is a border, so it takes one column and
  no gap after it: `▎` paints only the left quarter of its cell, and the row's
  text starts in the next one; and fzf's own cursor gives up its pointer
  (`--pointer=''`) and its bold (`current-fg:regular`), keeping only the `C_ROW`
  background that `--highlight-line` paints across the row. The bar reads in
  greyscale because it is a shape rather than a colour, and it is the one
  multibyte glyph in a row because it sits outside every padded field — see the
  byte-precision rule below. `list_rows` reads `@project` once per render, and
  `switch_to` calls `sidebar_reload` because the bar has just moved.

- **A pane program renders and takes input. It never owns state.** The `meta`
  files are the truth and rows are built from them at draw time. A pane holding
  its own idea of what exists is two sources of truth, and it is what makes the
  next renderer swap expensive.

- **A project script's only output is `$MN_ENV`.** It writes `KEY=value`, mn
  sources that into the `dev` command's environment. Do *not* reintroduce a
  "compose a shell one-liner and inject it into a pane" step — that is what the
  `provision` subcommand exists to replace, and it is why nothing here has to
  quote a port into a command string.

- **A port belongs to a project that asks for one**, and asking means mentioning
  `MN_PORT` in the project script — `wants_port` greps for it. Nothing else in
  the contract says a project wants a port: sofia takes its port straight from
  `$MN_PORT` and never writes to `$MN_ENV` at all, so the other candidate rule,
  "keep the port only if `setup` wrote `PORT=` into `$MN_ENV`", would have freed
  a port hivemind was still listening on. `provision` re-checks the same
  predicate, which is what finally gives a worktree made before its project
  asked for a port one, on its next session build. And do not derive "has a
  provisioner" from a non-empty `PORT` again: that is what made a provisioner
  with no port skip `setup` altogether.

- **A task that starts with `#<number>` is an issue or a pull request**, and that
  prefix is the entire mechanism: `agent` fetches it and hands claude the body,
  the title and the URL alongside the task line. It is not stored, because the
  prompt field has to stay one editable line and the branch name is a slug of
  that line, so nothing longer can live there. The issues popup's `w` writes one;
  the PR picker deliberately does not, since a PR worktree seeds no agent at all
  (see below), but a number typed into `M-n` by hand still works. One call covers
  both kinds,
  because github numbers issues and PRs from a single sequence and `gh issue
  view` resolves either (verified against a real PR, body and `/pull/` URL and
  all), so the label in the prompt comes from the URL rather than from a second
  request. `gh` runs once, at pane start, outside anything that draws — see the
  cache rule below. Every failure (no `gh`, no such number, offline) falls
  through to the task alone.

- **A worktree from a PR takes the branch the PR is on, and `gh` does the
  checkout.** `gh pr checkout <n> --worktree <dir>` fetches, creates the local
  branch and sets its tracking config in one command, and by its own account
  sorts out a fork's push remote as well (that part is not measured here).
  Measured, on a same-repo PR: ~2s, it creates missing parents, it reuses a
  branch already there by fast-forwarding it, and on failure it exits 1 leaving
  no directory behind.
  Hand-rolling `fetch origin pull/<n>/head` gets the refspec right and the push
  config wrong, so don't. The branch name is the PR's own and is *not* offered
  for editing, unlike `M-n`'s: `pr_look` matches a badge by branch name and a
  push has to land on the ref the PR is for. `wt_build` is the half both paths
  share (port, meta, session, layout), and the checkout is the half that differs.

- **A PR worktree has a name where the others have a task, and its agent starts
  empty.** The two were one field until they had to come apart: a task seeds
  claude, and there is nothing to instruct an agent with when you have pulled a
  branch in to get at work that is already on it. So `meta` carries `SEED=no`,
  `agent` returns to a bare `exec claude` on it, and the field's prompt says
  `name:` rather than `task:` so the row's line does not read as an instruction
  nobody followed. An absent key means seed, which keeps every worktree made
  before the key existed working. It is written before `wt_build`, because
  `wt_build` starts the pane that reads it, and it beats a `#<number>` in the
  line: the number is on the row already, in the badge.

- **One palette, three renderers.** The `C_*` block at the top of `mn` holds
  GitHub Dark's colourblind flavour, copied from the values the user's OS theme
  already uses (`~/.config/gtk-4.0/github-dark-colorblind.css`). tmux takes the
  hex as it is, fzf takes it in `--color`, and `sgr()` turns it into an SGR
  escape for the things `printf` writes — `sgr_bg()` for the one row that has to
  paint its own background. The `E_*` vars are that conversion done
  once per process, because doing it per row is a fork per row. Do not add a
  literal colour anywhere else, and do not add a second palette for one surface.
  Two rules come with it: in this flavour success is blue and danger is orange,
  so **nothing may carry its meaning in a red/green pair** and every coloured
  state also has an ASCII mark (`~`, `!`, a port's colon) that survives a
  greyscale screenshot; and the accent (`C_KEY`) is spent only on the session the
  view is on and on keys you can press, which is why a focused pane border is the
  brighter of two greys rather than blue.

- **The status bar is the keymap, and it works out its own context.** The name
  and the project name came off it because neither ever changes and the sidebar's
  accent bar already says which session the view is on. Which keys it names is
  two questions tmux answers by itself, for no forks and no hook: a status
  format's `#{pane_id}` resolves against the *active* pane, so comparing it to
  `@sb_pane` and `@rsb_pane` names the section and tmux redraws the bar on
  `select-pane` unprompted; and a row is a worktree exactly when its id has a `/`
  in it, which is `#{m:*/*,#{@hover}}`. Only `@hover` is pushed, by the sidebar's
  `focus` binding. Do not move any of this into a `#()` — the one `#()` in the
  bar is the agent clock, and a second one would run a fork per status interval
  to answer what a format already knows. Keep the branches comma-free: a format
  splits `#{?:,}` on commas.

- **Depth is a convention: recessed panels, raised popups.** Sidebars and the
  status bar are `C_INSET`, darker than the terminal's own background, so the
  view reads as the thing being worked in; a popup is `C_RAISED`, lighter, so it
  reads as a card over the top. `set -p window-style` is what paints a sidebar
  pane (fzf paints only the rows it draws), and `POPUP_CARD` is the one place a
  popup's border and colours are defined — every popup passes it.

- **A sidebar's first row names it and lights up when it has the keyboard.**
  Nothing else on screen says which of the three panes the keys are going to, and
  a border cannot say it: with the issues pane hidden there is one divider and
  tmux draws it in `pane-active-border-style` whether the sidebar or the view is
  active. So each pane carries its own heading — fzf's `--header`, ANSI and all,
  set once at startup and re-read with `transform-header` on `ctrl-o`. On the
  left sidebar that key also does a `reload-sync`, because a port badge is only
  as true as the last draw and nothing announces a dev server going up, so the
  keyboard moving between panes is the moment worth looking again. Two things
  about how it is driven. It hangs off **`after-select-pane`**, not
  `pane-focus-in`: both were measured to fire, but the focus hooks need
  `focus-events on`, which starts forwarding focus escapes to fzf, nvim and
  claude, while every path in mn that changes the active pane (mouse clicks
  included) goes through `select-pane`. And the hook sends `ctrl-o` to *both*
  sidebars and tells them nothing else: `header` reads the active pane at draw
  time, so this stays the same one-way "look again" that `sidebar_reload` is.
  The heading survives a `reload-sync`, an Esc and a resize — measured, since a
  header that a redraw could blank would be worse than none.

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
- **A split run from outside the session starts in the caller's directory.**
  `split-window` with no `-c` does not inherit the target pane's cwd when the
  command comes from another process: measured, a session made with
  `-c /tmp/x` split from this repo put the new pane in the repo. Hence every
  `-c` in `layout_task`, and the rule in the README for a project's own layout.
- **tmux rescales panes proportionally** when a client attaches and when the
  terminal resizes, which overrides any build-time `resize-pane`. The sidebar
  needs hooks on **`client-attached` and `window-resized`**. `client-resized`
  does *not* hold it — measured drift to 62 columns on a resize to 260.
- **`run-shell` must be `-b`.** Without it the tmux server blocks while `mn`
  calls back into that same server, and deadlocks.
- **A `#()` in a status format is the only clock in the system.** tmux re-runs
  it every `status-interval`, which is what drives `mn poke`, and it is the
  reason nothing here needs a daemon or a claude hook to notice an agent going
  from busy to waiting. Three things measured before hanging anything off it: it
  fires on the clock even when its output is empty (so an invisible one still
  ticks); calling back into the *same* server from one does **not** deadlock it
  the way `run-shell` without `-b` does; and it does not run at all while no
  client is attached — which is the behaviour you want, since nobody is looking
  at the sidebar then. `status-interval` is set explicitly rather than left at
  tmux's 15, because the user's own `tmux.conf` is loaded by this server too.
  The tick is a diff against `$STATE/agents` and only then a `sidebar_reload`,
  for the same reason `pr_sync` does it that way: a redraw is several forks per
  row, so an unchanged screen has to cost nothing.
- **`send-keys -t ''` types into the active pane.** An unset option reads back as
  an empty string, and an empty target is not "no pane", it is the current one.
  `@rsb_pane` is unset until the first `M-i`, so the unguarded half of `headers`
  put its `C-o` into the view: into claude, where it toggles the transcript. Every
  send to a sidebar goes through `pane_shown` first.
- **`extended-keys on` alone is inert.** tmux ships no `extkeys` terminal
  feature for any terminal — the defaults are only `xterm*:clipboard:ccolour:
  cstyle:focus:title`, `screen*:title`, `rxvt*:ignorefkeys`. Hence the explicit
  `set -as terminal-features 'xterm-kitty:extkeys'`. Other terminals need their
  own entry or `C-S-*` silently does nothing.
- **Detached sessions are built at 80x24.** Percentage splits only take their
  intended proportions once a client attaches; don't chase the numbers before
  then.
- **`display-popup -E` blocks the process that ran it** until the popup's
  command exits, and it needs a client attached to draw on — from a detached
  server it fails silently and returns at once. Both are why the issue popup
  works from inside fzf's `execute-silent`: fzf sits frozen behind a popup that
  owns the keyboard, and redraws when it closes.

- **A closing popup kills the process group inside it.** So a `&` started from
  one dies with it, and `switch_to` is the last thing every popup does:
  `pr_sync` was being killed a second into its `gh` call, and the worktree you
  had just made from a PR sat there with no badge until the next switch. Hence
  the `( trap '' HUP; … ) &`, which is also why the sync needs no `setsid` or
  `nohup`, neither of which mn is allowed to depend on. Measured: with the
  trap the cache file appears after the popup is gone; without it, never.

- **`mn` cannot run without a tty** — it ends in `tmux attach`. Everything
  before that still executes, so running it from a script pre-builds both
  servers and only the attach fails.

## Bash and fzf gotchas

- **`printf` precision counts BYTES, not display columns.** `%-22.22s` on a
  string starting with `…` (3 bytes) fits two fewer characters, which knocks the
  port column out of alignment on exactly the rows that have a status mark. The
  sidebar's marks (`~`, `!`) are ASCII for this reason.

- **fzf's left columns are reclaimable; one column at the right is not.** The
  pointer and the marker each own a column of every row, the marker even in a
  list with no `--multi`, so `--pointer='' --marker=''` hands both back and a row
  starts in the pane's own first column. Measured in a live fzf: item text at
  column 2 with both, column 1 with one of them, column 0 with neither.
  `--gutter=''` is the one fzf refuses outright (`gutter display width should be
  1`), but with no pointer column the gutter is never drawn anyway. At the right
  it keeps a single column for the scrollbar: in a 30-column pane a 29-character
  row fits and a 30-character one comes back as 27 plus `··`. So both sidebars
  lay their rows out in `w - 2`, that column and one of air beside it, and
  `list_rows` splits its `cw` as line one `1` (the accent bar) `+ 2` (the state
  mark) `+ tw` (the headline) and line two `3` (the bar and two of indent) `+ pw`
  (the PR field) `+ 1 + 5` (port). So `tw = cw - 3` and `pw = cw - 9`, and every
  row is laid out that way — a project's name starts in the same column as a
  worktree's task, because the two columns before it are the mark its own agent
  gets. Nothing indents under a project, and the rule that separates one
  project's rows from the next's is `cw` wide.

- **A row is one fzf item of two lines, so the list is NUL-terminated.**
  `list_rows` prints `\0` after each item and the two sidebar callers pass
  `--read0`; fzf then treats it as one row, so the highlight covers
  both lines and `pos(n)` still counts items. It follows that nothing may count
  *lines* to find a row any more: `goto` and `switch_to` index `sessions`
  instead, which is the same order `list_rows` renders in. Measured in a live
  fzf, including that `--read0` holds across a `reload`.
  The rule between projects is a *third* line inside the item it sits above, for
  the same reason: fzf has no unselectable row, so a separator of its own would
  be an item you could put the cursor on and press Enter at. It is kept out of
  the cursor's tint by painting its own background — see the rule below.
  Measured in a live fzf: `pos(4)` still lands on the fourth session with two
  rules above it.

- **`reload` throws the left sidebar's cursor away; `reload-sync` keeps it.**
  fzf empties the list and refills it as the command streams, and a row costs
  several forks, so the redraw lands before the rows do and the cursor comes
  back at the top — or clamps to however many had arrived, if a `pos(n)` follows
  it. Measured against a producer that sleeps between items: `reload` left the
  cursor on item 1, `reload-sync` on item 5, where it started. Hence `ctrl-r`
  and `resize` both reload the left sidebar synchronously, which is also what
  lets `switch_to` redraw and move the cursor in one breath. The issues pane
  keeps a plain `reload`: walking the left sidebar re-renders it against a
  different project, so its cursor belongs back at the top anyway.

- **A row's width comes from `$FZF_COLUMNS`, not `@sbwidth`.** fzf exports it to
  the commands it reloads from, so a row lays itself out for the pane as it is
  now. `@sbwidth` only records what `mn width` last asked for, and a divider
  dragged with the mouse never touches it, so reading it left the port badge
  stranded mid-pane. It is 0 on fzf's `start` event, hence the fallback to the
  pane's own `#{pane_width}` for the first draw. The one caller fzf cannot tell
  is `picker`, which pipes `list_rows` *into* fzf rather than reloading from it,
  so it exports the variable itself from `tput cols` — inside a popup that is the
  card with its borders already taken off, 38 for a `-w 40`, measured. Without
  it the popup's rows lay themselves out for the sidebar instead, which left the
  rule short of the card's edge.

- **`set -euo pipefail` turns a missing file into a truncated list.** `sed …
  file | head -1` on an absent file exits 2, pipefail propagates it, `set -e`
  kills `mn list` mid-stream and the sidebar silently loses every row after it.
  `meta` guards with `[ -f ]` for exactly this.

- **`set -e` does not fire on a failed `A && B`.** All the `[ -f x ] && { …; }`
  guards rely on that; verified, not assumed.

- **A key sent to a pane is swallowed while its fzf is inside an `execute`.**
  So a popup opened from a binding cannot send itself a redraw on the way out:
  `issue_new`'s `display-popup -E` returns while fzf is still waiting on the
  child, and the `C-r` after it never arrived: measured, no `gh` call and no new
  row, and the pane came back with its heading overwritten. Chain the reload
  onto the binding instead (`n:execute-silent(…)+reload(…)`), so it is fzf's own
  next action. `issues_reload` and `sidebar_reload` still work because they are
  sent from somewhere else to a pane that is idle. A sidebar key that opens a
  popup is the case where both would fire, because `rm_worktree`, `hide_project`
  and `wt_build` each call `sidebar_reload` themselves for the menu's benefit —
  hence `MN_CALLER_REDRAWS`, which `row_popup` sets with `display-popup -e` and
  `sidebar_reload` returns on. Measured all three ways: the send alone left the
  removed worktree on screen, the two together left it there under a half-drawn
  duplicate, and the chained reload alone came back right.

- **A popup clobbers the fzf pane underneath it, and on a list that can come
  back shorter only `clear-screen` repairs it.** The popup is an overlay tmux
  paints over the pane; fzf's incremental redraw has no idea its screen was
  damaged, so it repaints the rows it thinks changed and leaves the tail of the
  old list showing. Measured after removing a worktree with `x`: `reload-sync`
  and plain `reload` both came back with a duplicated row and a stale scrollbar,
  and `+clear-screen+reload-sync(…)` came back exact. The issues pane wants none
  of it, measured the same way: its list is the same length after the popup as
  before, so the reload covers every row that was on screen.

- **The issues pane is cached, and the cache is the contract.** `issue_fetch`
  writes `~/.local/state/mullion/<project>/issues` and nothing else reads `gh`.
  Walking the left sidebar calls `issues_reload` on every row, so an uncached
  render would be an API call per keystroke. A failed fetch keeps the previous
  file rather than truncating it. A PR badge is the same arrangement one
  step further out: `pr_sync` is the only thing that runs `gh` for it, it writes
  `<project>/prs`, one line per branch, and it runs in the *background* — from
  `start` and from `switch_to`, TTL-guarded — so `list_rows` reads a file and no
  keypress waits on github. Its project list is `projects.conf` rather than
  `$STATE/*`, because a main checkout's own branch is in that file too and a
  project with no worktrees has no state dir to be found by; keying the file by
  branch stays unambiguous, since git refuses to check one branch out twice. The
  branch a project's row is looked up by comes from `head_branch`, which reads
  `.git/HEAD` — a file read like the port's, and a detached HEAD has no `ref:`
  line, so the row simply gets no badge. A checkout sitting on the repo's
  *default* branch is not asked about either: `default_branch` reads
  `.git/refs/remotes/origin/HEAD` (the same kind of file read, and `pack-refs`
  leaves a symref loose), and `pr_sync` drops the pair before the `gh` call
  rather than filtering at draw time, so it costs a request as well as a badge.
  Sofia's `master` is the head of a pull request somebody closed in 2016, and a
  row saying `#263 closed` says nothing about the checkout you are looking at. A
  repo with no `origin/HEAD` keeps the old behaviour rather than guessing.
  `gh pr list --head` is what makes that
  cache safe to overwrite: it exits 0 and prints nothing when a branch has no PR,
  and nonzero when the call failed, so an offline sync keeps the badges it had.
  The rule is about *drawing*, not about `gh`:
  the popup Enter opens fetches the issue body live, `issue_open` has always
  handed `--web` to `gh`, `issue_form` files one with a live `gh issue create`,
  and the PR picker's whole list is a live `gh pr list` with no cache and no TTL
  behind it. Each is one keypress by one person, and
  none of them redraws. Do not "fix" that by caching bodies or PR lists, and do
  not read `gh` from anything that draws a row.

- **fzf's defaults put two things on screen the theme has to take back.**
  `--gutter` defaults to `▌`, which draws a grey bar down every row the cursor
  is not on, and `hl`/`current-hl` default to a red that has nothing to do with
  the palette. Neither is wanted here: the accent bar is the active session's and
  the cursor is a tint, so the pointer and the marker are emptied and their
  columns go back to the rows, as the reclaimable-columns rule above sets out.
  `--color` can be passed more than once and later specs merge, so the shared
  look lives in `SB_LOOK` and each caller adds only its own `bg` and `gutter` —
  a sidebar's and a popup's differ.

- **A row's first line is the item's headline** — a worktree's task, a project's
  name. It carries `C_TEXT`, the project name bold and the task not, or the
  status colour while setup is running or broken; the line under it stays
  furniture. So brightness is not free to mean "active" any more — that is the
  bar's job.

- **An item's own background beats `--highlight-line`'s, which is how the rule
  between projects stays out of the cursor's tint.** The rule lives inside the
  item below it, so fzf paints `current-bg` across it along with the row — unless
  the line says what its background is, in which case the item's own colour wins
  for the cells it covers. Measured in a live fzf with a magenta test rule: the
  glyphs kept magenta on the current line and only the padding after them took
  the tint. Two halves of it to keep: a `49` (default background) reset does
  *not* work, because fzf reads that as "no background" and paints over it, so it
  has to be a real colour — `$E_INSET_BG` for a sidebar, `$E_RAISED_BG` for a
  popup, which is why `list_rows` takes the surface it is drawn on as an
  argument. And the colour has to reach `w - 1`, because that is how far fzf pads
  the line it is highlighting; hence the space after the rule's glyphs, which
  also leaves them ending at `cw` with the rows above.

- **A row's colour wraps its padded field, never sits inside it.** Same reason
  as the byte-precision rule above: `%-*.*s` counts an escape sequence's bytes
  towards the width, so colouring inside the field costs the port column a
  dozen columns per row. Verified in a live fzf pane with `capture-pane -e`.

- **`tr` maps bytes, so `tr ' ' '─'` prints mojibake** — it replaces each space
  with the *first byte* of a three-byte character. Build a rule with parameter
  expansion instead (`rule=${rule// /─}`), which is also fork-free.

- **`clear-query` before `hide-input`, never after.** `esc:hide-input+clear-query`
  parses, runs, and leaves the query in force with the input line gone, so the
  list stays cut down to whatever you last typed and nothing on screen says why.
  Measured both orders in a live fzf; only `esc:clear-query+hide-input` puts the
  rows back.

- **`/dev/tcp` is a bash builtin**, so the free-port check needs no `nc` or
  python. It is a *connect* test, so it cannot see a port that is allocated but
  idle — `alloc_port` also skips every port already in a worktree's `meta`.

- **Whether a port is *live* comes from `/proc/net/tcp`, not from a connect.**
  `listening` scans that and `tcp6` for state `0A` once per render, and the badge
  is a `case` glob against the result, so one read answers for every row where a
  connect would open and drop a socket on somebody's dev server for every row
  drawn. Two things it cost to get right: the remote port has to be pinned to
  `0000`, or an outbound connection *to* one of those ports reads as a server on
  it; and the scan is `sed`'s, because bash's own `read` goes a byte at a time on
  /proc — 57ms for 191 lines against 4 for the sed, both measured here. It is one
  of the two things a row reads that are not under `$STATE` — the other is
  `$CC_SESSIONS` for the agent mark — and both are file reads rather than
  network calls.

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

Point the copy at fakes as well, or the probe builds the user's real work: give
it its own `projects.conf` (it is read from `dirname $SELF`) and its own
`XDG_STATE_HOME` holding hand-written `meta` files, and give the fake projects
either no `project.sh` at all or one whose `layout` phases send something inert.
Replace the `claude` and `mn agent` sends in `layout_task` too, since that is
the layout a worktree with no `layout-task` phase still gets. Otherwise starting
the probe launches an agent per worktree and a dev server with it. A state dir is
named by `dirslug`, so `proj/task-1` lives in `<state>/mullion/proj/task-1`.
`WT_ROOT` needs the same treatment and has no env var, so sed it too, or the
probe writes checkouts into the user's real `~/.mullion/worktrees`.
A hand-written `meta` needs its `SESSION=` and a `WT=` that exists: `reconcile`
reclaims a worktree whose `WT` is gone, and `ensure_work` runs
`new-session -s ''` for a missing `SESSION`, which fails the whole start with
`duplicate session:` and nothing else to go on. A project only needs a
`.git/HEAD` holding `ref: refs/heads/<branch>` for its row's PR badge; a bare
sha there is a detached HEAD and gets none.

`CC_SESSIONS` is the third one to sed, at a fake dir of hand-written
`<pid>.json` files — `{"tmux":"proj-a/task-1:@1.%1","status":"waiting"}` is
enough of one, and `{"tmux":"proj-a:@1.%1",…}` marks the project's own row, since
a main checkout's session is named after it. The pid in the filename has to be a
live process or the `/proc` guard drops it, which a `sleep 600 &` covers.
Otherwise the probe's rows read the agent state of the user's real sessions, and
all four marks (`!`, `?`, `~`, `*`, blank) become impossible to test
deterministically. Ten seconds of
`status-interval` is also long enough that a test asserting on a redraw should
flip a file and then wait a full interval rather than poll tightly.

Anything that talks to github needs a repo `gh` can resolve without being the
user's. A throwaway clone is enough, and it costs no objects:

```bash
git clone --shared --no-checkout ~/projects/sofia /tmp/t/probe
git -C /tmp/t/probe remote set-url origin git@github.com:owner/sofia.git
```

`gh` then answers for the real project while `git worktree add`, `branch -d` and
the state dir all land in the throwaway. Give it a `.mullion/project.sh` that
mentions `MN_PORT` and does nothing, and the port path is exercised too.

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
