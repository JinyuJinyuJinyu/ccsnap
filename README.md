# ccsnap

Snapshot and restore [Claude Code](https://claude.com/claude-code) sessions living in tmux.

If you keep a lot of Claude Code sessions open across tmux panes, a reboot loses all of them and you reopen each one by hand. `ccsnap` records which conversation is in which pane. After the restart, one command rebuilds the tmux sessions and resumes every conversation where it was.

```
$ ccsnap list
    #  saved                claude  tmux sessions (panes)
  *  1  2026-11-16 09:47:49      11  api(5) web(3) notes(2) review(2)
     2  2026-11-16 09:45:49      10  api(4) web(3) notes(2) review(2)

$ ccsnap restore
```

## What a snapshot holds

- Every tmux session, window and pane: names, window numbers, the exact pane layout, working directories, and which window and pane were active.
- For each pane running Claude Code: the conversation's session id, its name, and the flags it was started with, such as `--dangerously-skip-permissions` or `--model`.
- Panes showing a background job, whether through `claude attach` or because the conversation was sent to the background.

A snapshot is a small JSON file of pointers. The conversations themselves stay where Claude Code keeps them, in `~/.claude`.

Not recorded: Claude sessions running outside tmux, and settings changed inside a session after it started (for example with `/model`).

## Requirements

- Python 3.9 or newer (standard library only)
- tmux
- The Claude Code CLI (`claude`)
- Linux or macOS

### How far it has been tested

| | Status |
|---|---|
| Linux | In daily use on Ubuntu 24.04 with tmux 3.4 and Claude Code 2.1.325. Restores have been run after real reboots with up to 18 conversations, including background jobs. |
| macOS | Not yet run on a Mac in this version. The code that reads processes on macOS is the same code Linux runs when `CCSNAP_PS=1` is set, and it produces identical snapshots there. The launchd autosave is carried over from the first, macOS-only version. Use `ccsnap restore -n` to preview before trusting it. |

## Install

It is a single file. Put it anywhere on your `PATH`:

```sh
mkdir -p ~/.local/bin
curl -fsSL https://raw.githubusercontent.com/JinyuJinyuJinyu/ccsnap/main/ccsnap -o ~/.local/bin/ccsnap
chmod +x ~/.local/bin/ccsnap
ccsnap auto on    # optional: snapshot every 2 minutes
```

## Usage

| Command | What it does |
|---|---|
| `ccsnap` | Save a snapshot now (same as `ccsnap save`) |
| `ccsnap list` | List saved snapshots, newest first; `*` marks the one `restore` would pick |
| `ccsnap show [N]` | Show what a snapshot holds and the command each pane would get |
| `ccsnap restore [N]` | Rebuild the missing tmux sessions and resume Claude in each pane |
| `ccsnap restore -n` | Show what a restore would do, without doing it |
| `ccsnap restore -a` | Attach to the first restored session when done |
| `ccsnap restore --only NAME…` | Restore just these tmux sessions |
| `ccsnap restore --replace` | Also kill and rebuild tmux sessions that already exist |
| `ccsnap auto on\|off\|status` | Turn automatic snapshots on or off |

`N` is the number shown by `ccsnap list`. A snapshot's name, any unique prefix of it, or a path to the file also works.

## How restore behaves

- **It only adds.** A tmux session that already exists is left alone, so running restore twice is harmless. `--replace` is the exception: it kills the named sessions and rebuilds them, for when a session exists but is broken. It asks first unless you pass `-y`, and it never kills the session it is running in.
- **It picks the snapshot from before the reboot.** Autosave keeps running while the machine comes back up, and what it records then is the empty state, not the one you want. Until a restore has succeeded since the last boot, the default is the newest snapshot taken before that boot. After that, it is the newest snapshot holding a tmux session that is not running.
- **It waits for folders that are not there yet.** If a pane's working directory is missing, for example because a data disk is still being checked and mounted, restore waits for it (`--wait`, default 300 seconds). A tmux session whose folder never appears is skipped, and restore exits with status 1. It never starts Claude in a different folder.
- **It does not start a conversation twice.** If a conversation is already running somewhere else, its command is typed into the pane but not run.
- **It types the command into the pane's shell**: `claude --resume <id>` plus the original flags. Any alias or wrapper you have for `claude` therefore still applies. Flags that choose a session (`--resume`, `--continue`, `--name`) and a prompt given on the command line are left out, so nothing is sent again. Commands are sent 0.4 seconds apart (`--stagger`).
- **A pane that was showing a background job** gets `claude attach <job>`, which restarts the job if needed.
- **A session that never got a first message** has nothing to resume. It starts fresh in the same folder, with the same flags and name.
- **A pane that was not running Claude** comes back as a shell in its directory. Other programs are not restarted.
- **Very long conversations** stop at Claude Code's own "Resume this conversation?" question, which shows what resuming will cost. Answering it is left to you.

When it finishes it prints the `tmux attach` line. It does not reopen terminal windows or tabs.

## Automatic snapshots

`ccsnap auto on` sets up a job that runs `ccsnap save --quiet` every 2 minutes. Use `--every MIN` for a different interval.

| System | Mechanism |
|---|---|
| Linux | A systemd user timer, `ccsnap.timer`, in `~/.config/systemd/user` |
| macOS | A launchd agent, `com.ccsnap.autosave`, which also runs at login. Errors go to `~/.ccsnap/auto.log` |
| Neither | `ccsnap auto on` prints a cron line to add instead |

A new file is written only when something changed. The newest 200 snapshots are kept.

Autosave runs the copy of `ccsnap` you ran `auto on` from. Keep that file on a disk that is available at boot.

## Where things are kept

| System | Folder |
|---|---|
| Linux | `~/.local/state/ccsnap` (or under `$XDG_STATE_HOME`) |
| macOS | `~/.ccsnap` |

Set `CCSNAP_HOME` to use a different folder on either system.

## How it works

Claude Code keeps a small record for each running session in `~/.claude/sessions/<pid>.json`, holding the session id, working directory and name. For each tmux pane, `ccsnap` walks the processes below the pane's shell until it finds one with such a record, then reads that process's command line for its flags. A pane running `claude attach` has no record of its own and is recognised by its command line. Process information comes from `/proc` on Linux and from `ps` on macOS.

Restore rebuilds each tmux session with `new-session`, `new-window` and `split-window`, re-applies the saved layout string, then sends the command to each pane.

## Limitations

- **It depends on undocumented Claude Code files.** The layout of `~/.claude/sessions`, `~/.claude/jobs` and the transcripts is not a public interface, so a future Claude Code release may break `ccsnap`.
- **Interrupted work is not continued.** A conversation that was in the middle of a turn when the machine went down comes back at its last saved message.
- **On macOS, flags whose value contains a space are not carried over intact**, because `ps` does not show where one argument ends. This does not affect Linux.
- **One tmux server at a time**: the default one, or the one named with `-L`.

## Upgrading from the first version

The first version (0.1) was written for macOS only. This one replaces it:

- Snapshots written by 0.1 are in a different format. They are left where they are and ignored, so run `ccsnap save` once after upgrading.
- Run `ccsnap auto on` again to rewrite the launchd agent. The old agent keeps working in the meantime.
- Restore no longer asks for confirmation, and `--loose` (reopening sessions that ran outside tmux) is gone.
- The model and permission mode are taken from the flags the session was started with, not from the transcript.

## Environment variables

| Variable | Purpose |
|---|---|
| `CCSNAP_HOME` | Where ccsnap keeps its data |
| `CLAUDE_CONFIG_DIR` | Claude Code's directory (default `~/.claude`) |
| `CCSNAP_TMUX_SOCKET` | Use `tmux -L <name>` instead of the default server (same as `ccsnap -L <name>`) |
| `CCSNAP_PS` | Set to `1` to read processes through `ps` even where `/proc` exists |

To try it without touching your real sessions, point it at a scratch tmux server and a scratch data directory:

```sh
export CCSNAP_TMUX_SOCKET=scratch CCSNAP_HOME=/tmp/ccsnap-test
tmux -L scratch new-session -d -s demo
ccsnap save && ccsnap show
tmux -L scratch kill-server
ccsnap restore && tmux -L scratch ls
```

## Uninstall

```sh
ccsnap auto off
rm ~/.local/bin/ccsnap
rm -rf ~/.local/state/ccsnap ~/.ccsnap
```

## License

MIT
