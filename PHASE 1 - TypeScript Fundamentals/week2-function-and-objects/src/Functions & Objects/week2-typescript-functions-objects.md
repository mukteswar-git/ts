# Week 2: TypeScript — Functions & Objects

> A complete, hands-on tutorial covering typed functions, object types, advanced type patterns, and type narrowing.

---

## Table of Contents

1. [Typed Function Parameters and Return Types](#1-typed-function-parameters-and-return-types)
2. [Optional and Default Parameters](#2-optional-and-default-parameters)
3. [Rest Parameters with Types](#3-rest-parameters-with-types)
4. [Function Overloads](#4-function-overloads)
5. [Object Types and Nested Objects](#5-object-types-and-nested-objects)
6. [Index Signatures](#6-index-signatures)
7. [Intersection Types (A & B)](#7-intersection-types-a--b)
8. [Literal Types](#8-literal-types)
9. [Type Narrowing](#9-type-narrowing)
10. [Discriminated Unions](#10-discriminated-unions)
11. [Putting It All Together](#11-putting-it-all-together)

---

## 1. Typed Function Parameters and Return Types

TypeScript lets you explicitly annotate both the **input** and **output** of every function. This catches bugs at compile time instead of runtime.

### Basic Syntax

```typescript
function add(a: number, b: number): number {
  return a + b;
}

const greet = (name: string): string => {
  return `Hello, ${name}!`;
};
```

### Why Bother?

Without types, the following JavaScript silently produces `NaN`:

```javascript
// JavaScript — no error!
function add(a, b) { return a + b; }
add("5", 10); // "510" (string concatenation!)
```

With TypeScript:

```typescript
function add(a: number, b: number): number { return a + b; }
add("5", 10); // ❌ Error: Argument of type 'string' is not assignable to parameter of type 'number'
```

### The `void` Return Type

Use `void` when a function doesn't return a meaningful value:

```typescript
function logMessage(msg: string): void {
  console.log(msg);
  // No return statement needed
}
```

### The `never` Return Type

Use `never` for functions that **never** return — they either throw or run forever:

```typescript
function throwError(message: string): never {
  throw new Error(message);
}

function infiniteLoop(): never {
  while (true) {}
}
```

### Typing Callback Functions

```typescript
// Inline callback type
function processItems(items: number[], callback: (item: number) => void): void {
  items.forEach(callback);
}

// Using a type alias for the callback
type Transformer = (input: string) => string;

function applyTransform(value: string, transform: Transformer): string {
  return transform(value);
}

applyTransform("hello", (s) => s.toUpperCase()); // "HELLO"
```

### Return Type Inference

TypeScript can often infer return types — but explicit annotations are recommended for public APIs and complex functions:

```typescript
// TypeScript infers: () => number
function getYear() {
  return new Date().getFullYear();
}

// Explicit is better for clarity and safety
function getYear(): number {
  return new Date().getFullYear();
}
```

---

## 2. Optional and Default Parameters

### Optional Parameters

Add `?` after a parameter name to make it optional. Optional parameters are typed as `T | undefined` internally.

```typescript
function buildUrl(base: string, path?: string): string {
  if (path) {
    return `${base}/${path}`;
  }
  return base;
}

buildUrl("https://example.com");           // ✅ "https://example.com"
buildUrl("https://example.com", "about");  // ✅ "https://example.com/about"
```

> **Rule:** Optional parameters must come **after** required parameters.

```typescript
// ❌ Error: Required parameter 'b' cannot follow optional parameter 'a'
function bad(a?: string, b: number): void {}

// ✅ Correct
function good(b: number, a?: string): void {}
```

### Default Parameters

Default parameters are **implicitly optional** and provide a fallback value:

```typescript
function createUser(name: string, role: string = "viewer"): object {
  return { name, role };
}

createUser("Alice");           // { name: "Alice", role: "viewer" }
createUser("Bob", "admin");    // { name: "Bob", role: "admin" }
```

TypeScript infers the type from the default value — no annotation needed:

```typescript
function repeat(text: string, times = 3): string {
  //                          ^^^^^^^^ TypeScript infers: times: number
  return text.repeat(times);
}
```

### Optional vs Default — When to Use Which

| Scenario | Use |
|---|---|
| Parameter may not be provided, and absence matters | `optional` (`?`) |
| Parameter may not be provided, but a sensible default exists | `default` (`= value`) |
| You need to distinguish between `undefined` and "not passed" | `optional` |

```typescript
// Optional: the absence means "don't include it"
function query(filter?: string): void {
  const sql = filter ? `WHERE ${filter}` : "";
  console.log(`SELECT * FROM table ${sql}`);
}

// Default: always needs a value, but has a common case
function paginate(page: number, pageSize: number = 20): void {
  console.log(`Page ${page}, ${pageSize} items`);
}
```

---

## 3. Rest Parameters with Types

Rest parameters collect **all remaining arguments** into a typed array.

### Basic Rest Parameters

```typescript
function sum(...numbers: number[]): number {
  return numbers.reduce((total, n) => total + n, 0);
}

sum(1, 2, 3);       // 6
sum(10, 20, 30, 40); // 100
sum();              // 0
```

### Rest Parameters with Mixed Types

Rest parameters must be the **last** parameter:

```typescript
function logWithPrefix(prefix: string, ...messages: string[]): void {
  messages.forEach(msg => console.log(`[${prefix}] ${msg}`));
}

logWithPrefix("INFO", "Server started", "Listening on port 3000");
// [INFO] Server started
// [INFO] Listening on port 3000
```

### Typed Tuple Rest Parameters

For heterogeneous rest parameters, use tuple types:

```typescript
type StringAndNumbers = [string, ...number[]];

function firstStringThenNumbers(...args: StringAndNumbers): void {
  const [label, ...values] = args;
  console.log(`${label}: ${values.join(", ")}`);
}

firstStringThenNumbers("Scores", 95, 87, 100); // Scores: 95, 87, 100
```

### Spreading into Typed Functions

```typescript
function add(a: number, b: number, c: number): number {
  return a + b + c;
}

const args: [number, number, number] = [1, 2, 3];
add(...args); // ✅ TypeScript knows the tuple matches the parameters
```

---

## 4. Function Overloads

Function overloads let you define **multiple call signatures** for a single function. This is useful when a function behaves differently depending on the argument types.

### The Pattern

1. Write **overload signatures** (no implementation body)
2. Write one **implementation signature** (must handle all overloads)

```typescript
// Overload signatures
function format(value: string): string;
function format(value: number): string;
function format(value: Date): string;

// Implementation signature (not callable directly)
function format(value: string | number | Date): string {
  if (typeof value === "string") return value.trim();
  if (typeof value === "number") return value.toFixed(2);
  return value.toISOString();
}

format("  hello  ");   // "hello"
format(3.14159);       // "3.14"
format(new Date());    // "2024-01-15T..."
```

### Overloads with Different Argument Counts

```typescript
function createElement(tag: string): HTMLElement;
function createElement(tag: string, id: string): HTMLElement;
function createElement(tag: string, id: string, className: string): HTMLElement;

function createElement(tag: string, id?: string, className?: string): HTMLElement {
  const el = document.createElement(tag);
  if (id) el.id = id;
  if (className) el.className = className;
  return el;
}
```

### Overloads with Return Type Changes

This is where overloads truly shine — the return type changes based on input:

```typescript
function parseInput(input: string): string[];
function parseInput(input: string[]): string;

function parseInput(input: string | string[]): string | string[] {
  if (typeof input === "string") {
    return input.split(",").map(s => s.trim());
  }
  return input.join(", ");
}

const arr = parseInput("a, b, c");   // Type: string[]
const str = parseInput(["a", "b"]);  // Type: string
```

> **Important:** The implementation signature is **not visible** to callers. Only the overload signatures are.

### When to Use Overloads

- When a function accepts different argument shapes and the return type varies
- When union types in a single signature would be too confusing
- When you need precise inference on return types based on inputs

---

## 5. Object Types and Nested Objects

### Inline Object Types

```typescript
function printCoord(point: { x: number; y: number }): void {
  console.log(`(${point.x}, ${point.y})`);
}

printCoord({ x: 3, y: 7 });
```

### Type Aliases

Use `type` to name and reuse object shapes:

```typescript
type Point = {
  x: number;
  y: number;
};

type User = {
  id: number;
  name: string;
  email: string;
};

function getInitials(user: User): string {
  return user.name.split(" ").map(n => n[0]).join(".");
}
```

### Interfaces

`interface` is the other way to define object shapes. Prefer `interface` for objects that may be extended:

```typescript
interface Product {
  id: string;
  name: string;
  price: number;
}

// Interfaces can be extended
interface DigitalProduct extends Product {
  downloadUrl: string;
  fileSize: number;
}
```

### Nested Objects

```typescript
type Address = {
  street: string;
  city: string;
  country: string;
  zip: string;
};

type Customer = {
  id: number;
  name: string;
  billing: Address;
  shipping: Address;
};

const customer: Customer = {
  id: 1,
  name: "Alice",
  billing: {
    street: "123 Main St",
    city: "Springfield",
    country: "US",
    zip: "12345",
  },
  shipping: {
    street: "456 Elm Ave",
    city: "Springfield",
    country: "US",
    zip: "12345",
  },
};
```

### Optional Object Properties

```typescript
type Config = {
  host: string;
  port: number;
  ssl?: boolean;       // optional
  timeout?: number;    // optional
};

const config: Config = { host: "localhost", port: 5432 };
// ssl and timeout can be omitted
```

### Readonly Properties

```typescript
type ImmutablePoint = {
  readonly x: number;
  readonly y: number;
};

const p: ImmutablePoint = { x: 5, y: 10 };
p.x = 20; // ❌ Error: Cannot assign to 'x' because it is a read-only property
```

---

## 6. Index Signatures

Index signatures describe **dictionary-like** objects where you don't know all the keys ahead of time.

### Basic Index Signature

```typescript
type StringMap = {
  [key: string]: string;
};

const headers: StringMap = {
  "Content-Type": "application/json",
  "Authorization": "Bearer abc123",
  "X-Custom-Header": "value",
};
```

### Number Index Signatures

```typescript
type NumberArray = {
  [index: number]: string;
};

// Arrays naturally satisfy this:
const fruits: NumberArray = ["apple", "banana", "cherry"];
console.log(fruits[0]); // "apple"
```

### Mixing Known and Unknown Keys

You can combine explicit properties with an index signature — but the explicit properties must match the index signature type:

```typescript
type Config = {
  debug: boolean;           // ❌ Error if index is [key: string]: string
};

// Fix: make the index signature compatible
type FlexibleConfig = {
  name: string;             // explicit known key
  version: string;          // explicit known key
  [key: string]: string;    // any other keys must also be strings
};
```

### Practical Example: Environment Variables

```typescript
type EnvVars = {
  NODE_ENV: "development" | "production" | "test";
  PORT: string;
  [key: string]: string; // Any other env vars
};

function getEnv(key: string, vars: EnvVars): string {
  return vars[key] ?? "undefined";
}
```

### Record Utility Type

`Record<K, V>` is a cleaner alternative to index signatures for uniform key-value mappings:

```typescript
// Equivalent to { [key: string]: number }
type ScoreMap = Record<string, number>;

const scores: ScoreMap = {
  alice: 95,
  bob: 87,
  carol: 100,
};

// Record with literal key union — even better!
type Day = "Mon" | "Tue" | "Wed" | "Thu" | "Fri";
type Schedule = Record<Day, string[]>;

const meetings: Schedule = {
  Mon: ["Standup"],
  Tue: ["Design Review"],
  Wed: [],
  Thu: ["1:1 with Manager"],
  Fri: ["Retrospective"],
};
```

---

## 7. Intersection Types (A & B)

Intersection types combine **multiple types into one**. The resulting type must satisfy **all** of the constituent types.

### Basic Intersection

```typescript
type HasName = { name: string };
type HasAge  = { age: number };

type Person = HasName & HasAge;

const alice: Person = { name: "Alice", age: 30 }; // ✅ Must have both
```

### Practical Use: Mixins and Roles

```typescript
type Timestamped = {
  createdAt: Date;
  updatedAt: Date;
};

type SoftDeletable = {
  deletedAt: Date | null;
};

type Post = {
  id: string;
  title: string;
  content: string;
};

// A database record has all three shapes
type PostRecord = Post & Timestamped & SoftDeletable;

const post: PostRecord = {
  id: "abc",
  title: "Hello World",
  content: "My first post",
  createdAt: new Date(),
  updatedAt: new Date(),
  deletedAt: null,
};
```

### Intersection vs Union

| Concept | Symbol | Meaning |
|---|---|---|
| Union | `A \| B` | Either A **or** B (or both) |
| Intersection | `A & B` | Both A **and** B simultaneously |

```typescript
type A = { foo: string };
type B = { bar: number };

type AorB  = A | B;        // Must have foo OR bar (or both)
type AandB = A & B;        // Must have BOTH foo AND bar

const val1: AorB  = { foo: "hi" };          // ✅
const val2: AorB  = { bar: 42 };            // ✅
const val3: AandB = { foo: "hi", bar: 42 }; // ✅
const val4: AandB = { foo: "hi" };          // ❌ Missing bar
```

### Intersection with Functions

```typescript
type Logger = {
  log(msg: string): void;
};

type Serializer = {
  serialize(data: unknown): string;
};

type Plugin = Logger & Serializer;

function usePlugin(plugin: Plugin): void {
  plugin.log("Starting...");
  const result = plugin.serialize({ key: "value" });
  plugin.log(`Result: ${result}`);
}
```

### When Intersections Conflict

If intersecting two types with the same property but different types, the property becomes `never`:

```typescript
type A = { x: string };
type B = { x: number };
type C = A & B;

// C.x is string & number = never
const c: C = { x: "hello" }; // ❌ 'string' is not assignable to 'never'
```

---

## 8. Literal Types

Literal types constrain a value to one **specific value**, not just a general type.

### String Literals

```typescript
type Direction = "north" | "south" | "east" | "west";

function move(direction: Direction): void {
  console.log(`Moving ${direction}`);
}

move("north");   // ✅
move("up");      // ❌ Error: '"up"' is not assignable to type 'Direction'
```

### Numeric Literals

```typescript
type DiceRoll = 1 | 2 | 3 | 4 | 5 | 6;

function roll(): DiceRoll {
  return (Math.floor(Math.random() * 6) + 1) as DiceRoll;
}
```

### Boolean Literals

```typescript
type AlwaysTrue = true;
type AlwaysFalse = false;

// Useful in conditional types and discriminated unions
function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${value}`);
}
```

### Template Literal Types

TypeScript supports template literal types for string pattern matching:

```typescript
type EventName = `on${string}`;      // onAny string
type ClickEvent = `on${"Click" | "DoubleClick"}`;  // "onClick" | "onDoubleClick"

type CSSProperty = `${"margin" | "padding"}-${"top" | "right" | "bottom" | "left"}`;
// "margin-top" | "margin-right" | "margin-bottom" | "margin-left" | "padding-..."
```

### Literal Widening

TypeScript "widens" inferred types unless you prevent it:

```typescript
let direction = "north";          // Type: string (widened!)
const direction2 = "north";       // Type: "north" (literal — const doesn't widen)

// Force literal type on a let variable
let direction3 = "north" as const; // Type: "north"

// Object properties are widened unless you use as const
const config = { mode: "dark" };             // mode: string
const config2 = { mode: "dark" } as const;  // mode: "dark"
```

---

## 9. Type Narrowing

TypeScript tracks possible types through control flow. **Narrowing** is the process of refining a broad type into a more specific one.

### `typeof` Narrowing

```typescript
function printValue(value: string | number): void {
  if (typeof value === "string") {
    // TypeScript knows: value is string here
    console.log(value.toUpperCase());
  } else {
    // TypeScript knows: value is number here
    console.log(value.toFixed(2));
  }
}
```

`typeof` returns: `"string"`, `"number"`, `"bigint"`, `"boolean"`, `"symbol"`, `"undefined"`, `"object"`, `"function"`

> **Gotcha:** `typeof null === "object"` — always check for `null` separately!

### `instanceof` Narrowing

```typescript
class Dog {
  bark() { return "Woof!"; }
}

class Cat {
  meow() { return "Meow!"; }
}

function makeSound(animal: Dog | Cat): string {
  if (animal instanceof Dog) {
    return animal.bark(); // TypeScript knows: animal is Dog
  }
  return animal.meow();   // TypeScript knows: animal is Cat
}
```

### `in` Narrowing

The `in` operator checks if a property exists on an object:

```typescript
type Admin = { role: "admin"; permissions: string[] };
type Guest = { role: "guest"; sessionId: string };

type User = Admin | Guest;

function describeUser(user: User): string {
  if ("permissions" in user) {
    // TypeScript knows: user is Admin
    return `Admin with ${user.permissions.length} permissions`;
  }
  // TypeScript knows: user is Guest
  return `Guest session: ${user.sessionId}`;
}
```

### Equality Narrowing

```typescript
function compare(a: string | number, b: string | boolean): void {
  if (a === b) {
    // Both must be string (the only type they share)
    console.log(a.toUpperCase()); // ✅ TypeScript knows a is string
  }
}
```

### Truthiness Narrowing

```typescript
function printName(name: string | null | undefined): void {
  if (name) {
    // name is string here (null and undefined are falsy)
    console.log(name.trim());
  } else {
    console.log("No name provided");
  }
}
```

### Custom Type Guards

Write a function that returns a **type predicate** (`value is Type`):

```typescript
type Fish = { swim: () => void };
type Bird = { fly: () => void };

function isFish(animal: Fish | Bird): animal is Fish {
  return "swim" in animal;
}

function move(animal: Fish | Bird): void {
  if (isFish(animal)) {
    animal.swim(); // ✅ TypeScript knows: animal is Fish
  } else {
    animal.fly();  // ✅ TypeScript knows: animal is Bird
  }
}
```

### Assertion Functions

```typescript
function assertIsString(value: unknown): asserts value is string {
  if (typeof value !== "string") {
    throw new Error(`Expected string, got ${typeof value}`);
  }
}

function processInput(input: unknown): void {
  assertIsString(input);
  // TypeScript now knows input is string
  console.log(input.toUpperCase());
}
```

---

## 10. Discriminated Unions

A discriminated union is a union of types that each share a **common literal property** (the discriminant). TypeScript uses this property to narrow the type precisely.

### The Pattern

```typescript
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number }
  | { kind: "rectangle"; width: number; height: number };
```

Here `kind` is the **discriminant** — a literal type that uniquely identifies each variant.

### Switching on the Discriminant

```typescript
function getArea(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      // TypeScript knows: shape is { kind: "circle"; radius: number }
      return Math.PI * shape.radius ** 2;

    case "square":
      // TypeScript knows: shape is { kind: "square"; side: number }
      return shape.side ** 2;

    case "rectangle":
      // TypeScript knows: shape is { kind: "rectangle"; width: number; height: number }
      return shape.width * shape.height;
  }
}
```

### Exhaustiveness Checking

Add a `default` case with `never` to get a compile error if you add a new variant but forget to handle it:

```typescript
function getArea(shape: Shape): number {
  switch (shape.kind) {
    case "circle":    return Math.PI * shape.radius ** 2;
    case "square":    return shape.side ** 2;
    case "rectangle": return shape.width * shape.height;
    default:
      const _exhaustive: never = shape; // ❌ Error if a case is missing!
      throw new Error(`Unhandled shape: ${_exhaustive}`);
  }
}
```

If you add `| { kind: "triangle"; base: number; height: number }` to `Shape` without adding a `case "triangle"`, TypeScript will flag the `never` assignment as an error.

### Real-World Example: API Responses

```typescript
type ApiResponse<T> =
  | { status: "success"; data: T }
  | { status: "error"; error: string; code: number }
  | { status: "loading" };

function handleResponse<T>(response: ApiResponse<T>): string {
  switch (response.status) {
    case "success":
      return `Got data: ${JSON.stringify(response.data)}`;
    case "error":
      return `Error ${response.code}: ${response.error}`;
    case "loading":
      return "Loading...";
  }
}
```

### Example: Redux-Style Action Types

```typescript
type Action =
  | { type: "INCREMENT"; amount: number }
  | { type: "DECREMENT"; amount: number }
  | { type: "RESET" }
  | { type: "SET_VALUE"; value: number };

function reducer(state: number, action: Action): number {
  switch (action.type) {
    case "INCREMENT": return state + action.amount;
    case "DECREMENT": return state - action.amount;
    case "RESET":     return 0;
    case "SET_VALUE": return action.value;
  }
}
```

---

## 11. Putting It All Together

Here's a complete, realistic example that combines all the concepts from this week:

```typescript
// --- Literal & Discriminated Union Types ---
type NotificationChannel = "email" | "sms" | "push";

type Notification =
  | { channel: "email"; to: string; subject: string; body: string }
  | { channel: "sms"; to: string; message: string }
  | { channel: "push"; deviceId: string; title: string; payload?: object };

// --- Object Type with Index Signature ---
type NotificationConfig = {
  defaultChannel: NotificationChannel;
  maxRetries: number;
  [key: string]: unknown; // allow provider-specific options
};

// --- Intersection Type ---
type Timestamped = { sentAt: Date };
type NotificationRecord = Notification & Timestamped;

// --- Function Overloads ---
function formatNotification(n: { channel: "email" } & Notification): string;
function formatNotification(n: { channel: "sms" } & Notification): string;
function formatNotification(n: Notification): string {
  switch (n.channel) {
    case "email":  return `Email → ${n.to}: ${n.subject}`;
    case "sms":    return `SMS → ${n.to}: ${n.message}`;
    case "push":   return `Push → ${n.deviceId}: ${n.title}`;
  }
}

// --- Typed Function with Optional & Default Params ---
function sendNotification(
  notification: Notification,
  config: NotificationConfig,
  retries: number = 0,
  onSuccess?: (record: NotificationRecord) => void
): NotificationRecord {
  // --- Type Narrowing with discriminant ---
  const record: NotificationRecord = {
    ...notification,
    sentAt: new Date(),
  };

  console.log(`[Attempt ${retries + 1}] ${formatNotification(notification)}`);
  onSuccess?.(record);
  return record;
}

// --- Rest Parameters ---
function broadcastNotification(
  config: NotificationConfig,
  ...notifications: Notification[]
): NotificationRecord[] {
  return notifications.map(n => sendNotification(n, config));
}

// --- Usage ---
const config: NotificationConfig = {
  defaultChannel: "email",
  maxRetries: 3,
  apiKey: "secret-key-123",  // allowed by index signature
};

const records = broadcastNotification(
  config,
  { channel: "email", to: "alice@example.com", subject: "Welcome!", body: "Hello Alice" },
  { channel: "sms", to: "+1234567890", message: "Your code: 1234" },
  { channel: "push", deviceId: "device-abc", title: "New message", payload: { count: 1 } }
);

console.log(`Sent ${records.length} notifications`);
```

---

## Quick Reference Card

| Feature | Syntax | Example |
|---|---|---|
| Typed params | `param: Type` | `(x: number)` |
| Return type | `(): Type` | `(): string` |
| Optional param | `param?: Type` | `(name?: string)` |
| Default param | `param = value` | `(role = "user")` |
| Rest params | `...param: Type[]` | `(...ids: number[])` |
| Overloads | Multiple signatures | `f(x: string): string;` |
| Object type | `{ key: Type }` | `{ name: string }` |
| Index signature | `[key: K]: V` | `[key: string]: number` |
| Intersection | `A & B` | `User & Admin` |
| Literal type | `"value"` | `"left" \| "right"` |
| `typeof` guard | `typeof x === "type"` | `typeof x === "string"` |
| `instanceof` guard | `x instanceof Class` | `x instanceof Date` |
| `in` guard | `"key" in obj` | `"swim" in animal` |
| Type predicate | `x is Type` | `(x): x is Fish` |
| Discriminated union | Common literal property | `{ kind: "circle" }` |

---

## Exercises

1. **Function Types** — Write a `calculate` function with overloads: when called with two `number` arguments it returns a `number`, when called with two `string` arguments it returns a `string` (concatenation).

2. **Object Types** — Model a `Library` with a collection of `Book` objects (each with `title`, `author`, `year`, and optional `isbn`), plus an index signature for arbitrary metadata.

3. **Discriminated Union** — Create a `Result<T>` type with `{ ok: true; value: T }` and `{ ok: false; error: string }` variants. Write a `unwrap` function that returns the value or throws the error.

4. **Type Narrowing** — Write a function `flatten` that accepts `string | string[] | null` and always returns a `string[]`. Use type narrowing to handle each case.

5. **Intersection Types** — Define `Paginated<T>` as a type that intersects `{ items: T[]; total: number }` with `{ page: number; pageSize: number }`. Write a function that builds a `Paginated<User>` from raw data.

---

*Next Week: Generics, Utility Types, and Type Manipulation*
