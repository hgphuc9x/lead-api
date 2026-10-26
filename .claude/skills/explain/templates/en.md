# English Templates

## Code Explanation

```markdown
## 📁 File: {filename}

### Purpose
[1-2 sentences describing what this file does]

### Breakdown

**Imports**
- Import X from Y for [purpose]

**Main class/function**
- [Explain logic]
- [Use real-world analogy if helpful]

**Methods**
- `methodX()`: [what it does, when called]

### 💡 Key Points
- Point 1
- Point 2

### 🔗 Connections
- Called by: ...
- Calls: ...
```

---

## Concept Explanation

```markdown
## 🎯 {Concept Name}

### What is it?
[Simple explanation, 2-3 sentences]

### Real-world Analogy
[Compare to something familiar]

Example: Repository Pattern is like a **librarian**:
- You don't search the storage yourself
- You ask the librarian (Repository) to find it
- Librarian knows where things are and how to find them fast

### In code

```typescript
// Short code example
```

### Why use it?
- Reason 1
- Reason 2

### Without it?
[Problems you'd face]

### 📚 Learn more
- Keywords to search
```

---

## Flow Explanation

```markdown
## 🔄 Flow: {Action Name}

### Overview
[1-2 sentences describing this flow]

### Diagram

```
[Client]
    │
    ▼
[Controller] ──── Receive request, validate input
    │
    ▼
[Service] ──────── Business logic, orchestrate
    │
    ▼
[Repository] ───── Query database
    │
    ▼
[Database]
    │
    ▼
[Response] ◄────── Transform & return
```

### Step by Step

**Step 1: Client sends request**
- Endpoint: POST /products
- Body: { name, price, ... }

**Step 2: Controller receives**
- Validate DTO
- Extract user from JWT (if needed)
- Call Service

**Step 3: Service processes**
- Business logic
- Call Repository

**Step 4: Repository queries**
- Build query
- Execute

**Step 5: Response**
- Transform data
- Return to client

### 🔍 Debug tips
- If error at step X, check...
```

---

## Why Explanation

```markdown
## ❓ Why: {Question}

### Short Answer
[1-2 sentences, direct answer]

### Detailed Explanation

**Reason 1: ...**
[Explain]

**Reason 2: ...**
[Explain]

### Without it?
[Problems you'd face]

### Trade-offs
| Pros | Cons |
|------|------|
| ... | ... |

### Alternatives?
[Other options and when to use them]

### 📌 Conclusion
[Summary recommendation]
```