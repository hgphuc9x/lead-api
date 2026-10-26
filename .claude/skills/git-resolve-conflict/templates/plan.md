# Resolution Plan Template

## Format

```
🔀 CONFLICT RESOLUTION PLAN

Merging: {source_branch} → {target_branch}
Reason: {merge commit message or PR title if available}

Conflicts ({count} files):

┌─ {file_path_1} ────────────────────────────────────────┐
│ OURS ({target}):   {description of our changes}       │
│ THEIRS ({source}): {description of their changes}     │
│ STRATEGY:          {Keep both | Keep ours | ...}      │
│ RISK:              {Low | Medium | High}              │
└────────────────────────────────────────────────────────┘

┌─ {file_path_2} ────────────────────────────────────────┐
│ OURS ({target}):   {description}                      │
│ THEIRS ({source}): {description}                      │
│ STRATEGY:          {strategy}                         │
│ RISK:              {risk level}                       │
│ ⚠️ NEEDS REVIEW:   {reason if high risk}              │
└────────────────────────────────────────────────────────┘

Summary:
  ✓ Auto-resolve: {n} files
  ⚠️ Need review: {n} files

Proceed? (yes / no / file-by-file / show-diff)
```

---

## Confirm Responses

| Response | Action |
|----------|--------|
| `yes`, `y` | Resolve all files as planned |
| `no`, `n`, `abort` | Abort, no changes |
| `file-by-file`, `one` | Confirm each file individually |
| `show-diff`, `diff` | Show detailed diff for each file |
| `skip {file}` | Skip specific file, resolve others |

---

## After Resolution

```
✅ CONFLICTS RESOLVED

Resolved files:
  ✓ src/features/cart/cart.service.ts
  ✓ src/features/cart/cart.controller.ts

Staged and ready. Next steps:
  $ git commit                    # Complete merge
  $ git merge --abort             # Cancel merge (if needed)
```

---

## Error Cases

### No conflicts
```
✅ No conflicts found.

Status: {clean | merge in progress | rebase in progress}
```

### Cannot auto-resolve
```
❌ Cannot auto-resolve some files.

Files needing manual resolution:
  • src/config/constants.ts - Conflicting values
  • src/shared/types.ts - Complex logic overlap

Options:
  1. Resolve manually, then run /resolve-conflict again
  2. Run /resolve-conflict {file} for each file
  3. Abort merge: git merge --abort
```