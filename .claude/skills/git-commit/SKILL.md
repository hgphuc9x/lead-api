---
name: git-commit
description: >
  Write conventional commit messages.
  Use when user says "commit", "git commit", "save changes",
  "tạo commit", "viết commit message", or finished a task and wants to commit.
argument-hint: "[type] [message]"
allowed-tools:
  - Bash
  - Read
---

# Commit Message Generator

## Usage

```
/commit                     → Auto-detect type from changes
/commit feat add login      → Quick commit with type + message
/commit --amend             → Amend last commit message
```

## Workflow

1. **Check staged changes**:
   ```bash
   git diff --staged --stat
   ```
   - If nothing staged → suggest `git add` first

2. **Read project conventions**: `CLAUDE.md` (commit section if exists)

3. **Analyze changes**:
   ```bash
   git diff --staged
   ```

4. **Detect type** from changed files/content (see `./references/conventions.md`)

5. **Generate message** → show preview:

   ```
   📝 COMMIT PREVIEW
   
   feat(auth): add JWT refresh token endpoint
   
   - Add POST /auth/refresh endpoint
   - Create RefreshToken entity
   - Add token rotation logic
   
   Staged files (3):
     M src/features/auth/auth.controller.ts
     A src/features/auth/entities/refresh-token.entity.ts
     M src/features/auth/auth.service.ts
   
   Commit? (yes/no/edit)
   ```

6. **Execute after confirm**:
   - `yes` → `git commit -m "..."`
   - `edit` → user modifies, then commit
   - `no` → abort

## Quick Commit

Skip preview for simple commits:

```
/commit fix typo in readme
    ↓
git commit -m "fix: typo in readme"
```

## Rules

- **Staged only**: Never auto `git add`
- **Scope from path**: `src/features/auth/` → `(auth)`
- **Lowercase**: Type and scope always lowercase
- **No period**: Don't end subject with `.`
- **Imperative**: "add" not "added"
- **50/72 rule**: Subject ≤50 chars, body wrap at 72