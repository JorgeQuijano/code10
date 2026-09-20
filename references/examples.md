# Power of 10 for TypeScript: before and after

Pairs written so the two sides are equivalent — same inputs, same outputs, same side effects. A rewrite that changes behaviour is not a Rule fix, it is a bug with a comment attached.

## Rule 1 — Simple control flow

```typescript
// Before: callback nest, depth 3, error path unreachable if process throws
getData((err, data) => {
  processData(data, (err2, result) => {
    saveResult(result, (err3) => { log(err3); });
  });
});
```

```typescript
// After: flat, and the compiler checks that nothing is ignored
const data = await getData();
const result = await processData(data);
await saveResult(result);
```

Recursion → explicit queue (the rewrite that also gives you the bound):

```typescript
// Before
function walk(node: Node): number {
  return 1 + node.children.reduce((sum, child) => sum + walk(child), 0);
}
```

```typescript
// After: no stack growth, and depth is now boundable
function walk(root: Node): number {
  let count = 0;
  const queue: Node[] = [root];
  while (queue.length > 0 && count < MAX_NODES) {
    const node = queue.pop();
    if (node === undefined) break;
    count += 1;
    queue.push(...node.children);
  }
  return count;
}
```

## Rule 2 — Bounded loops

```typescript
// Before: termination depends on the queue being finite and pop() behaving
while (true) {
  const item = queue.pop();
  if (item === undefined) break;
  process(item);
}
```

```typescript
// After: same work, stated bound. shift() preserves the original order.
for (let i = 0; i < MAX_ITEMS && queue.length > 0; i += 1) {
  const item = queue.shift();
  if (item === undefined) break;
  process(item);
}
```

## Rule 3 — Bounded memory

```typescript
// Before: memory grows with input volume, and nothing upstream bounds it
async function total(ids: readonly string[]): Promise<number> {
  const events = [];
  for (const id of ids) {
    events.push(...(await fetchEvents(id)));
  }
  return events.reduce((sum, event) => sum + event.amount, 0);
}
```

```typescript
// After: same number, constant memory
async function total(ids: readonly string[]): Promise<number> {
  let sum = 0;
  for (const id of ids) {
    for (const event of await fetchEvents(id)) {
      sum += event.amount;
    }
  }
  return sum;
}
```

The other half of Rule 3 — allocation inside a proven hot loop — is measurement-gated. Do not "fix" it:

```typescript
// Not a Rule 3 fix. Modern engines handle short-lived objects; this obscures the
// code and only pays off where a profile shows GC pressure.
const buf = new Float64Array(1024);
```

If you do build a cache, bound it: `new Map()` with an eviction policy and a max size, not an unbounded map keyed by request.

## Rule 4 — Small functions

```typescript
// Before: one function, four levels, three responsibilities
async function handle(req: Request): Promise<Response> {
  if (req.method === 'GET') {
    const user = await db.findUser(req.userId);
    if (user) {
      if (user.isActive) {
        // ... 50 more lines
      }
    }
  }
}
```

```typescript
// After: guard clauses, one job each
async function handle(req: Request): Promise<Response> {
  if (req.method !== 'GET') return methodNotAllowed();
  const user = await findActiveUser(req.userId);
  if (user === undefined) return notFound();
  return ok(serializeUser(user));
}
```

## Rule 5 — Assertions, not assumptions

```typescript
// Before: parses nothing, asserts everything
function orderTotal(data: string): number {
  const order = JSON.parse(data) as Order;
  return order.items.reduce((sum, item) => sum + item.price, 0);
}
```

```typescript
// After: parse once, then work with the proven type
import { z } from 'zod';

const OrderSchema = z.object({
  items: z.array(z.object({ price: z.number().positive() })).min(1),
});

function orderTotal(data: unknown): number {
  const order = OrderSchema.parse(data);
  return order.items.reduce((sum, item) => sum + item.price, 0);
}
```

`OrderSchema.parse` narrows `unknown` to the schema's type, so Rule 5 and Rule 7 never disagree about how a boundary is handled.

## Rule 6 — Smallest possible scope

```typescript
// Before: module-level mutable state, one owner, every reader affected
let counter = 0;
export function increment(): void { counter += 1; }
```

```typescript
// After: no shared state at all when the value is only used here
export function createCounter(): { increment: () => number } {
  let counter = 0;
  return { increment: () => (counter += 1) };
}
```

## Rule 7 — Check every return value

```typescript
// Before: the promise and the return value are both dropped
function fire(req: Request): void {
  saveAudit(req.id);
  db.findUser(req.id);
}
```

```typescript
// After: awaited, or discarded with a stated reason
async function fire(req: Request): Promise<void> {
  await saveAudit(req.id);
  const user = await db.findUser(req.id);
  if (user === undefined) throw new Error(`no user for ${req.id}`);
}
```

Exhaustive unions, no fallthrough:

```typescript
// Before: a new status silently does nothing
function label(status: Status): string {
  if (status === 'open') return 'Open';
  return 'Closed';
}
```

```typescript
// After: adding a member breaks the build
function label(status: Status): string {
  switch (status) {
    case 'open':
      return 'Open';
    case 'closed':
      return 'Closed';
  }
}
```

## Rule 8 — No metaprogramming

```typescript
// Before: no tool can resolve this import
function loadDriver(name: string): Driver {
  return require(`./drivers/${name}`);
}
```

```typescript
// After: static imports, exhaustive lookup, unknown key rejected at the boundary
import { postgresDriver } from './drivers/postgres';
import { mysqlDriver } from './drivers/mysql';

const drivers = { postgres: postgresDriver, mysql: mysqlDriver };

function loadDriver(type: 'postgres' | 'mysql'): Driver {
  return drivers[type];
}
```

## Rule 9 — No mutation, shallow indirection

```typescript
// Before: mutates the caller's object, then chases four levels
function rename(order: Order, name: string): string {
  order.name = name.trim();
  order.tags.push('renamed');
  return order.customer?.profile?.address?.city?.toString() ?? '';
}
```

```typescript
// After: new value out, caller's object untouched, absence is a branch
interface Customer { readonly address: Address; }
interface Address { readonly city: string; }
interface Order {
  readonly name: string;
  readonly tags: readonly string[];
  readonly customer: Customer | undefined;
}

function rename(order: Order, name: string): Order {
  return { ...order, name: name.trim(), tags: [...order.tags, 'renamed'] };
}

function city(order: Order): string {
  if (order.customer === undefined) return 'Unknown';
  return order.customer.address.city;
}
```

## Rule 10 — Strict compilation

```json
// Before: strict off, and the suppression habit follows
{ "compilerOptions": { "strict": false, "skipLibCheck": true } }
```

```json
// After: see references/checks.md for the full block
{
  "compilerOptions": {
    "strict": true,
    "exactOptionalPropertyTypes": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "useUnknownInCatchVariables": true
  }
}
```

```json
// Before: assertions, half of them invisible to the compiler
function parse(data: any): UserData {
  return JSON.parse(data) as UserData;
}
```

```typescript
// After: the boundary is the only place that knows about `unknown`
function parseUser(data: unknown): UserData {
  return UserSchema.parse(data);
}
```
