# Week 3: Generics & Utility Types in TypeScript

> **Prerequisites:** Basic TypeScript knowledge (types, interfaces, type aliases), familiarity with ES6+ JavaScript.

---

## Table of Contents

1. [Generics Basics](#1-generics-basics)
2. [Generic Functions](#2-generic-functions)
3. [Generic Interfaces](#3-generic-interfaces)
4. [Generic Constraints with `extends`](#4-generic-constraints-with-extends)
5. [Built-in Utility Types: Partial, Required, Readonly](#5-built-in-utility-types-partial-required-readonly)
6. [Pick, Omit, Record, Exclude, Extract](#6-pick-omit-record-exclude-extract)
7. [ReturnType, Parameters, Awaited](#7-returntype-parameters-awaited)
8. [Nullable Types and `strictNullChecks`](#8-nullable-types-and-strictnullchecks)
9. [Non-null Assertion Operator (`!`)](#9-non-null-assertion-operator-)
10. [Type Assertions (`as`)](#10-type-assertions-as)
11. [Const Assertions (`as const`)](#11-const-assertions-as-const)
12. [Exercises](#12-exercises)

---

## 1. Generics Basics

Generics allow you to write **reusable, type-safe code** that works with multiple types without sacrificing type information. Think of a generic as a "type variable" — a placeholder that gets filled in when the code is used.

### The Problem Generics Solve

Without generics, you face a tradeoff: use `any` and lose type safety, or duplicate code for each type.

```typescript
// ❌ Using `any` — loses type safety
function identity(value: any): any {
  return value;
}
const result = identity(42); // type is `any`, not `number`

// ❌ Duplicating code for each type
function identityNumber(value: number): number { return value; }
function identityString(value: string): string { return value; }

// ✅ Generics — reusable AND type-safe
function identity<T>(value: T): T {
  return value;
}
const num = identity(42);       // type: number
const str = identity("hello");  // type: string
```

### Generic Type Parameter Naming Conventions

| Name | Common Usage |
|------|-------------|
| `T`  | General "Type" |
| `K`  | Key (especially in objects) |
| `V`  | Value |
| `E`  | Element (in arrays/collections) |
| `R`  | Return type |

These are conventions, not rules — you can use any valid identifier.

---

## 2. Generic Functions

A generic function declares one or more **type parameters** in angle brackets `<T>` before the parameter list.

### Basic Generic Function

```typescript
function firstElement<T>(arr: T[]): T | undefined {
  return arr[0];
}

const first = firstElement([1, 2, 3]);       // type: number | undefined
const name  = firstElement(["Alice", "Bob"]); // type: string | undefined
const empty = firstElement([]);               // type: undefined
```

### Multiple Type Parameters

```typescript
function pair<A, B>(first: A, second: B): [A, B] {
  return [first, second];
}

const p = pair("age", 30); // type: [string, number]
```

### Generic Arrow Functions

```typescript
// In a .ts file
const wrap = <T>(value: T): { value: T } => ({ value });

// In a .tsx file (JSX), add a trailing comma to avoid ambiguity with JSX tags
const wrap = <T,>(value: T): { value: T } => ({ value });
```

### Inferring Type Arguments

TypeScript can often **infer** the generic type from the arguments — you don't always need to specify it explicitly:

```typescript
function merge<T, U>(obj1: T, obj2: U): T & U {
  return { ...obj1, ...obj2 };
}

// Type inferred from arguments — no need to write merge<{name:string},{age:number}>
const person = merge({ name: "Alice" }, { age: 30 });
// person: { name: string } & { age: number }

// Explicit type argument when inference isn't possible
const empty = merge<{}, {}>({}, {});
```

---

## 3. Generic Interfaces

Interfaces can also be generic, making them reusable for different data shapes.

### Basic Generic Interface

```typescript
interface Box<T> {
  value: T;
  label: string;
}

const numberBox: Box<number> = { value: 42, label: "Answer" };
const stringBox: Box<string> = { value: "hello", label: "Greeting" };
```

### Generic Interface for Data Structures

```typescript
interface Stack<T> {
  push(item: T): void;
  pop(): T | undefined;
  peek(): T | undefined;
  isEmpty(): boolean;
  size(): number;
}

class ArrayStack<T> implements Stack<T> {
  private items: T[] = [];

  push(item: T): void {
    this.items.push(item);
  }

  pop(): T | undefined {
    return this.items.pop();
  }

  peek(): T | undefined {
    return this.items[this.items.length - 1];
  }

  isEmpty(): boolean {
    return this.items.length === 0;
  }

  size(): number {
    return this.items.length;
  }
}

const numStack = new ArrayStack<number>();
numStack.push(1);
numStack.push(2);
console.log(numStack.pop()); // 2
```

### Generic API Response Interface

A common real-world pattern:

```typescript
interface ApiResponse<T> {
  data: T;
  status: number;
  message: string;
  timestamp: Date;
}

interface User {
  id: number;
  name: string;
  email: string;
}

interface Post {
  id: number;
  title: string;
  body: string;
}

// Reuse the same ApiResponse shape for different data types
async function fetchUser(id: number): Promise<ApiResponse<User>> {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
}

async function fetchPost(id: number): Promise<ApiResponse<Post>> {
  const res = await fetch(`/api/posts/${id}`);
  return res.json();
}
```

### Generic Classes

```typescript
class Repository<T extends { id: number }> {
  private store: Map<number, T> = new Map();

  save(entity: T): T {
    this.store.set(entity.id, entity);
    return entity;
  }

  findById(id: number): T | undefined {
    return this.store.get(id);
  }

  findAll(): T[] {
    return Array.from(this.store.values());
  }

  delete(id: number): boolean {
    return this.store.delete(id);
  }
}

const userRepo = new Repository<User>();
userRepo.save({ id: 1, name: "Alice", email: "alice@example.com" });
const alice = userRepo.findById(1); // type: User | undefined
```

---

## 4. Generic Constraints with `extends`

Sometimes you need to restrict what types a generic parameter can be. Use `extends` to add **constraints**.

### Basic Constraint

```typescript
// Without constraint — TypeScript doesn't know T has .length
function printLength<T>(item: T): void {
  console.log(item.length); // ❌ Error: Property 'length' does not exist on type 'T'
}

// With constraint — T must have a 'length' property
interface HasLength {
  length: number;
}

function printLength<T extends HasLength>(item: T): void {
  console.log(item.length); // ✅ Safe — T is guaranteed to have .length
}

printLength("hello");        // 5
printLength([1, 2, 3]);      // 3
printLength({ length: 10 }); // 10
// printLength(42);          // ❌ Error: number doesn't have 'length'
```

### Constraining to Object Keys with `keyof`

A very common pattern: constraining a type parameter to be a key of another type.

```typescript
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { id: 1, name: "Alice", age: 30 };

const name = getProperty(user, "name"); // type: string ✅
const age  = getProperty(user, "age");  // type: number ✅
// getProperty(user, "email");          // ❌ Error: not a key of user
```

### Multiple Constraints

```typescript
interface Serializable {
  serialize(): string;
}

interface Loggable {
  log(): void;
}

// T must satisfy BOTH interfaces
function process<T extends Serializable & Loggable>(item: T): string {
  item.log();
  return item.serialize();
}
```

### Default Type Parameters

```typescript
interface PaginatedResult<T, Meta = Record<string, unknown>> {
  items: T[];
  total: number;
  page: number;
  meta: Meta;
}

// Uses default type for Meta
const result: PaginatedResult<User> = {
  items: [],
  total: 0,
  page: 1,
  meta: {},
};
```

---

## 5. Built-in Utility Types: Partial, Required, Readonly

TypeScript ships with a collection of **utility types** — generic types built into the language that transform other types.

### `Partial<T>`

Makes **all properties optional**. Useful for update/patch operations.

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
}

type PartialUser = Partial<User>;
// Equivalent to:
// {
//   id?: number;
//   name?: string;
//   email?: string;
//   age?: number;
// }

function updateUser(id: number, updates: Partial<User>): User {
  const user = findUserById(id);
  return { ...user, ...updates };
}

// ✅ Only provide the fields you want to change
updateUser(1, { name: "Bob" });
updateUser(1, { email: "bob@example.com", age: 25 });
```

### `Required<T>`

Makes **all properties required** (removes optionality). The opposite of `Partial`.

```typescript
interface Config {
  host?: string;
  port?: number;
  debug?: boolean;
}

type StrictConfig = Required<Config>;
// Equivalent to:
// {
//   host: string;
//   port: number;
//   debug: boolean;
// }

// Useful when you've applied defaults and need to guarantee all values exist
function applyDefaults(config: Config): Required<Config> {
  return {
    host: config.host ?? "localhost",
    port: config.port ?? 3000,
    debug: config.debug ?? false,
  };
}
```

### `Readonly<T>`

Makes **all properties read-only** — they cannot be reassigned after creation.

```typescript
interface Point {
  x: number;
  y: number;
}

const origin: Readonly<Point> = { x: 0, y: 0 };
// origin.x = 1; // ❌ Error: Cannot assign to 'x' because it is a read-only property

// Useful for immutable state, constants, and function parameters you don't want mutated
function translate(point: Readonly<Point>, dx: number, dy: number): Point {
  // point.x += dx; // ❌ Can't mutate the input
  return { x: point.x + dx, y: point.y + dy }; // ✅ Return a new point
}
```

> **Note:** `Readonly<T>` is **shallow** — nested objects remain mutable. For deep immutability, use `as const` (covered later) or a library like `immer`.

---

## 6. Pick, Omit, Record, Exclude, Extract

### `Pick<T, K>`

Creates a new type with only the **specified properties** from `T`.

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
  createdAt: Date;
}

// Pick only the fields safe to send to the client
type PublicUser = Pick<User, "id" | "name" | "email">;
// { id: number; name: string; email: string; }

// Use it in a function
function sanitizeUser(user: User): PublicUser {
  return { id: user.id, name: user.name, email: user.email };
}
```

### `Omit<T, K>`

Creates a new type **excluding** the specified properties from `T`. The inverse of `Pick`.

```typescript
// Omit sensitive fields
type SafeUser = Omit<User, "password">;
// { id: number; name: string; email: string; createdAt: Date; }

// When creating a new record, you don't have an id yet
type CreateUserInput = Omit<User, "id" | "createdAt">;
// { name: string; email: string; password: string; }

function createUser(input: CreateUserInput): User {
  return {
    ...input,
    id: generateId(),
    createdAt: new Date(),
  };
}
```

### `Record<K, V>`

Creates an object type with **keys of type `K`** and **values of type `V`**.

```typescript
// Simple lookup map
type ColorMap = Record<string, string>;
const colors: ColorMap = {
  primary: "#007bff",
  secondary: "#6c757d",
};

// Typed role permissions
type Role = "admin" | "editor" | "viewer";
type Permission = "read" | "write" | "delete";

const rolePermissions: Record<Role, Permission[]> = {
  admin:  ["read", "write", "delete"],
  editor: ["read", "write"],
  viewer: ["read"],
};

// Useful for grouping data
interface Product { name: string; price: number; }
type Category = "electronics" | "clothing" | "food";

const inventory: Record<Category, Product[]> = {
  electronics: [{ name: "Laptop", price: 999 }],
  clothing:    [{ name: "T-Shirt", price: 19 }],
  food:        [{ name: "Coffee", price: 12 }],
};
```

### `Exclude<T, U>`

From a union type `T`, **removes** all members that are assignable to `U`.

```typescript
type Status = "active" | "inactive" | "pending" | "banned";

type ActiveStatus = Exclude<Status, "banned">;
// "active" | "inactive" | "pending"

type WorkingStatus = Exclude<Status, "inactive" | "banned">;
// "active" | "pending"

// Works with any union, not just string literals
type NonNullable<T> = Exclude<T, null | undefined>; // This is actually a built-in!
```

### `Extract<T, U>`

From a union type `T`, **keeps only** the members that are assignable to `U`. The inverse of `Exclude`.

```typescript
type AllEvents = "click" | "focus" | "blur" | "change" | "submit" | "keydown";

// Keep only keyboard-related events
type KeyboardEvents = Extract<AllEvents, "keydown" | "keyup" | "keypress">;
// "keydown"  (only the ones present in AllEvents)

// A more powerful use case — extract types from a union
type Shape =
  | { kind: "circle";    radius: number }
  | { kind: "rectangle"; width: number; height: number }
  | { kind: "triangle";  base: number; height: number };

type CircleShape = Extract<Shape, { kind: "circle" }>;
// { kind: "circle"; radius: number }
```

---

## 7. ReturnType, Parameters, Awaited

These utility types let you **introspect function signatures** and extract type information from them.

### `ReturnType<T>`

Extracts the **return type** of a function type `T`.

```typescript
function greet(name: string): string {
  return `Hello, ${name}!`;
}

type Greeting = ReturnType<typeof greet>;
// string

function createUser(name: string, age: number) {
  return { id: Math.random(), name, age, createdAt: new Date() };
}

type CreatedUser = ReturnType<typeof createUser>;
// { id: number; name: string; age: number; createdAt: Date; }

// Useful when you can't easily name the return type
function fetchDashboardData() {
  return {
    users: [] as User[],
    posts: [] as Post[],
    stats: { total: 0, active: 0 },
  };
}

type DashboardData = ReturnType<typeof fetchDashboardData>;
// { users: User[]; posts: Post[]; stats: { total: number; active: number; } }
```

### `Parameters<T>`

Extracts the **parameter types** of a function as a **tuple**.

```typescript
function createProduct(name: string, price: number, category: string): void {
  // ...
}

type ProductParams = Parameters<typeof createProduct>;
// [name: string, price: number, category: string]

// Access individual parameter types
type NameParam     = Parameters<typeof createProduct>[0]; // string
type PriceParam    = Parameters<typeof createProduct>[1]; // number
type CategoryParam = Parameters<typeof createProduct>[2]; // string

// Practical: create a wrapper that logs calls
function withLogging<T extends (...args: any[]) => any>(
  fn: T
): (...args: Parameters<T>) => ReturnType<T> {
  return (...args) => {
    console.log("Calling with:", args);
    const result = fn(...args);
    console.log("Result:", result);
    return result;
  };
}

const loggedGreet = withLogging(greet);
loggedGreet("Alice"); // Logs the call and result, type is fully preserved
```

### `Awaited<T>`

Unwraps the resolved type of a **Promise** (recursively).

```typescript
// Without Awaited
type P1 = Promise<string>;           // Promise<string>
type P2 = Promise<Promise<number>>;  // Promise<Promise<number>>

// With Awaited
type R1 = Awaited<Promise<string>>;           // string
type R2 = Awaited<Promise<Promise<number>>>;  // number

// Practical: get the resolved type of an async function
async function fetchUsers(): Promise<User[]> {
  const res = await fetch("/api/users");
  return res.json();
}

type FetchedUsers = Awaited<ReturnType<typeof fetchUsers>>;
// User[]  — not Promise<User[]>

// Very useful when working with Promise.all
const results = await Promise.all([
  fetchUsers(),
  fetchPost(1),
]);

type AllResults = Awaited<typeof results>;
// [User[], Post]  — the resolved tuple type
```

---

## 8. Nullable Types and `strictNullChecks`

### What are Nullable Types?

In TypeScript, **`null`** and **`undefined`** are their own types. Whether they can be assigned to other types depends on the `strictNullChecks` compiler option.

### `strictNullChecks: false` (default in older configs)

```typescript
// Without strictNullChecks, null/undefined are assignable to anything
let name: string = null;      // ✅ No error (but dangerous!)
let age: number = undefined;  // ✅ No error
```

### `strictNullChecks: true` (recommended)

With strict null checks enabled (default in modern `tsconfig.json`), you must **explicitly include** `null` or `undefined` in the type:

```typescript
// tsconfig.json: { "strict": true } or { "strictNullChecks": true }

let name: string = null;      // ❌ Error: Type 'null' is not assignable to type 'string'
let name: string | null = null; // ✅ Explicit

// Optional properties are implicitly T | undefined
interface User {
  id: number;
  nickname?: string; // string | undefined
}
```

### Working with Nullable Types

```typescript
function getUserName(user: User | null): string {
  // ❌ Error — user might be null
  // return user.name;

  // ✅ Option 1: Check first
  if (user === null) {
    return "Anonymous";
  }
  return user.name; // TypeScript knows user is not null here

  // ✅ Option 2: Optional chaining + nullish coalescing
  return user?.name ?? "Anonymous";
}

// Narrowing with typeof
function double(value: number | null | undefined): number {
  if (value == null) return 0; // covers both null and undefined
  return value * 2;
}
```

### `NonNullable<T>`

A built-in utility type that removes `null` and `undefined` from a union:

```typescript
type MaybeString = string | null | undefined;
type DefiniteString = NonNullable<MaybeString>; // string

type MaybeUser = User | null | undefined;
type DefiniteUser = NonNullable<MaybeUser>; // User
```

---

## 9. Non-null Assertion Operator (`!`)

The non-null assertion operator `!` tells TypeScript: **"I guarantee this value is not null or undefined at runtime."** TypeScript will trust you and remove `null | undefined` from the type.

### Syntax

```typescript
value!  // Assert that 'value' is not null or undefined
```

### Basic Usage

```typescript
function processName(name: string | null): string {
  // TypeScript doesn't know name is safe here
  return name!.toUpperCase(); // Assert it's not null
}
```

### Common Use Cases

```typescript
// DOM elements — getElementById returns HTMLElement | null
const button = document.getElementById("submit-btn");
// button.addEventListener(...) // ❌ Error: button might be null

const button = document.getElementById("submit-btn")!;
// button.addEventListener(...) // ✅ We've asserted it exists

// React refs
const inputRef = useRef<HTMLInputElement>(null);

function focusInput() {
  // inputRef.current might be null before mounting
  inputRef.current!.focus(); // Assert it's mounted
}

// After a type guard has already been applied
interface Response {
  data?: string;
  error?: string;
}

function handleSuccess(res: Response) {
  if (res.data !== undefined) {
    // Inside here, TypeScript knows data is string
    console.log(res.data.toUpperCase()); // ✅ No assertion needed
  }
}
```

### ⚠️ When NOT to Use `!`

The `!` operator **bypasses TypeScript's safety checks**. If you're wrong about the value being non-null, you'll get a runtime `TypeError`.

```typescript
// ❌ Dangerous — what if the element doesn't exist?
const el = document.getElementById("might-not-exist")!;
el.textContent = "Hello"; // Runtime error if element is missing

// ✅ Better — check first
const el = document.getElementById("might-not-exist");
if (el) {
  el.textContent = "Hello";
}
```

> **Rule of thumb:** Prefer type narrowing (if checks, optional chaining) over `!`. Reserve `!` for cases where you have context TypeScript can't infer (e.g., DOM manipulation after the DOM has loaded, or after external setup).

---

## 10. Type Assertions (`as`)

Type assertions let you **override TypeScript's inferred type** for a value. You're telling the compiler: "Trust me, I know better than you about this type."

### Syntax

```typescript
value as TargetType
// or (older syntax, avoid in .tsx files):
<TargetType>value
```

### Basic Usage

```typescript
// TypeScript infers a broad type
const input = document.getElementById("username"); // HTMLElement | null

// Assert to a more specific type
const inputElement = document.getElementById("username") as HTMLInputElement;
const value = inputElement.value; // ✅ .value exists on HTMLInputElement

// Asserting from a union
function processInput(val: string | number) {
  const str = val as string;      // Assert it's a string
  console.log(str.toUpperCase()); // No error — but risky if val is actually a number!
}
```

### Asserting API Responses

```typescript
async function fetchUser(id: number): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  const data = await res.json(); // type: any

  return data as User; // Assert that the shape matches User
}
```

### Narrowing with `as`

```typescript
// When you know the shape of data coming from outside TypeScript
const config = JSON.parse(process.env.APP_CONFIG!) as AppConfig;

// In event handlers
document.addEventListener("click", (e) => {
  const target = e.target as HTMLButtonElement;
  console.log(target.textContent);
});
```

### Double Assertion (Escape Hatch)

TypeScript won't let you assert to a completely unrelated type. In rare cases, you can go through `unknown` first:

```typescript
const x = "hello" as number; // ❌ Error: string and number have no overlap

// Double assertion — only use this when you truly know what you're doing
const x = "hello" as unknown as number; // ✅ Compiles, but very dangerous
```

### ⚠️ Assertions vs. Type Guards

| Feature | Type Assertion (`as`) | Type Guard (if check) |
|---|---|---|
| Safety | Bypasses checks | Checked at runtime |
| When to use | Trusted external data | Uncertain runtime values |
| Error risk | Runtime errors possible | Compile-time safety |

```typescript
// ❌ Assertion — no runtime safety
function getLength(val: string | number): number {
  return (val as string).length; // Crashes if val is a number!
}

// ✅ Type guard — safe
function getLength(val: string | number): number {
  if (typeof val === "string") {
    return val.length;
  }
  return val.toString().length;
}
```

---

## 11. Const Assertions (`as const`)

`as const` is a special assertion that tells TypeScript to infer the **most specific, immutable type** for a value — treating it as if it were a literal constant.

### The Problem

```typescript
const colors = ["red", "green", "blue"];
// TypeScript infers: string[]  — mutable, broad type

const direction = { x: 1, y: -1 };
// TypeScript infers: { x: number; y: number }  — mutable
```

### With `as const`

```typescript
const colors = ["red", "green", "blue"] as const;
// TypeScript infers: readonly ["red", "green", "blue"]  — tuple of literals!

const direction = { x: 1, y: -1 } as const;
// TypeScript infers: { readonly x: 1; readonly y: -1 }  — literal types!
```

### Practical Examples

**Creating a union type from an array:**

```typescript
const STATUSES = ["active", "inactive", "pending", "banned"] as const;

// Extract the union type automatically
type Status = typeof STATUSES[number];
// "active" | "inactive" | "pending" | "banned"

// Now Status is always in sync with STATUSES — no duplication!
function setStatus(status: Status) { /* ... */ }
setStatus("active");  // ✅
// setStatus("deleted"); // ❌ Error
```

**Typed constants / enums alternative:**

```typescript
const Direction = {
  Up:    "UP",
  Down:  "DOWN",
  Left:  "LEFT",
  Right: "RIGHT",
} as const;

type Direction = typeof Direction[keyof typeof Direction];
// "UP" | "DOWN" | "LEFT" | "RIGHT"

function move(dir: Direction): void { /* ... */ }
move(Direction.Up);  // ✅
// move("up");        // ❌ Error — wrong case
```

**Route/config objects:**

```typescript
const ROUTES = {
  home:    "/",
  about:   "/about",
  users:   "/users",
  profile: "/users/:id",
} as const;

type AppRoute = typeof ROUTES[keyof typeof ROUTES];
// "/" | "/about" | "/users" | "/users/:id"
```

**Tuples:**

```typescript
// Without as const
function getMinMax(nums: number[]): [number, number] {
  return [Math.min(...nums), Math.max(...nums)];
}

// With as const — literal tuple types preserved
const RGB_WHITE = [255, 255, 255] as const;
// type: readonly [255, 255, 255]

function processColor(r: number, g: number, b: number) { /* ... */ }
processColor(...RGB_WHITE); // ✅ TypeScript knows the exact values
```

### `as const` vs `Object.freeze()`

| Feature | `as const` | `Object.freeze()` |
|---|---|---|
| Enforced | Compile time only | Runtime |
| Deep? | Yes (all nested) | No (shallow) |
| Effect on type | Narrows to literals | No type change |

---

## 12. Exercises

### Exercise 1: Generic `zip` Function
Write a generic function `zip<A, B>(arr1: A[], arr2: B[]): [A, B][]` that combines two arrays into an array of tuples.

```typescript
// Expected output:
zip([1, 2, 3], ["a", "b", "c"]);
// [[1, "a"], [2, "b"], [3, "c"]]
```

<details>
<summary>Solution</summary>

```typescript
function zip<A, B>(arr1: A[], arr2: B[]): [A, B][] {
  const length = Math.min(arr1.length, arr2.length);
  return Array.from({ length }, (_, i) => [arr1[i], arr2[i]]);
}

const zipped = zip([1, 2, 3], ["a", "b", "c"]);
// type: [number, string][]
```

</details>

---

### Exercise 2: Deep Readonly

Using the built-in utility types, create a `DeepReadonly<T>` type that makes an object and all its nested properties readonly.

```typescript
interface Config {
  server: {
    host: string;
    port: number;
  };
  database: {
    url: string;
  };
}
```

<details>
<summary>Solution</summary>

```typescript
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};

type ReadonlyConfig = DeepReadonly<Config>;
// Now server.host, server.port, database.url are all readonly
```

</details>

---

### Exercise 3: Filter by Property

Write a generic function that filters an array of objects by a specific key and value.

```typescript
filterBy(users, "role", "admin"); // Returns only admin users
```

<details>
<summary>Solution</summary>

```typescript
function filterBy<T, K extends keyof T>(
  arr: T[],
  key: K,
  value: T[K]
): T[] {
  return arr.filter(item => item[key] === value);
}

const users = [
  { id: 1, name: "Alice", role: "admin" },
  { id: 2, name: "Bob",   role: "user" },
  { id: 3, name: "Carol", role: "admin" },
];

const admins = filterBy(users, "role", "admin");
// [{ id: 1, ... }, { id: 3, ... }]
```

</details>

---

### Exercise 4: `as const` Enum

Create a `COLORS` constant using `as const` and derive a `Color` type from it. Write a function that accepts only valid colors.

<details>
<summary>Solution</summary>

```typescript
const COLORS = {
  Red:   "#FF0000",
  Green: "#00FF00",
  Blue:  "#0000FF",
  White: "#FFFFFF",
  Black: "#000000",
} as const;

type Color = typeof COLORS[keyof typeof COLORS];
// "#FF0000" | "#00FF00" | "#0000FF" | "#FFFFFF" | "#000000"

function applyColor(element: HTMLElement, color: Color): void {
  element.style.backgroundColor = color;
}

applyColor(div, COLORS.Red);   // ✅
// applyColor(div, "#123456"); // ❌ Not a valid COLORS value
```

</details>

---

## Summary

| Concept | Key Point |
|---|---|
| **Generics** | Type variables for reusable, type-safe code |
| **`extends` constraint** | Restrict what types a generic can accept |
| `Partial<T>` | All properties become optional |
| `Required<T>` | All properties become required |
| `Readonly<T>` | All properties become read-only (shallow) |
| `Pick<T, K>` | Keep only specified properties |
| `Omit<T, K>` | Remove specified properties |
| `Record<K, V>` | Object with typed keys and values |
| `Exclude<T, U>` | Remove union members matching U |
| `Extract<T, U>` | Keep union members matching U |
| `ReturnType<T>` | Infer a function's return type |
| `Parameters<T>` | Infer a function's parameter types as a tuple |
| `Awaited<T>` | Unwrap the resolved type of a Promise |
| **`strictNullChecks`** | Forces explicit handling of null/undefined |
| **`!` operator** | Assert a value is non-null (use sparingly) |
| **`as` assertion** | Override TypeScript's inferred type |
| **`as const`** | Narrow to literal types; create immutable values |

---

*Week 3 complete! Next up — **Week 4: Advanced Types** (Conditional Types, Mapped Types, Template Literal Types).*
