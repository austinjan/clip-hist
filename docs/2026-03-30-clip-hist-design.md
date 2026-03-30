# clip-hist Design Spec

## Problem

Terminal users frequently need to copy a command and its output — typically to paste into AI tools (Claude, ChatGPT) for debugging help. Current solutions re-execute the last command to capture output, which is dangerous for destructive commands (`rm`, `git push`, etc.).

## Solution

`clip-hist` is a single Rust binary that provides:

1. **Automatic capture** — shell hooks log every command + output to a history file without re-execution
2. **Interactive picker** — Ctrl+Y launches a TUI to browse captured commands, defaulting to the most recent
3. **Clipboard copy** — selected command + output is copied as a markdown code block, ready for AI chat

## Architecture

```
┌─────────────────────────────┐       ┌─────────────────────────┐
│  Shell Hooks (~10 lines)    │       │  clip-hist binary       │
│                             │       │                         │
│  zsh: preexec / precmd      │──────▶│  Subcommands:           │
│  PowerShell: PSReadLine     │ write │  - log   (append entry) │
│                             │       │  - pick  (TUI picker)   │
│  On each command:           │       │  - clear (reset history)│
│  1. preexec: save command   │       │                         │
│  2. precmd: save output     │       │  History file (JSONL)   │
│  3. call `clip-hist log`    │       │  TUI (ratatui)          │
│                             │       │  Clipboard (arboard)    │
└─────────────────────────────┘       └─────────────────────────┘
```

## History Storage

### File Location

- Linux/macOS: `~/.local/share/clip-hist/history.jsonl`
- Windows: `%APPDATA%\clip-hist\history.jsonl`

### Format

One JSON object per line (JSONL):

```json
{"cmd": "git status", "output": "On branch main\nnothing to commit", "ts": 1711785600, "cwd": "/Users/me/project"}
```

### Fields

| Field    | Type   | Description                          |
|----------|--------|--------------------------------------|
| `cmd`    | string | The command as typed                  |
| `output` | string | Combined stdout + stderr             |
| `ts`     | u64    | Unix timestamp when command executed  |
| `cwd`    | string | Working directory at execution time   |

### Limits

- Rolling 50 entries (oldest pruned when new entry appended)
- Max output size per entry: 64 KB (truncated with `[...truncated]` marker)

## Shell Integration

### zsh

```zsh
# >>> clip-hist >>>
__clip_hist_cmd=""
__clip_hist_output_file=$(mktemp)

clip-hist-preexec() {
  __clip_hist_cmd="$1"
}

clip-hist-precmd() {
  local exit_code=$?
  if [[ -n "$__clip_hist_cmd" ]]; then
    clip-hist log --cmd "$__clip_hist_cmd" --exit-code $exit_code
    __clip_hist_cmd=""
  fi
}

autoload -Uz add-zsh-hook
add-zsh-hook preexec clip-hist-preexec
add-zsh-hook precmd clip-hist-precmd

clip-hist-widget() {
  clip-hist pick
  zle reset-prompt
}
zle -N clip-hist-widget
bindkey '^Y' clip-hist-widget
# <<< clip-hist <<<
```

**Capturing output:** zsh `preexec`/`precmd` hooks cannot directly capture command output. Two approaches:

- **Approach A: `script` wrapper** — wrap command execution with `script` to tee output to a file. Intrusive, may break interactive programs.
- **Approach B: `PROMPT_COMMAND` + temp file** — redirect output via shell function. Also intrusive.
- **Approach C: Terminal integration** — use terminal emulator APIs (iTerm2, Ghostty, Windows Terminal) to read back scrollback. Requires terminal-specific code.
- **Recommended: Approach A with opt-in** — use `script -q` to capture output transparently. Users can disable for specific commands via an ignore list. This is the simplest approach that actually captures output without re-execution.

### PowerShell

```powershell
# >>> clip-hist >>>
$__clipHistLastCmd = ""

Set-PSReadLineKeyHandler -Chord 'Ctrl+y' -ScriptBlock {
    clip-hist pick
    [Microsoft.PowerShell.PSConsoleReadLine]::InvokePrompt()
}

# Use PSReadLine AddToHistory handler to capture commands
Set-PSReadLineOption -AddToHistoryHandler {
    param($command)
    $__clipHistLastCmd = $command
    return $true
}

# Capture output via Out-Default proxy
# (Implementation details TBD — PowerShell output capture is complex)
# >>> clip-hist <<<
```

**PowerShell output capture:** More complex than zsh. Options:

- **Approach A: `Start-Transcript`** — captures everything but output is messy with formatting artifacts
- **Approach B: Proxy `Out-Default`** — intercept pipeline output. Clean but complex to implement correctly.
- **Recommended: Start-Transcript with post-processing** — simplest reliable approach, clean up formatting in `clip-hist log`

## TUI Picker (`clip-hist pick`)

### Layout

```
┌─ clip-hist ─────────────────────────────────────────┐
│  Commands (↑↓ to navigate, Enter to copy, Esc quit) │
│ ─────────────────────────────────────────────────── │
│   git log --oneline -5                   10:23:01   │
│   cargo test                             10:24:15   │
│ ▶ git status                             10:25:30   │  ← focused (last command)
│ ─────────────────────────────────────────────────── │
│  Output preview:                                     │
│  ┌─────────────────────────────────────────────────┐ │
│  │ On branch main                                  │ │
│  │ Changes not staged for commit:                  │ │
│  │   modified:   src/main.rs                       │ │
│  │                                                 │ │
│  └─────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

### Keybindings

| Key       | Action                                  |
|-----------|-----------------------------------------|
| `↑` / `k` | Move to previous command               |
| `↓` / `j` | Move to next command                   |
| `Enter`   | Copy selected command + output to clipboard |
| `Esc` / `q` | Quit without copying                 |

### Behavior

- Opens with the **last command focused** — user can immediately press Enter for the common case
- Command list shows command text + timestamp
- Bottom pane shows output preview of the focused command, scrollable
- After copying, exits and prints "Copied to clipboard!" to terminal

## Clipboard Format

Output is copied as a markdown fenced code block:

````
```
$ git status
On branch main
Changes not staged for commit:
  modified:   src/main.rs
```
````

## CLI Interface

```
clip-hist <subcommand>

Subcommands:
  log     Append a command entry to history
  pick    Open the interactive TUI picker
  clear   Clear all history
  init    Print shell integration snippet for a given shell

Options:
  --help     Show help
  --version  Show version
```

### `clip-hist log`

```
clip-hist log --cmd <command> [--output <output>] [--exit-code <code>]
```

Called by shell hooks after each command. Appends to history file, prunes old entries.

### `clip-hist pick`

```
clip-hist pick
```

Opens TUI picker. No arguments needed.

### `clip-hist clear`

```
clip-hist clear
```

Removes all history entries.

### `clip-hist init`

```
clip-hist init <shell>    # shell: zsh | powershell
```

Prints the shell hook snippet to stdout for easy setup:

```bash
# Setup for zsh
clip-hist init zsh >> ~/.zshrc

# Setup for PowerShell
clip-hist init powershell >> $PROFILE
```

## Tech Stack

| Component      | Crate / Tool         |
|----------------|----------------------|
| Language        | Rust                |
| TUI framework  | ratatui + crossterm  |
| Clipboard      | arboard              |
| JSON parsing   | serde + serde_json   |
| CLI parsing     | clap                |
| File paths      | dirs                |

## Cross-Platform Support

| Feature           | macOS          | Windows         | Linux          |
|-------------------|----------------|-----------------|----------------|
| Shell hooks       | zsh            | PowerShell      | zsh/bash       |
| Clipboard         | arboard        | arboard         | arboard        |
| History path      | ~/.local/share | %APPDATA%       | ~/.local/share |
| TUI               | crossterm      | crossterm       | crossterm      |

## Open Questions

1. **Output capture strategy** — the `script` approach for zsh needs prototyping to confirm it doesn't break interactive programs (vim, less, etc.). May need a blocklist of commands that skip capture.
2. **PowerShell output capture** — `Start-Transcript` vs `Out-Default` proxy needs experimentation.
3. **Multi-select** — should users be able to select multiple commands at once? Deferred to v2.
4. **Bash support** — zsh hooks shown above, bash equivalent (`PROMPT_COMMAND` + `DEBUG` trap) can be added later.
