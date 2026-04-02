# TypeScript Style Guide (Strict Enforcement)

These rules are **mandatory** for all TypeScript code in Solana projects. No exceptions.

## Type System

### Always Use `Array<T>`, Never `item[]`

For consistency with other generic syntax (`Promise<T>`, `Map<K, V>`, `Set<T>`):

```typescript
// CORRECT
const users: Array<User> = [];
const ids: Array<string> = [];
function getItems(): Array<Item> { ... }

// WRONG - Never do this
const users: User[] = [];
const ids: string[] = [];
function getItems(): Item[] { ... }
```

### Never Use `any`

Use proper types, `unknown`, or generic constraints:

```typescript
// WRONG
function processData(data: any): any { ... }
const result: any = fetchSomething();

// CORRECT
function processData<T>(data: T): ProcessedResult<T> { ... }
function processData(data: unknown): Result { ... }
```

If you truly need to bypass type checking (rare), use `unknown` with type guards:

```typescript
function handleUnknown(value: unknown): string {
  if (typeof value === 'string') {
    return value;
  }
  if (typeof value === 'object' && value !== null && 'toString' in value) {
    return String(value);
  }
  return 'unknown';
}
```

## Naming Conventions

### No Abbreviations

Use full words for clarity:

```typescript
// WRONG
const ctx = getContext();
const tx = buildTransaction();
const sig = await sendTx(tx);
const acct = getAccount();
const params = { amt: 100 };

// CORRECT
const context = getContext();
const transaction = buildTransaction();
const signature = await sendTransaction(transaction);
const account = getAccount();
const parameters = { amount: 100 };
```

**Common violations to avoid:**
| Wrong | Correct |
|-------|---------|
| `ctx` | `context` |
| `tx` | `transaction` |
| `sig` | `signature` |
| `acct`, `acc` | `account` |
| `amt` | `amount` |
| `msg` | `message` |
| `err`, `e` | `error` |
| `req`, `res` | `request`, `response` |
| `cb` | `callback` |
| `fn` | `function` or descriptive name |
| `idx`, `i` | `index` (or domain-specific like `userIndex`) |
| `val` | `value` |
| `num` | `number` or domain-specific |
| `str` | `string` or domain-specific |
| `obj` | descriptive name |
| `arr` | descriptive plural |
| `pubkey` | `publicKey` |

### Arrays Are Plurals, Items Are Singular

```typescript
// CORRECT
const users: Array<User> = [];
users.forEach((user) => {
  console.log(user.name);
});

const transactions: Array<Transaction> = [];
for (const transaction of transactions) {
  await processTransaction(transaction);
}

// WRONG
const userList: Array<User> = [];
userList.forEach((item) => { ... });

const txArray: Array<Transaction> = [];
for (const element of txArray) { ... }
```

### Functions Are Verby

Functions should describe actions:

```typescript
// CORRECT
function calculateTotal(items: Array<Item>): number { ... }
function fetchUserBalance(userId: string): Promise<number> { ... }
function buildTransaction(instructions: Array<Instruction>): Transaction { ... }
function validateSignature(signature: string): boolean { ... }

// WRONG
function total(items: Array<Item>): number { ... }
function userBalance(userId: string): Promise<number> { ... }
function transaction(instructions: Array<Instruction>): Transaction { ... }
```

### Match Names to Types

Never confuse transaction vs instruction vs signature:

```typescript
// WRONG - Deceptive naming
const transaction = buildInstruction(data); // Returns Instruction, not Transaction!
const signature = buildTransaction(instructions); // Returns Transaction, not signature!

// CORRECT - Names match types
const instruction = buildInstruction(data);
const transaction = buildTransaction(instructions);
const signature = await sendTransaction(transaction);
```

### Never Use `e` for Thrown Objects

```typescript
// WRONG
try {
  await riskyOperation();
} catch (e) {
  console.error(e);
}

// CORRECT
try {
  await riskyOperation();
} catch (thrownObject) {
  const error = ensureError(thrownObject);
  console.error(error.message);
}
```

## Async/Await

### Favor async/await Over .then()/.catch()

```typescript
// WRONG
function fetchData() {
  return fetch(url)
    .then((response) => response.json())
    .then((data) => processData(data))
    .catch((error) => handleError(error));
}

// CORRECT
async function fetchData() {
  try {
    const response = await fetch(url);
    const data = await response.json();
    return processData(data);
  } catch (thrownObject) {
    const error = ensureError(thrownObject);
    handleError(error);
  }
}
```

### Top-Level Await

`tsx` supports top-level await. No need for IIFEs:

```typescript
// WRONG
(async () => {
  const result = await initialize();
  console.log(result);
})();

// CORRECT
const result = await initialize();
console.log(result);
```

## Error Handling

### Thrown Object Handling

JavaScript allows throwing anything. Handle it properly:

```typescript
// Include this utility in your project
const ensureError = function (thrownObject: unknown): Error {
  if (thrownObject instanceof Error) {
    return thrownObject;
  }
  // In JS it's possible to throw *anything*. A sensible programmer
  // will only throw Errors but we must still check to satisfy
  // TypeScript (and flag any craziness)
  return new Error(`Non-Error thrown: ${String(thrownObject)}`);
};

// Usage
try {
  await riskyOperation();
} catch (thrownObject) {
  const error = ensureError(thrownObject);
  console.error('Operation failed:', error.message);
  throw error;
}
```

## Comments

### Use `//` for Most Comments

Comments should be above (not beside) the code:

```typescript
// CORRECT
// Calculate the user's total balance including pending transactions
const totalBalance = calculateTotalBalance(user);

// WRONG
const totalBalance = calculateTotalBalance(user); // Calculate total balance
```

### JSDoc/TSDoc Uses `/** */`

The only exception for `/* */` syntax:

```typescript
/**
 * Calculates the total balance for a user.
 * @param user - The user account to calculate balance for
 * @returns The total balance in lamports
 */
function calculateTotalBalance(user: User): bigint {
  ...
}
```

### Don't State the Obvious

Remove comments that just repeat variable names or obvious code:

```typescript
// WRONG - Comment adds nothing
// Get the user
const user = getUser();

// Set the amount
const amount = 100;

// CORRECT - No comment needed, code is self-documenting
const user = getUser();
const amount = 100;

// CORRECT - Comment adds useful context
// Retry with exponential backoff to handle rate limiting
const result = await retryWithBackoff(fetchData, { maxAttempts: 3 });
```

## tsconfig.json

Avoid using `tsconfig.json` unless necessary (tsx doesn't usually need one). If you do need one, state why at the top of the file:

```json
// tsconfig.json needed for: [specific reason, e.g., "path aliases" or "strict null checks in IDE"]
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "NodeNext",
    "strict": true,
    "noEmit": true
  }
}
```

## Unit Testing

### Use Node.js Built-in Test Library

```typescript
// CORRECT - Use node:test
import { before, describe, test } from 'node:test';
import assert from 'node:assert';

describe('UserService', () => {
  test('should create a user', async () => {
    const user = await createUser({ name: 'Alice' });
    assert.strictEqual(user.name, 'Alice');
  });
});

// WRONG - Don't use Jest/Mocha/Vitest
import { describe, it, expect } from 'vitest';
```

### Use `test` Not `it`

```typescript
// CORRECT
test('should return the correct balance', () => { ... });

// WRONG
it('should return the correct balance', () => { ... });
```

### Run Tests with tsx

```bash
# CORRECT
npx tsx --test tests/**/*.test.ts

# Or in package.json
{
  "scripts": {
    "test": "tsx --test tests/**/*.test.ts"
  }
}
```

## Import Organization

Order imports logically:

```typescript
// 1. Node.js built-ins
import { describe, test } from 'node:test';
import assert from 'node:assert';

// 2. External packages
import { Address, Signer } from '@solana/kit';
import { createTransfer } from '@solana-program/system';

// 3. Internal modules (absolute paths)
import { UserService } from '@/services/user';

// 4. Relative imports
import { helpers } from './helpers';
import type { Config } from './types';
```

## Code Quality Checklist

Before committing TypeScript code:

- [ ] All arrays use `Array<T>` syntax
- [ ] No `any` types anywhere
- [ ] No abbreviations in variable/function names
- [ ] Arrays are plurals, items are singular
- [ ] Functions have verby names
- [ ] Names match their types (instruction ≠ transaction)
- [ ] Using async/await, not .then()/.catch()
- [ ] Error handling uses ensureError pattern
- [ ] Comments use `//` (except JSDoc)
- [ ] No obvious/redundant comments
- [ ] Tests use `node:test` with `test()` not `it()`
- [ ] No unused imports or variables
- [ ] No console.log statements (except in test files or CLI tools)
