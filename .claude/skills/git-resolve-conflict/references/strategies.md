# Resolution Strategies

## Quick Reference

| Conflict Pattern | Strategy | Risk |
|------------------|----------|------|
| Different functions added | Keep both | Low |
| Different imports added | Keep both, sort | Low |
| Same config key changed | Ask user | High |
| Same function body changed | Analyze + ask | High |
| One adds, one deletes | Ask user | High |
| Formatting only | Keep either | Low |

---

## Pattern 1: Independent Additions

**Scenario**: Both branches add different things

```typescript
<<<<<<< HEAD
import { JwtService } from './jwt.service';
=======
import { HashService } from './hash.service';
>>>>>>> feature/auth
```

**Resolution**: Keep both
```typescript
import { HashService } from './hash.service';
import { JwtService } from './jwt.service';
```

---

## Pattern 2: Same File, Different Functions

**Scenario**: Both add new methods to same class

```typescript
class CartService {
  // existing code...

<<<<<<< HEAD
  getTotal(): number {
    return this.items.reduce((sum, i) => sum + i.price, 0);
  }
=======
  clear(): void {
    this.items = [];
  }
>>>>>>> feature/cart
}
```

**Resolution**: Keep both methods
```typescript
class CartService {
  // existing code...

  getTotal(): number {
    return this.items.reduce((sum, i) => sum + i.price, 0);
  }

  clear(): void {
    this.items = [];
  }
}
```

---

## Pattern 3: Same Function Modified (⚠️ ASK USER)

**Scenario**: Both modify same function differently

```typescript
<<<<<<< HEAD
async createOrder(dto: CreateOrderDto) {
  // Added validation
  if (!dto.items.length) throw new BadRequestException('Empty cart');
  return this.orderRepo.save(dto);
}
=======
async createOrder(dto: CreateOrderDto) {
  // Added logging
  this.logger.log(`Creating order for user ${dto.userId}`);
  return this.orderRepo.save(dto);
}
>>>>>>> feature/order
```

**Resolution**: Must merge manually - KEEP BOTH changes
```typescript
async createOrder(dto: CreateOrderDto) {
  // Added validation (from main)
  if (!dto.items.length) throw new BadRequestException('Empty cart');
  // Added logging (from feature/order)
  this.logger.log(`Creating order for user ${dto.userId}`);
  return this.orderRepo.save(dto);
}
```

**Action**: Show both to user, propose merged version, ask confirm.

---

## Pattern 4: Config/Constant Changes (⚠️ ASK USER)

**Scenario**: Same config value changed differently

```typescript
<<<<<<< HEAD
export const MAX_CART_ITEMS = 50;
=======
export const MAX_CART_ITEMS = 100;
>>>>>>> feature/cart
```

**Resolution**: Cannot auto-decide. Ask user:
```
⚠️ CONFIG CONFLICT

MAX_CART_ITEMS changed in both branches:
  - main: 50
  - feature/cart: 100

Which value should we keep? (ours/theirs/custom)
```

---

## Pattern 5: Package.json Dependencies

**Scenario**: Both add different dependencies

```json
<<<<<<< HEAD
  "dependencies": {
    "bcrypt": "^5.1.0"
  }
=======
  "dependencies": {
    "jsonwebtoken": "^9.0.0"
  }
>>>>>>> feature/auth
```

**Resolution**: Merge both, sort alphabetically
```json
  "dependencies": {
    "bcrypt": "^5.1.0",
    "jsonwebtoken": "^9.0.0"
  }
```

---

## Pattern 6: One Deletes, Other Modifies (⚠️ ASK USER)

**Scenario**: One branch deletes code that other branch modified

```typescript
<<<<<<< HEAD
// Function was deleted in main
=======
// Function was updated in feature branch
function oldHelper() {
  // new implementation
}
>>>>>>> feature/x
```

**Resolution**: Ask user
```
⚠️ DELETE vs MODIFY CONFLICT

`oldHelper()` was:
  - DELETED in main (intentionally removed?)
  - MODIFIED in feature/x (still needed?)

Keep the function? (yes/no)
```

---

## Pattern 7: Entity/DTO Fields

**Scenario**: Both add fields to same entity

```typescript
export class User {
  id: number;
  email: string;
<<<<<<< HEAD
  phoneNumber: string;  // Added for profile
=======
  avatar: string;       // Added for UI
>>>>>>> feature/profile
}
```

**Resolution**: Keep both fields
```typescript
export class User {
  id: number;
  email: string;
  avatar: string;
  phoneNumber: string;
}
```

---

## Decision Tree

```
Conflict detected
    │
    ├─ Different locations in file?
    │   └─ YES → Keep both (safe)
    │
    ├─ Same location, different content?
    │   ├─ Imports/dependencies? → Merge + sort
    │   ├─ New functions/methods? → Keep both
    │   ├─ Same function modified? → ⚠️ Ask user
    │   └─ Config values? → ⚠️ Ask user
    │
    └─ Delete vs Modify?
        └─ ⚠️ Always ask user
```

---

## Output Messages

### Safe resolution
```
✓ Resolved: src/features/cart/cart.service.ts
  Strategy: Keep both (independent additions)
```

### Needs confirmation
```
⚠️ Manual review needed: src/features/order/order.service.ts
  Both branches modified createOrder() differently.
  
  [Show proposed merge]
  
  Accept this resolution? (yes/no/show-diff)
```

### Abort option
```
❌ Cannot auto-resolve: src/config/constants.ts
  Conflicting values for same constant.
  
  Options:
  1. Keep ours (main): MAX_ITEMS = 50
  2. Keep theirs (feature): MAX_ITEMS = 100
  3. Enter custom value
  4. Skip this file
  
  Choice? (1/2/3/4)
```