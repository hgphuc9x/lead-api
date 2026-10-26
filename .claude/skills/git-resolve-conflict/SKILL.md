---
name: git-resolve-conflict
description: >
  Resolve git merge conflicts intelligently.
  Use when user says "resolve conflict", "fix conflict", "merge conflict",
  "giải quyết conflict", or when git shows conflict markers.
argument-hint: "[file-path]"
allowed-tools:
  - Bash
  - Read
  - Write
---

# Merge Conflict Resolver

## Usage

```
/resolve-conflict                   → Resolve all conflicts
/resolve-conflict src/auth.ts       → Resolve specific file
/resolve-conflict --abort           → Abort merge, restore state
```

## Workflow

1. **List conflicted files**:
   ```bash
   git diff --name-only --diff-filter=U
   ```
   - If no conflicts → "No conflicts found"

2. **Get merge context**:
   ```bash
   git log --oneline -1 HEAD        # Current branch (ours)
   git log --oneline -1 MERGE_HEAD  # Incoming branch (theirs)
   ```

3. **For each conflicted file**:
   - Read file content (with conflict markers)
   - Identify feature from path → read `CONTEXT.md` if exists
   - Choose strategy from `./references/strategies.md`

4. **Show resolution plan** using format from `./templates/plan.md`

5. **Execute after confirm**:
   - `yes` → Resolve + `git add`
   - `no` → Abort
   - `file-by-file` → Confirm each file

6. **After resolve**: Show "Run `git commit` to complete merge"

## Rules

- **Read context first**: CONTEXT.md explains feature intent
- **Never auto-resolve**: Always show plan + confirm
- **Preserve both when safe**: Default to keeping both if independent
- **Ask when ambiguous**: Same function modified differently
- **No git commit**: Only resolve + stage, user commits manually