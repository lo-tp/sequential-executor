---
name: toggle-auto-commit
description: Toggle auto-commit mode on or off. When on, the agent commits after each logical change.
---

# Toggle Auto Commit

Toggle the auto-commit flag.

## Action

1. Check if `/tmp/pi-auto-commit` exists.
   - If it does, delete it → auto-commit is now **off**.
   - If it does not, create it (empty file) → auto-commit is now **on**.
2. Report the new state: "Auto-commit: on" or "Auto-commit: off".

## Behavior when on

When `/tmp/pi-auto-commit` exists, after completing a logical unit of work (a feature, a fix, a refactor), commit it with a descriptive message without being asked.
