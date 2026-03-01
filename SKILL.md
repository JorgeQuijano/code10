---
name: power-of-ten-typescript
description: "The Power of 10 rules adapted for TypeScript: 10 rules for developing safety-critical TypeScript code. Use this skill when writing TypeScript code that requires high reliability, fewer bugs, and easier static analysis."
user-invocable: true
---

# Power of 10: TypeScript Rules for Safety-Critical Code

The Power of 10 rules originated from NASA's Jet Propulsion Laboratory as guidelines for writing safety-critical code. This skill adapts those principles for TypeScript development.

---

## Rule 1: Simple Control Flow

- **Avoid:** Recursion, nested callbacks (3+ levels), callback hell
- **Preferred:** for/while loops, Array methods, async/await

```typescript
// ❌ Avoid - nested callbacks
getData((err, data) => {
  processData(data, (err2, result) => {
    saveResult(result, (err3) => { /* ... */ });
  });
});

// ✅ Preferred - async/await
const data = await getData();
const result = await processData(data);
await saveResult(result);
```

---

## Rule 2: Bounded Loops

- **Avoid:** while(true), unclear termination
- **Preferred:** Explicit bounds, max iterations

```typescript
// ❌ Avoid - unclear termination
while (true) {
  const item = queue.pop();
  if (!item) break;
  process(item);
}

// ✅ Preferred - explicit bounds
const MAX_ITEMS = 1000;
for (let i = 0; i < Math.min(queue.length, MAX_ITEMS); i++) {
  process(queue[i]);
}
```

---

## Rule 3: No Dynamic Allocation in Hot Paths

- **Avoid:** new objects in loops, closures capturing large scope
- **Preferred:** Pre-allocate, reuse objects

```typescript
// ❌ Avoid - creating objects in loop
function processItems(items: string[]) {
  return items.map(item => new ProcessedItem(item));
}

// ✅ Preferred - reuse objects, use primitives
function processItems(items: string[]): string[] {
  return items.map(item => item.trim().toLowerCase());
}
```

---

## Rule 4: Small Functions

- **Avoid:** Functions > 40 lines, 3+ nesting levels
- **Preferred:** Single responsibility, extract helpers

```typescript
// ❌ Avoid - large function with deep nesting
async function handleUserRequest(req: Request) {
  if (req.method === 'GET') {
    const user = await db.findUser(req.userId);
    if (user) {
      if (user.isActive) {
        // ... 50 more lines
      }
    }
  }
}

// ✅ Preferred - small, focused functions
async function handleUserRequest(req: Request): Promise<Response> {
  if (!isGetRequest(req)) return badRequest();
  const user = await fetchUser(req.userId);
  if (!user) return notFound();
  return ok(await serializeUser(user));
}
```

---

## Rule 5: Assertive Coding

- **Avoid:** Missing null checks, assuming valid input
- **Preferred:** assert/zod, validate external input

```typescript
// ❌ Avoid - assume valid input
function calculateShipping(order: Order): number {
  return order.items.reduce((sum, item) => sum + item.price, 0);
}

// ✅ Preferred - validate and assert
import { z } from 'zod';

const OrderSchema = z.object({
  items: z.array(z.object({ price: z.number().positive() })).min(1)
});

function calculateShipping(order: unknown): number {
  const validOrder = OrderSchema.parse(order);
  return validOrder.items.reduce((sum, item) => sum + item.price, 0);
}
```

---

## Rule 6: Minimize Scope

- **Avoid:** Global variables, unused class properties
- **Preferred:** const by default, declare at use point

```typescript
// ❌ Avoid - global state
let globalCounter = 0;
class Counter {
  increment() { globalCounter++; }
}

// ✅ Preferred - local scope
function createCounter() {
  let counter = 0;
  return {
    increment: () => counter++
  };
}
```

---

## Rule 7: Validate All Inputs

- **Avoid:** any type, ignoring strict mode
- **Preferred:** strict: true, runtime validation

```typescript
// ❌ Avoid - any type
function parseData(data: any): any {
  return JSON.parse(data);
}

// ✅ Preferred - strict typing
interface UserData {
  id: string;
  name: string;
  email: string;
}

function parseData(data: string): UserData {
  const parsed = JSON.parse(data) as UserData;
  if (!parsed.id || !parsed.name || !parsed.email) {
    throw new Error('Invalid user data');
  }
  return parsed;
}
```

---

## Rule 8: Limit Metaprogramming

- **Avoid:** Dynamic requires, complex conditionals
- **Preferred:** Static imports, simple env config

```typescript
// ❌ Avoid - dynamic requires
function loadDriver(name: string) {
  return require(\`./drivers/\${name}\`);
}

// ✅ Preferred - static imports
import { PostgresDriver } from './drivers/postgres';
import { MySQLDriver } from './drivers/mysql';

const drivers = { postgres: PostgresDriver, mysql: MySQLDriver };
function getDriver(type: 'postgres' | 'mysql'): Driver {
  return new drivers[type]();
}
```

---

## Rule 9: Restrict Object Mutation

- **Avoid:** Deep nesting (a.b.c.d), chained optional
- **Preferred:** Flat structures, immutable updates

```typescript
// ❌ Avoid - deep nesting
const user = getUser();
const city = user?.profile?.address?.city?.toString();

// ✅ Preferred - flat, explicit
interface Address { city: string; }
interface Profile { address: Address; }
interface User { profile: Profile; }

function getCity(user: User | null): string {
  if (!user?.profile?.address) return 'Unknown';
  return user.profile.address.city;
}

// Immutable updates
const updateUser = (user: User, name: string): User => ({
  ...user,
  name,
  profile: {
    ...user.profile,
    address: user.profile.address // reference unchanged
  }
});
```

---

## Rule 10: Strict Compilation

- **Avoid:** Disabling strict mode, @ts-ignore
- **Preferred:** strict: true, ESLint, zero warnings

```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  }
}
```

```yaml
# .eslintrc.yml
rules:
  no-unused-vars: error
  no-implicit-any: error
  @typescript-eslint/strict-boolean-expressions: error
```

---

## Summary Checklist

- [ ] Use async/await over callbacks
- [ ] Always bound loops with explicit limits
- [ ] Avoid object creation in hot paths
- [ ] Keep functions under 40 lines
- [ ] Validate all external input with Zod or similar
- [ ] Use const and minimize scope
- [ ] Avoid \`any\` type, enable strict mode
- [ ] Prefer static imports over dynamic requires
- [ ] Flatten object structures, use immutable updates
- [ ] Enable strict TS config, zero warnings
