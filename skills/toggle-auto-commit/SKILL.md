---
name: toggle-auto-commit
description: Toggle auto-commit mode. When /tmp/pi-auto-commit exists, commit after each logical change without being asked. Check for this flag at the start of any coding session.
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
