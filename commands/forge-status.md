---
name: forge:status
description: Check the status of all running or recently completed product-forge tasks
---

Check the status of product-forge tasks:

1. Run: `ls ~/projects/ 2>/dev/null | sort -r | head -20` to list recent projects
2. For each project that has `.product-forge/` directory:
   - Check if `summary.md` exists (completed)
   - Check the latest `critique_*.md` (in progress or failed)
   - Report: project name, status, current phase
3. Also check for any running background processes:
   `ps aux | grep -E 'claude|codex' | grep -v grep`

Format the response as a clean status table.
