# You are inside mullion

mullion (`mn`) is the tmux workspace this session runs in. A project has one
main checkout and, under it, a worktree per task: its own branch, a checkout
under `~/.mullion/worktrees/<project>/<branch>`, a tmux session named
`<project>/<branch>`, an agent started holding the task, and the project's dev
server beside it. The person has a sidebar listing every row with a mark for
what its agent is doing, so they may be looking at another row while you work.

## Handing work off

When you find a separate piece of work that does not belong on this branch, do
not do it here. Hand it to a new worktree, whose agent starts on it at once:

    mn new <project> "<task>" <branch> "<title>"

- `<project>` is the project named above.
- `<task>` is the new agent's opening prompt: one line saying what and why.
  Start it with `#<number> ` when it is a GitHub issue or pull request; the
  agent is then handed the title, URL and body. If the work deserves an issue,
  file one with `gh issue create` first and hand the number over.
- `<branch>` is lowercase kebab-case, two to five words, no issue number.
- `<title>` is what the sidebar row says: two to five words, sentence case, no
  issue number.

The new branch starts at the main checkout's current commit, not at yours, so
it sees none of this branch's changes; say so in the task if the work depends
on them. The command prints one line when the row is up, and the person's view
stays where it is. Do not switch to, cd into or edit the new worktree; it
belongs to its agent now.

## What is not yours

- Work only in this checkout. Other worktrees belong to other agents.
- Removing a worktree is the person's key, once the work has landed.
- Never run `mn reload` or `mn stop`; they take down the whole screen.
