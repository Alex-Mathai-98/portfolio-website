---
categories: [Technical]
math: false
---

# Tmux is the Terminal Multiplexer you crave

If you spend any meaningful time in the terminal, `tmux` will change how you work. It lets you run multiple terminal sessions inside a single window, keep processes alive after you disconnect, and organize your workspace with splits and tabs -- all from the keyboard.

---

## Expert Workflow

If you only take two rules away from this post, take these. They are what separates someone who *uses* tmux from someone who actually gets value out of it on a remote machine.

### Rule 1: One tmux session per machine

When you SSH into a machine, attach to (or create) **a single tmux session for that machine** -- not one per task. If you need to run multiple things in parallel on the same machine, open new **windows** inside that one session instead of spawning new sessions.

Why? Because sessions are easy to lose track of (you have to `tmux ls` to even see them), but windows are visible in the status bar at the bottom of your screen at all times. One session per machine keeps your mental model simple: *one machine = one place to look*.

| Action | Keys |
|---|---|
| Create / attach to a session | `tmux new -s gpu1` / `tmux attach -t gpu1` |
| Open a new **named** window inside the session | `Ctrl+b` then type `:new-window -n <name>` |
| Switch between windows | `Ctrl+b n` (next), `Ctrl+b p` (previous), `Ctrl+b <number>` |

### Rule 2: Always rename the window after the experiment running in it

The default window name is whatever shell you're running (`bash`, `zsh`, ...), which is useless when you have four windows open. **The moment you kick off an experiment in a window, rename that window to the experiment's short code.** This costs you two seconds and saves you minutes every time you tab back to check on something.

| Action | Keys |
|---|---|
| Rename current window | `Ctrl+b ,` (opens a prompt -- type the name and press Enter) |

### Walkthrough: running two experiments on a remote GPU box

You SSH into `gpu1` to run two ablations in parallel: `lr-3e4-bs64` and `lr-1e3-bs128`.

**Step 1.** SSH in and start the one session for this machine. Name the first window after the experiment you're about to run -- the `-n` flag on `tmux new` lets you name window `0` on creation:

```bash
ssh gpu1
tmux new -s gpu1 -n lr-3e4-bs64    # session "gpu1", first window named "lr-3e4-bs64"
# or: tmux attach -t gpu1          # if the session already exists
```

Kick off the first experiment in this window:

```bash
python train.py --lr 3e-4 --bs 64
```

**Step 2.** Open a second window for the other experiment, named on creation:

```
Ctrl+b :new-window -n lr-1e3-bs128
```

You're dropped into a fresh shell in the new window. Start the second run:

```bash
python train.py --lr 1e-3 --bs 128
```

Your status bar at the bottom now reads something like:

```
[gpu1] 0:lr-3e4-bs64   1:lr-1e3-bs128*
```

Detach with `Ctrl+b d`, close your laptop, come back tomorrow, `ssh gpu1 && tmux attach -t gpu1`, and you immediately know which window is which without having to peek at logs. With mouse mode on (see [Mouse Support](#7-mouse-support)) you can just click a window's name in the status bar to jump to it.

**Step 3.** When `lr-3e4-bs64` finishes and you want to reuse window `0` for a *different* experiment (say `lr-5e4-bs64`), **rename the window before starting the new run**. This is the one moment where `Ctrl+b ,` earns its keep -- otherwise the window's name lies about what's running in it, which is worse than no name at all.

Click on window `0` in the status bar (or `Ctrl+b p` if you don't have mouse mode on), then rename it:

```
Ctrl+b ,                  # prompt appears at the bottom
                          # backspace the old name, type: lr-5e4-bs64
                          # press Enter
```

Then start the new experiment:

```bash
python train.py --lr 5e-4 --bs 64
```

That is the entire workflow. The rest of this post is the reference material that supports it.

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
| **Rename** current window | `Ctrl+b ,` (opens a prompt -- type the name, e.g. the experiment name) |
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
  Ctrl+b ,                  Rename current window
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
