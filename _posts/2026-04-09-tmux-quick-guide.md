---
categories: [Technical]
math: false
---

# Tmux is the Terminal Multiplexer you crave

If you spend any meaningful time in the terminal, `tmux` will change how you work. It lets you run multiple terminal sessions inside a single window, keep processes alive after you disconnect, and organize your workspace with splits and tabs -- all from the keyboard.

---

## 1. What Is tmux?

**tmux** (terminal multiplexer) is a program that lets you:

- Run **multiple terminal sessions** inside one window
- **Detach** from a session and **reattach** later -- your processes keep running
- **Split** your terminal into panes (side-by-side or stacked)
- **Create tabs** (called "windows" in tmux) to organize different tasks

Think of it as a window manager for your terminal.

---

## 2. Why Use tmux?

- **Persistent Sessions** -- SSH into a remote server, start a long-running job, detach, close your laptop, go home, reattach -- the job is still running. No more `nohup` hacks or praying your WiFi holds up.

- **Workspace Organization** -- Working on a project that needs a dev server, a log tail, and an editor? Instead of juggling multiple terminal windows, split one tmux session into panes or tabs for each task.

- **Pair Programming and Sharing** -- Multiple people can attach to the same tmux session simultaneously. Useful for pair programming or live debugging with a colleague over SSH.

- **Keyboard-Driven Efficiency** -- Once you learn the keybindings, you rarely need to touch the mouse. Everything is a quick `Ctrl+b` combo away.

---

## 3. The Prefix Key

Almost every tmux command starts with the **prefix key**: `Ctrl+b`.

You press `Ctrl+b`, release, then press the next key. For example, `Ctrl+b c` means: press `Ctrl+b`, let go, then press `c`.

> Throughout this post, I'll write this as `Ctrl+b <key>`.

---

## 4. Session Management

Sessions are the top-level containers in tmux. Each session can have multiple windows (tabs), and each window can have multiple panes.

| Action | Command |
|---|---|
| **Create** a new named session | `tmux new -s "name"` |
| **List** all sessions | `tmux ls` |
| **Attach** to a session | `tmux attach -t "name"` |
| **Detach** from current session | `Ctrl+b d` |

### Example Workflow

```bash
# Start a new session for your project
tmux new -s myproject

# ... do some work ...

# Detach (you can close your terminal now)
# Press: Ctrl+b d

# Later, reattach
tmux attach -t myproject
```

---

## 5. Windows (Tabs)

Windows are like tabs within a session. You'll see them listed at the bottom of your tmux status bar.

| Action | Command |
|---|---|
| **Create** a new window | `Ctrl+b c` |
| **Create** a named window | `Ctrl+b` then type `:new-window -n name` |
| **Switch** to next window | `Ctrl+b n` |
| **Switch** to previous window | `Ctrl+b p` |
| **Switch** to window by number | `Ctrl+b <number>` |
| **Kill** current window | `Ctrl+b &` |
| **Kill** window (command mode) | `Ctrl+b` then type `:kill-window` |

> If you have `set -g mouse on` in your config, you can also just **click the tab** in the status bar.

---

## 6. Panes (Splits)

Panes let you divide a single window into multiple terminal views.

| Action | Command |
|---|---|
| **Split** horizontally (left/right) | `Ctrl+b %` |
| **Split** vertically (top/bottom) | `Ctrl+b "` |
| **Navigate** between panes | `Ctrl+b <arrow key>` |
| **Close** current pane | `Ctrl+d` or type `exit` |

---

## 7. Mouse Support

By default, tmux doesn't respond to mouse events. To enable scrolling and pane selection with the mouse, run this mid-session:

```
Ctrl+b then type :set mouse on
```

Or add it permanently to your `~/.tmux.conf`:

```bash
set -g mouse on
```

---

## 8. Quick Reference Card

Here's everything in one place:

```
SESSION COMMANDS
  tmux new -s "name"        Create named session
  tmux ls                   List sessions
  tmux attach -t "name"     Attach to session
  Ctrl+b d                  Detach from session

WINDOW (TAB) COMMANDS
  Ctrl+b c                  New window
  Ctrl+b n / p              Next / Previous window
  Ctrl+b <number>           Jump to window by number
  Ctrl+b &                  Kill window

PANE (SPLIT) COMMANDS
  Ctrl+b %                  Split horizontally
  Ctrl+b "                  Split vertically
  Ctrl+b <arrow>            Navigate panes

OTHER
  Ctrl+b :                  Enter command mode
  :set mouse on             Enable mouse support
```

---

## 9. Getting Started

Install tmux if you don't have it:

```bash
# macOS
brew install tmux

# Ubuntu/Debian
sudo apt install tmux

# Fedora
sudo dnf install tmux
```

Then just type `tmux` and start experimenting. The best way to learn is to use it for a week and let the keybindings become muscle memory.

---

tmux is one of those tools that feels like overhead at first, but once it clicks, you wonder how you ever worked without it.
