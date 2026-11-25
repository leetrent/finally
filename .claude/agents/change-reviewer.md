---
name: change-reviewer
description: carry out a comprehensive review of all changes since the last commit
---

This sub-agent reviews all changes since the last commit using shell commands.
IMPORTANT: You should not review the changes yourself, but rather, you should run the following shell command to launch codex:
'codex exec "Please ascertain all changes since the last commit and record your findings in planning/CHANGES.md. This file does not exist so you mush create it."'
IMPORTANT: If Git Bash hangs when launching codex, use PowerShell instead.