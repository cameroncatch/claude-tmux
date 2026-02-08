# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is claudemacs?

A tmux-native session manager for Claude Code. It's a single bash script (~540 lines) that provides an fzf-based interactive picker to manage multiple Claude Code sessions organized into projects and tasks. Supports both local and remote sessions via SSH/Tailscale.

## Running and Testing

```bash
# Run the interactive picker (requires tmux + fzf)
./claudemacs

# Subcommands
./claudemacs new          # Create a new task
./claudemacs new-project  # Create a new project
./claudemacs projects     # List projects
./claudemacs setup        # Print tmux.conf binding instructions
```

There is no build step, test suite, linter, or formatter. The script is tested manually.

## Architecture

The entire tool is a single executable bash script (`claudemacs`) with no external package dependencies beyond standard Unix tools, `tmux`, `fzf`, and optionally `tailscale`/`python3` for remote features.

### Session Naming Convention

Sessions use the format `cm/<project>` with tmux windows as individual tasks. A target reference looks like `cm/myproject:2` (session:window_index).

### Remote Session Format

Remote targets use `remote:<ip>:<session>:<window>` and are accessed via SSH.

### Script Organization

The script follows a functional pattern with clear sections:

- **Data management** (`_toggle_archive`, `_unarchive`, `_is_archived`) — archive state stored in `~/.local/share/claudemacs/archived`
- **Query functions** (`_sessions`, `_projects`, `_tasks`, `_remote_peers`, `_all_projects`) — read tmux/tailscale state
- **Preview/display** (`_preview`, `_merged_tasks`) — render pane output for fzf preview
- **Actions** (`_switch`, `_kill_task`, `_create_project`, `_create_task` + remote variants) — mutate tmux state
- **Interactive flows** (`_new_project_flow`, `_new_task_flow`) — multi-step user interactions
- **Picker** (`_picker`) — main fzf interface with keybindings
- **Entry point** (`main`) — case dispatch on `$1`

### Status Detection

Task status is determined by inspecting the last lines of tmux pane output:
- `●` running — Claude is executing (detects "esc to interrupt" / "ctrl+c to interrupt")
- `◎` waiting — Claude needs user input (detects permission prompts, Y/n, etc.)
- `○` idle — shell prompt or Claude idle
- `⚙` process — non-Claude process running
- `▪` archived — manually archived by user

### Key Design Decisions

- The script re-invokes itself (`$self`) from within fzf bindings for preview, reload, switch, and kill actions
- Remote task fetching runs in the background (`_fetch_remote &`) and results are merged on fzf focus events
- Internal subcommands prefixed with `_` (e.g., `_preview`, `_switch`) are exposed via `main()` for fzf callback use but are not part of the public interface
