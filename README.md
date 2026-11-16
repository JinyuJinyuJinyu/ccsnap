# ccsnap

Snapshot and restore [Claude Code](https://claude.com/claude-code) sessions living in tmux.

If you keep a lot of Claude Code sessions open across tmux panes, a reboot loses all of them and you reopen each one by hand. `ccsnap` records which conversation is in which pane. After the restart, one command rebuilds the tmux sessions and resumes every conversation where it was.

```
$ ccsnap list
ID                 SAVED             LAST SEEN   TMUX  PANES  CLAUDE  LOOSE
20261116-135533    Mon 16 Nov 13:55  just now       3      6       5      0

$ ccsnap restore
```

## What a snapshot holds

- Every tmux session, window and pane: names, window numbers, the exact pane layout, working directories, and which window and pane were active.
- For each pane running Claude Code: the conversation's session id, its name, its model, and its permission mode (including `--dangerously-skip-permissions`).
- Panes that are viewing a background job through `claude attach`.
- Claude sessions running outside tmux. These are recorded, and restored only on request (see `--loose`).

A snapshot is a small JSON file of pointers. The conversations themselves stay where Claude Code keeps them, in `~/.claude`.

## Requirements

- Python 3.9 or newer (standard library only)
- tmux
- The Claude Code CLI (`claude`)

Built and tested on macOS with tmux 3.7b and Claude Code 2.1.325. Saving and restoring have not been tested on Linux. `ccsnap auto` uses launchd, so on other systems it prints a cron line to add instead.

## Install

It is a single file. Put it anywhere on your `PATH`:

```sh
mkdir -p ~/.local/bin
curl -fsSL https://raw.githubusercontent.com/JinyuJinyuJinyu/ccsnap/main/ccsnap -o ~/.local/bin/ccsnap
chmod +x ~/.local/bin/ccsnap
ccsnap auto on    # optional: snapshot every 5 minutes
```

## Usage

| Command | What it does |
|---|---|
| `ccsnap` | Save a snapshot now (same as `ccsnap save`) |
| `ccsnap list` | List saved snapshots |
| `ccsnap show [ID]` | Show what a snapshot holds (default: the latest) |
| `ccsnap restore [ID]` | Rebuild the tmux sessions and resume Claude in each pane |
| `ccsnap restore -n` | Show what a restore would do, and the commands it would type, without doing it |
| `ccsnap restore --only NAME` | Restore just one tmux session (repeatable) |
| `ccsnap restore --loose` | Also reopen the sessions that ran outside tmux, in a tmux session named `loose` |
| `ccsnap restore -y` | Do not ask for confirmation |
| `ccsnap auto on\|off\|status` | Turn automatic snapshots on or off |

A snapshot id can be shortened to any unique prefix.

## How restore behaves

- **It only adds.** A tmux session that already exists is left alone, and a conversation that is already running somewhere is not started a second time. Running restore twice is harmless.
- **It picks a sensible snapshot.** Without an id, it uses the most recent snapshot that holds something not currently running. A snapshot taken just after the reboot, with almost nothing in it, does not get in the way of the one you want.
- **It shows the plan first** and asks before doing anything.
- **It starts conversations one at a time**, waiting for each to come up (up to `--wait` seconds, default 45) before starting the next, so a dozen large transcripts do not all load at once.
- **It types the command into the pane's shell**: `claude --resume <id>` plus the model and permission flags. Any alias or wrapper you have for `claude` therefore still applies. A pane that was viewing a background job gets `claude attach <job>`.
- **A pane that was not running Claude** comes back as a shell in its directory. Other programs are not restarted.
- **If a working directory no longer exists**, the pane opens in your home directory and the output says so.
- **When it finishes** it prints the `tmux attach -t <name>` lines. It does not reopen terminal windows or tabs.

## Automatic snapshots

`ccsnap auto on` installs a launchd agent (`com.ccsnap.autosave`) that runs `ccsnap save --auto` every 5 minutes and at login. Use `--every MIN` for a different interval.

A new file is written only when something changed; otherwise the latest snapshot's "last seen" time is updated. Old snapshots are pruned automatically. These are kept:

- the newest 30,
- the last snapshot of each of the 14 most recent days that have one,
- the last snapshot of each of the 8 most recent tmux server lifetimes, which is the record of what a restart took away.

Snapshots live in `~/.ccsnap/snapshots`. Errors from the background job go to `~/.ccsnap/auto.log`.

## How it works

Claude Code keeps a small record for each running session in `~/.claude/sessions/<pid>.json`, holding the session id, working directory and name. `ccsnap` matches those records to tmux panes by terminal device, and reads the tail of each transcript for the model and permission mode last in use. Only those metadata fields are read. If the session records cannot be used, it falls back to `claude agents --json`.

Restore rebuilds each tmux session with `new-session`, `new-window` and `split-window`, re-applies the saved layout string, then sends the resume command to each pane.

## Limitations

- **It depends on undocumented Claude Code files.** The layout of `~/.claude/sessions`, `~/.claude/jobs` and the transcripts is not a public interface, so a future Claude Code release may break `ccsnap`.
- **Interrupted work is not continued.** A conversation that was in the middle of a turn when the machine went down comes back at its last saved message.
- **Testing so far** has been save, kill the tmux server, restore. Treat it as a young tool.

## Environment variables

| Variable | Purpose |
|---|---|
| `CCSNAP_HOME` | Where ccsnap keeps its data (default `~/.ccsnap`) |
| `CLAUDE_CONFIG_DIR` | Claude Code's directory (default `~/.claude`) |
| `CCSNAP_TMUX_SOCKET` | Use `tmux -L <name>` instead of the default server |

To try it without touching your real sessions, point it at a scratch tmux server and a scratch data directory:

```sh
export CCSNAP_TMUX_SOCKET=scratch CCSNAP_HOME=/tmp/ccsnap-test
tmux -L scratch new-session -d -s demo
ccsnap save && ccsnap show
```

## Uninstall

```sh
ccsnap auto off
rm ~/.local/bin/ccsnap
rm -rf ~/.ccsnap
```

## License

MIT
