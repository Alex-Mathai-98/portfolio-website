## Learning how to use Claude to run jobs in the background.

When you run a command in the background using Claude Code, it creates a
  background shell with a unique ID. Here's how the killing process works:

  1. Background shells get IDs: Each background command gets assigned a shell
  ID (like "shell_1", "shell_2", etc.)
  2. Finding the shell ID: You can use the /bashes command in Claude Code to
  list all running background shells and their IDs
  3. Using BashOutput: The BashOutput tool lets you check what a background
  shell is doing by providing its shell ID. This also shows you the shell ID
  if you forgot it.
  4. Using KillBash: The KillBash tool terminates a specific background shell
  by providing its shell ID.

  Example workflow:
  - Start: bash tools/run.sh runs in background, gets ID "shell_1"
  - Check: Use BashOutput with "shell_1" to see Jekyll server output
  - Kill: Use KillBash with "shell_1" to stop the Jekyll server

  This gives you precise control - you can run multiple background processes
  and kill specific ones without affecting others.