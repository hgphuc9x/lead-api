# Commit Conventions

## Format

```
<type>(<scope>): <subject>

<body>

<footer>
```

## Types

| Type | When to use | Example changes |
|------|-------------|-----------------|
| `feat` | New feature | Add endpoint, new component, new entity |
| `fix` | Bug fix | Fix crash, correct logic, handle edge case |
| `refactor` | Code restructure (no behavior change) | Rename, extract function, reorganize |
| `docs` | Documentation | README, comments, CONTEXT.md |
| `style` | Formatting (no code change) | Prettier, eslint fixes, whitespace |
| `test` | Add/update tests | spec files, e2e tests |
| `chore` | Maintenance | Dependencies, configs, scripts |
| `perf` | Performance | Optimize query, reduce bundle |
| `ci` | CI/CD | GitHub Actions, Docker |
| `build` | Build system | Webpack, tsconfig |
| `revert` | Revert commit | Undo previous commit |

## Auto-detect Type

| Changed files/content | Detected type |
|-----------------------|---------------|
| `*.spec.ts`, `*.test.ts` only | `test` |
| `README.md`, `CONTEXT.md`, `*.md` only | `docs` |
| `package.json`, `tsconfig.json`, `.eslintrc` only | `chore` |
| `.github/workflows/*` | `ci` |
| `Dockerfile`, `docker-compose.yml` | `build` |
| New files + new exports | `feat` |
| Fix in existing logic, error handling | `fix` |
| Rename, move files, no logic change | `refactor` |

## Scope Detection

| File path | Scope |
|-----------|-------|
| `src/features/auth/*` | `auth` |
| `src/features/product/*` | `product` |
| `src/features/cart/*` | `cart` |
| `src/shared/*` | `shared` |
| `src/config/*` | `config` |
| Multiple features | omit scope or use `core` |

## Subject Rules

- Imperative mood: "add" not "adds" or "added"
- Lowercase first letter
- No period at end
- Max 50 characters
- Complete the sentence: "This commit will..."

## Body Rules

- Separate from subject by blank line
- Wrap at 72 characters
- Explain WHAT and WHY, not HOW
- Use bullet points for multiple changes

## Examples

### Simple feature
```
feat(auth): add JWT refresh token endpoint
```

### Feature with body
```
feat(auth): add JWT refresh token endpoint

- Add POST /auth/refresh endpoint
- Create RefreshToken entity with rotation
- Add 7-day expiry for refresh tokens
```

### Bug fix
```
fix(cart): correct quantity validation

Quantity was allowing negative values which caused
checkout to fail silently.

Closes #123
```

### Breaking change
```
feat(api)!: change response format to envelope pattern

BREAKING CHANGE: All API responses now wrapped in
{ success, data, error } format.
```

### Multiple scopes (avoid if possible)
```
refactor: reorganize shared utilities

- Move hash utils to shared/utils
- Extract pagination helper
- Update imports across features
```