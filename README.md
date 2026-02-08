# claudemacs

tmux-native session manager for Claude Code. Manage multiple Claude tasks across projects and machines from a single keybinding.

## Features

- **Project/task organization** — group Claude sessions by project, each task in its own tmux window
- **Live status detection** — see which tasks are running, waiting for input, idle, or archived
- **Fuzzy picker** — switch between tasks instantly with fzf, with pane preview
- **Remote sessions** — create and manage tasks on remote machines over Tailscale, with persistent tmux sessions that survive disconnects
- **Archive** — hide finished tasks without killing them

## Requirements

- [tmux](https://github.com/tmux/tmux) (3.2+)
- [fzf](https://github.com/junegunn/fzf) (0.38+)
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- [Tailscale](https://tailscale.com) (only for remote sessions)
- Python 3 (only for remote peer discovery)

## Install

```bash
# Clone and make executable
git clone <repo-url> ~/claudemacs
chmod +x ~/claudemacs/claudemacs

# Or just copy the single script anywhere on your PATH
cp claudemacs /usr/local/bin/
```

## tmux setup

Add to `~/.tmux.conf`:

```tmux
bind C-t display-popup -E -d '#{pane_current_path}' -w 80% -h 80% "/path/to/claudemacs"
```

Reload:

```bash
tmux source-file ~/.tmux.conf
```

Now `Ctrl-b Ctrl-t` opens the claudemacs picker from anywhere inside tmux.

## Usage

### Picker keybindings

| Key       | Action                        |
|-----------|-------------------------------|
| `enter`   | Switch to selected task       |
| `alt-n`   | Create new task               |
| `alt-o`   | Create new project            |
| `alt-a`   | Archive/unarchive task        |
| `ctrl-d`  | Kill task                     |
| `esc`     | Close picker                  |

### CLI

```bash
claudemacs              # Open interactive picker
claudemacs new          # Create a new task
claudemacs new-project  # Create a new project
claudemacs projects     # List projects
claudemacs setup        # Print tmux.conf setup instructions
```

### Status indicators

| Icon | Status    | Meaning                                    |
|------|-----------|--------------------------------------------|
| `◎`  | waiting   | Claude is waiting for user input           |
| `●`  | running   | Claude is actively running a tool          |
| `○`  | idle      | Claude session at prompt, not doing anything |
| `⚙`  | process   | Non-Claude process running (node, etc.)    |
| `▪`  | archived  | Hidden from default view                   |

Tasks are sorted by status: waiting first, then running/idle, process, archived last.

## Remote sessions

claudemacs can create and manage Claude sessions on remote machines over Tailscale. Remote tmux sessions persist across network disconnects — your Claude tasks keep running even when your laptop sleeps.

### How it works

1. **Remote tmux for persistence** — tasks run in tmux on the remote machine, so they survive disconnects
2. **Local proxy for window management** — a local tmux session SSHes into the remote tmux, giving you seamless switching from the picker
3. **`select-window` for task switching** — picking different remote tasks sends `tmux select-window` to the remote rather than creating new connections

### Prerequisites

Both machines need:
- tmux
- Claude Code CLI
- Tailscale (connected to the same tailnet)

The local machine also needs fzf and Python 3.

### SSH configuration

SSH multiplexing is strongly recommended. It reuses TCP connections so that remote task fetching and window switching are fast (~0.2s instead of ~2s per command).

Add to `~/.ssh/config` on your **local** machine:

```ssh-config
Host *
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h-%p
    ControlPersist 10m
```

Create the sockets directory:

```bash
mkdir -p ~/.ssh/sockets
```

#### SSH key authentication

Password-based SSH won't work — claudemacs uses `BatchMode=yes` for all background SSH commands. Set up key-based auth:

```bash
# Generate a key if you don't have one
ssh-keygen -t ed25519

# Copy it to the remote machine (use the Tailscale IP or hostname)
ssh-copy-id user@100.x.x.x
```

Verify it works without a password prompt:

```bash
ssh -o BatchMode=yes 100.x.x.x echo ok
```

### Remote machine setup

Ensure tmux and Claude Code are in a standard PATH location on the remote. claudemacs automatically prepends common paths (`/usr/local/bin`, `/opt/homebrew/bin`, `/home/linuxbrew/.linuxbrew/bin`) for non-interactive SSH, but if your tools are elsewhere you may need to adjust `_RENV` in the script.

Verify the remote is reachable and tmux works:

```bash
ssh 100.x.x.x tmux -V
```

### Creating remote projects and tasks

1. Open the picker (`Ctrl-b Ctrl-t`)
2. Press `alt-o` to create a new project
3. Select the remote host from the host picker
4. Enter a project name and the **path on the remote machine**
5. Enter a name for the first task — Claude starts automatically

### Switching to remote tasks

Remote tasks appear in the picker alongside local ones, with the hostname shown in brackets:

```
catch       wearables      ● running
catch       movie          ○ idle
womoji      permissions    ◎ waiting  [MacBook Pro]
womoji      bugs           ○ idle     [MacBook Pro]
```

Select any remote task and press `enter`. claudemacs will:
- Send `tmux select-window` to the remote to switch to that task
- Create a local proxy session (if needed) that SSH-attaches to the remote tmux
- Switch you to the proxy — you're now in the remote Claude session

If the SSH connection drops (laptop sleep, network change), just select the task again from the picker. claudemacs detects the dead connection and reconnects automatically.

### Architecture

```
┌─ Local machine ──────────────────────────────┐
│                                               │
│  claudemacs picker (fzf popup)                │
│    ├─ local tasks: tmux capture-pane          │
│    └─ remote tasks: ssh + capture-pane        │
│                                               │
│  cm/myproject          (local tmux session)   │
│    ├─ window 0: task-a  (claude)              │
│    └─ window 1: task-b  (claude)              │
│                                               │
│  cm/remoteproject      (proxy session)        │
│    └─ window 0: remote  (ssh -t ... attach)   │
│                                               │
├───────────── Tailscale ──────────────────────►│
│                                               │
│  ┌─ Remote machine ────────────────────────┐  │
│  │                                         │  │
│  │  cm/remoteproject   (remote tmux)       │  │
│  │    ├─ window 0: task-x  (claude)        │  │
│  │    ├─ window 1: task-y  (claude)        │  │
│  │    └─ window 2: task-z  (claude)        │  │
│  │                                         │  │
│  └─────────────────────────────────────────┘  │
└───────────────────────────────────────────────┘
```

### Troubleshooting

**Remote tasks don't appear in picker**

- Check Tailscale: `tailscale status` — is the peer online?
- Check SSH: `ssh -o BatchMode=yes <ip> echo ok` — does it connect without a password?
- Check remote tmux: `ssh <ip> tmux list-sessions` — are there `cm/` sessions?
- If tmux isn't found: `ssh <ip> 'export PATH="$PATH:/usr/local/bin:/opt/homebrew/bin"; tmux -V'`

**"Shared connection closed" after switching**

The SSH multiplexed connection timed out. Just select the task again — claudemacs auto-reconnects. Increase `ControlPersist` in `~/.ssh/config` if this happens frequently.

**Remote status shows as "process" instead of running/idle**

The status detection runs `capture-pane` on the remote via SSH. If Claude's UI characters (`⏺`, `❯`) aren't detected, it falls back to "process". Ensure the remote machine has proper UTF-8 locale support.
