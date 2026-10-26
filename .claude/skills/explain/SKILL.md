---
name: explain
description: >
  Explain code, concepts, flow, or decisions for beginners.
  Use when user says "explain", "giải thích", "how does this work",
  "what is", "tại sao", "why", or wants to understand code/concepts.
argument-hint: "[code|concept|flow|why] [target]"
allowed-tools:
  - Read
---

# Explain for Beginners

## Usage

```
/explain code [feature-name]        → What the code does
/explain concept [concept-name]     → Pattern/concept explanation
/explain flow [feature-name]        → Request/data flow
/explain why [decision]             → Reasoning behind decisions
```

## Workflow

1. **Ask language**: "English or Vietnamese? (en/vi)"

2. **Read docs FIRST (no source scanning):**

| Mode | Read in order |
|------|---------------|
| `code` | CONTEXT.md of feature → BE/FE-ARCHITECTURE.md |
| `concept` | BE/FE-PROJECT-RULES.md → ARCHITECTURE.md |
| `flow` | API_SPEC.md → CONTEXT.md of feature |
| `why` | PROJECT-RULES.md → DATABASE.md (if relevant) |

3. **Only read specific source file** if user points to exact file

4. **Use template**: `./templates/{en|vi}.md`

## Doc Locations

```
01-share-docs/
├── DATABASE.md
└── API_SPEC.md

backend-nest-hoidanit/docs/
├── BE-PROJECT-RULES.md
└── BE-ARCHITECTURE.md

frontend-react-hoidanit/docs/
├── FE-PROJECT-RULES.md
└── FE-ARCHITECTURE.md

src/features/{feature}/CONTEXT.md   ← Feature-specific context
```

## Rules

- **NEVER Glob/scan source code**
- Docs contain all architectural decisions
- CONTEXT.md has feature-specific details
- Only read source when user gives exact file path

## Error Handling

| Error | Action |
|-------|--------|
| Missing mode | Ask: code, concept, flow, or why? |
| No CONTEXT.md | Read ARCHITECTURE.md instead |
| Need source detail | Ask user for specific file path |