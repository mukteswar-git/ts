# TypeScript Fundamentals — Week 1: Core Type System

> **Phase 1 of your TypeScript journey.** By the end of this week you'll understand why TypeScript exists, how to configure it, and how to use its foundational type system with confidence.

---

## Table of Contents

1. [What is TypeScript and Why Use It?](#1-what-is-typescript-and-why-use-it)
2. [Setting Up TypeScript (tsconfig.json)](#2-setting-up-typescript-tsconfigjson)
3. [Primitive Types](#3-primitive-types)
4. [Arrays and Tuples](#4-arrays-and-tuples)
5. [Type Inference vs Explicit Annotations](#5-type-inference-vs-explicit-annotations)
6. [Union Types](#6-union-types)
7. [Type Aliases](#7-type-aliases)
8. [Interfaces vs Type Aliases](#8-interfaces-vs-type-aliases)
9. [Readonly and Optional Properties](#9-readonly-and-optional-properties)
10. [The `any`, `unknown`, and `never` Types](#10-the-any-unknown-and-never-types)
11. [Week 1 Exercises](#11-week-1-exercises)
12. [Summary Cheat Sheet](#12-summary-cheat-sheet)

---

## 1. What is TypeScript and Why Use It?

**TypeScript** is a statically typed superset of JavaScript developed and maintained by Microsoft. It compiles down to plain JavaScript, so it runs anywhere JavaScript runs — browsers, Node.js, Deno, etc.

### The Core Problem TypeScript Solves

JavaScript is dynamically typed, which means type errors only surface at **runtime**:

```javascript
// Plain JavaScript — no error until you run it
function greet(user) {
  return "Hello, " + user.naem; // typo: .naem instead of .name
}

greet({ name: "Alice" }); // → "Hello, undefined" 😬
```

TypeScript catches this at **compile time**, before your code ever runs:

```typescript
// TypeScript — error caught immediately in your editor
function greet(user: { name: string }): string {
  return "Hello, " + user.naem; // ❌ Error: Property 'naem' does not exist
}
```

### Key Benefits

| Benefit | Description |
|---|---|
| **Early error detection** | Catches bugs at compile time, not runtime |
| **Better IDE support** | Autocompletion, refactoring, and inline docs |
| **Self-documenting code** | Types serve as living documentation |
| **Safer refactoring** | The compiler tells you what broke |
| **Team scalability** | Easier to understand unfamiliar codebases |

### How It Works

```
Your .ts file  →  TypeScript Compiler (tsc)  →  Plain .js file
```

TypeScript adds a **type-checking layer** on top of JavaScript. The types are erased at compile time — they have zero runtime cost.

---

## 2. Setting Up TypeScript (tsconfig.json)

### Installation

```bash
# Install TypeScript globally
npm install -g typescript

# Or as a dev dependency in a project
npm install --save-dev typescript

# Verify installation
tsc --version
```

### Initializing a Project

```bash
# Creates a tsconfig.json with sensible defaults
tsc --init
```

### Understanding tsconfig.json

The `tsconfig.json` file controls how TypeScript compiles your project. Here's a well-commented starter config:

```json
{
  "compilerOptions": {
    // ─── Output ───────────────────────────────────────────
    "target": "ES2020",          // Compile to this JS version
    "module": "commonjs",        // Module system (commonjs for Node, ESNext for bundlers)
    "outDir": "./dist",          // Where compiled JS files go
    "rootDir": "./src",          // Where your TypeScript source files live

    // ─── Type Checking ────────────────────────────────────
    "strict": true,              // Enables ALL strict checks (recommended)
    "noImplicitAny": true,       // Error when type is implicitly 'any'
    "strictNullChecks": true,    // null/undefined are not assignable to other types

    // ─── Module Resolution ────────────────────────────────
    "esModuleInterop": true,     // Smoother default imports from CommonJS modules
    "resolveJsonModule": true,   // Allow importing .json files

    // ─── Developer Experience ─────────────────────────────
    "sourceMap": true,           // Generate .map files for debugging
    "declaration": true,         // Generate .d.ts type declaration files
    "noUnusedLocals": true,      // Error on unused local variables
    "noUnusedParameters": true   // Error on unused function parameters
  },
  "include": ["src/**/*"],       // Files to compile
  "exclude": ["node_modules", "dist"]
}
```

### Running TypeScript

```bash
# Compile once
tsc

# Compile and watch for changes
tsc --watch

# Run TypeScript directly (no compile step needed)
npx ts-node src/index.ts
```

### Folder Structure

```
my-project/
├── src/
│   └── index.ts       ← Your TypeScript source
├── dist/              ← Compiled JavaScript output
├── tsconfig.json
└── package.json
```

---

## 3. Primitive Types

TypeScript has direct equivalents for JavaScript's primitive types. Notice they are **lowercase** — these are the primitive types, not the wrapper objects (`String`, `Number`, etc.).

### `string`

```typescript
let firstName: string = "Alice";
let greeting: string = `Hello, ${firstName}!`; // Template literals work fine
let empty: string = "";
```

### `number`

TypeScript has a single `number` type for all numeric values (integers, floats, hex, etc.):

```typescript
let age: number = 30;
let price: number = 9.99;
let hex: number = 0xff;
let binary: number = 0b1010;
let bigInt: bigint = 100n; // bigint is a separate type
```

### `boolean`

```typescript
let isLoggedIn: boolean = true;
let hasPermission: boolean = false;
```

### `null` and `undefined`

With `strictNullChecks: true` (recommended), these are their own distinct types:

```typescript
let nothing: null = null;
let notYet: undefined = undefined;

// Without strict mode, this would be allowed:
// let name: string = null; ← ❌ Error in strict mode
```

### Why Avoid Wrapper Types?

```typescript
// ❌ Don't use these — they're object wrappers, not primitives
let s: String = "hello";
let n: Number = 42;
let b: Boolean = true;

// ✅ Always use the lowercase versions
let s: string = "hello";
let n: number = 42;
let b: boolean = true;
```

---

## 4. Arrays and Tuples

### Arrays

Two equivalent syntaxes — both are idiomatic:

```typescript
// Syntax 1: type[]
let numbers: number[] = [1, 2, 3];
let names: string[] = ["Alice", "Bob"];

// Syntax 2: Array<type>  (generic syntax)
let numbers: Array<number> = [1, 2, 3];
let names: Array<string> = ["Alice", "Bob"];
```

Arrays of complex types:

```typescript
let matrix: number[][] = [[1, 2], [3, 4]];

type User = { name: string; age: number };
let users: User[] = [
  { name: "Alice", age: 30 },
  { name: "Bob", age: 25 },
];
```

Readonly arrays:

```typescript
const ids: readonly number[] = [1, 2, 3];
// or: ReadonlyArray<number>

ids.push(4); // ❌ Error: Property 'push' does not exist on type 'readonly number[]'
```

### Tuples

A **tuple** is a fixed-length array where each position has a specific type. Think of it as a typed, ordered collection.

```typescript
// A tuple with exactly [string, number]
let person: [string, number] = ["Alice", 30];

let name = person[0]; // type: string
let age  = person[1]; // type: number

person[2]; // ❌ Error: Tuple type '[string, number]' has no element at index '2'
```

Named tuple elements (improves readability):

```typescript
let point: [x: number, y: number] = [10, 20];
let rgb: [red: number, green: number, blue: number] = [255, 128, 0];
```

Destructuring tuples:

```typescript
const [firstName, userAge]: [string, number] = ["Alice", 30];
console.log(firstName); // "Alice"
console.log(userAge);   // 30
```

Optional and rest elements:

```typescript
// Optional last element
type StringNumberBool = [string, number, boolean?];
let a: StringNumberBool = ["hello", 1];        // ✅ valid
let b: StringNumberBool = ["hello", 1, true];  // ✅ valid

// Rest element
type StringRest = [string, ...number[]];
let c: StringRest = ["hello", 1, 2, 3]; // ✅ valid
```

**When to use tuples vs arrays:**
- Use **arrays** when all elements are the same type and the length varies.
- Use **tuples** when you have a fixed structure where position carries meaning (e.g., coordinate pairs, CSV rows, function return values).

---

## 5. Type Inference vs Explicit Annotations

TypeScript is smart — it can often **infer** the type of a variable without you writing it explicitly.

### Type Inference

```typescript
// TypeScript infers: string
let city = "Tokyo";
city = 42; // ❌ Error: Type 'number' is not assignable to type 'string'

// TypeScript infers: number
let count = 0;
count++;

// TypeScript infers: boolean
let isDone = false;

// TypeScript infers: number[]
let scores = [95, 87, 92];

// TypeScript infers return type: number
function add(a: number, b: number) {
  return a + b; // inferred return: number
}
```

### Explicit Annotations

Sometimes you need to be explicit — when inference isn't possible, when you want to be intentional, or when you want to declare a wider type:

```typescript
// Explicit annotation
let message: string = "Hello";

// Useful when the variable starts empty
let items: string[] = [];
items.push("apple"); // ✅ TypeScript knows items is string[]

// Without annotation, TypeScript would infer never[]
// let items = []; ← type: never[] — can't push anything!
```

### When to Use Which?

```typescript
// ✅ Let inference work for simple initializations
let name = "Alice";
let count = 0;
const PI = 3.14159;

// ✅ Be explicit when:
// 1. Variable is declared without initialization
let result: string;

// 2. You want a wider type than the literal
let status: string = "active"; // not inferred as literal "active"

// 3. Function parameters (always annotate these)
function greet(name: string): string {
  return `Hello, ${name}`;
}

// 4. When the inferred type isn't what you want
let mixed: (string | number)[] = [];
```

---

## 6. Union Types

A **union type** represents a value that can be one of several types. Use the `|` (pipe) operator.

```typescript
// A variable that can be string OR number
let id: string | number;
id = "abc-123"; // ✅
id = 42;        // ✅
id = true;      // ❌ Error: Type 'boolean' is not assignable to type 'string | number'
```

### Narrowing — Working Safely with Unions

When you have a union, TypeScript requires you to **narrow** the type before using type-specific methods:

```typescript
function printId(id: string | number): void {
  // id could be either — can't call .toUpperCase() directly

  if (typeof id === "string") {
    // TypeScript knows: id is string here
    console.log(id.toUpperCase());
  } else {
    // TypeScript knows: id is number here
    console.log(id.toFixed(2));
  }
}
```

### Union with `null` — Nullable Types

```typescript
// A common pattern: a value that might not exist yet
let username: string | null = null;

// Later...
username = "alice_dev";

// Safe usage with nullish coalescing
const display = username ?? "Anonymous";
```

### Union with Literals

You can create unions of specific **literal values** — great for representing a fixed set of states:

```typescript
type Direction = "north" | "south" | "east" | "west";
type Status = "pending" | "active" | "inactive" | "deleted";
type DiceRoll = 1 | 2 | 3 | 4 | 5 | 6;

let move: Direction = "north"; // ✅
let move2: Direction = "up";   // ❌ Error: Type '"up"' is not assignable to type 'Direction'
```

### Discriminated Unions

A powerful pattern for modeling complex state:

```typescript
type LoadingState = { status: "loading" };
type SuccessState = { status: "success"; data: string[] };
type ErrorState   = { status: "error";   message: string };

type FetchState = LoadingState | SuccessState | ErrorState;

function handleState(state: FetchState): void {
  switch (state.status) {
    case "loading":
      console.log("Loading...");
      break;
    case "success":
      console.log(state.data); // TypeScript knows: data is string[]
      break;
    case "error":
      console.log(state.message); // TypeScript knows: message is string
      break;
  }
}
```

---

## 7. Type Aliases

A **type alias** gives a name to any type — primitives, unions, objects, tuples, functions, and more.

```typescript
// Basic alias
type UserId = string;
type Score = number;
type Pair = [string, number];

// Object shape
type Point = {
  x: number;
  y: number;
};

// Union
type StringOrNumber = string | number;

// Function signature
type Formatter = (value: string) => string;

// Using the aliases
const userId: UserId = "usr_abc123";
const origin: Point = { x: 0, y: 0 };

const toUpper: Formatter = (s) => s.toUpperCase();
```

### Composing Type Aliases

```typescript
type Name = {
  firstName: string;
  lastName: string;
};

type Contact = {
  email: string;
  phone?: string;
};

// Intersection (&) — combine multiple types
type Person = Name & Contact & {
  age: number;
};

const alice: Person = {
  firstName: "Alice",
  lastName: "Smith",
  email: "alice@example.com",
  age: 30,
};
```

### Generic Type Aliases

```typescript
// A reusable "maybe" type
type Maybe<T> = T | null | undefined;

let userName: Maybe<string> = null;
let userAge: Maybe<number> = 25;

// A key-value map
type Dictionary<T> = {
  [key: string]: T;
};

const wordCount: Dictionary<number> = {
  hello: 3,
  world: 5,
};
```

---

## 8. Interfaces vs Type Aliases

Both `interface` and `type` can describe object shapes. They're often interchangeable, but have key differences.

### Defining an Interface

```typescript
interface User {
  id: number;
  name: string;
  email: string;
}

const user: User = {
  id: 1,
  name: "Alice",
  email: "alice@example.com",
};
```

### Key Difference 1: Declaration Merging

Interfaces support **declaration merging** — you can declare the same interface multiple times and TypeScript merges them. Type aliases do not support this.

```typescript
// ✅ Interfaces merge automatically
interface Window {
  title: string;
}
interface Window {
  theme: "light" | "dark";
}
// Window now has both title AND theme

// ❌ Type aliases cannot be redeclared
type Config = { debug: boolean };
type Config = { verbose: boolean }; // Error: Duplicate identifier 'Config'
```

This is especially useful when extending third-party library types.

### Key Difference 2: Extension Syntax

```typescript
// Interface — uses 'extends'
interface Animal {
  name: string;
}
interface Dog extends Animal {
  breed: string;
}

// Type alias — uses intersection (&)
type Animal = { name: string };
type Dog = Animal & { breed: string };

// Both produce the same result:
const dog: Dog = { name: "Rex", breed: "Labrador" };
```

### Key Difference 3: What They Can Describe

```typescript
// Type aliases can describe anything
type StringOrNumber = string | number;       // ✅ unions
type Callback = () => void;                  // ✅ functions
type Pair = [string, number];                // ✅ tuples
type Primitive = string | number | boolean;  // ✅ complex unions

// Interfaces can only describe object shapes
interface Runnable {
  run(): void; // ✅ method signatures
}
// interface ID = string | number; ← ❌ not possible
```

### When to Use Which?

| Situation | Recommendation |
|---|---|
| Describing object shapes (classes, props) | Either — `interface` is slightly preferred |
| Building public library API types | `interface` (allows consumers to extend via merging) |
| Union types, tuple types, mapped types | `type` (only option) |
| Extending third-party types | `interface` (declaration merging) |
| Personal preference / consistency | Pick one and be consistent |

> **Practical rule:** Use `interface` for object shapes and class contracts. Use `type` for everything else (unions, tuples, primitives, complex compositions).

---

## 9. Readonly and Optional Properties

### Optional Properties (`?`)

An optional property may or may not be present. Its type is automatically `T | undefined`.

```typescript
interface UserProfile {
  id: number;
  name: string;
  bio?: string;        // optional: string | undefined
  avatarUrl?: string;  // optional: string | undefined
}

// Both of these are valid:
const fullProfile: UserProfile = {
  id: 1,
  name: "Alice",
  bio: "TypeScript enthusiast",
  avatarUrl: "https://example.com/alice.jpg",
};

const minimalProfile: UserProfile = {
  id: 2,
  name: "Bob",
  // bio and avatarUrl are omitted — that's fine!
};
```

Accessing optional properties safely:

```typescript
// Optional chaining — won't throw if bio is undefined
const bioLength = profile.bio?.length;

// Nullish coalescing — provide a fallback
const bio = profile.bio ?? "No bio provided.";
```

### Readonly Properties

A `readonly` property can be set once (during initialization) but never changed afterward.

```typescript
interface Config {
  readonly apiUrl: string;
  readonly maxRetries: number;
  timeout: number; // mutable
}

const config: Config = {
  apiUrl: "https://api.example.com",
  maxRetries: 3,
  timeout: 5000,
};

config.timeout = 10000;  // ✅ OK — timeout is mutable
config.apiUrl = "other"; // ❌ Error: Cannot assign to 'apiUrl' because it is a read-only property
```

### `Readonly<T>` Utility Type

Make all properties of a type readonly at once:

```typescript
interface Point {
  x: number;
  y: number;
}

const origin: Readonly<Point> = { x: 0, y: 0 };
origin.x = 5; // ❌ Error: Cannot assign to 'x' because it is a read-only property
```

### `readonly` in Arrays

```typescript
const ids: readonly number[] = [1, 2, 3];
// Equivalent: ReadonlyArray<number>

ids.push(4);    // ❌ Error
ids[0] = 99;    // ❌ Error
ids.length;     // ✅ Reading is fine
```

### Combining Readonly and Optional

```typescript
interface ImmutableUser {
  readonly id: number;
  readonly createdAt: Date;
  name: string;          // mutable
  nickname?: string;     // mutable and optional
  readonly role: "admin" | "user"; // readonly but required
}
```

---

## 10. The `any`, `unknown`, and `never` Types

These three special types sit at the edges of TypeScript's type system.

### `any` — The Escape Hatch

`any` turns off type checking for a value. It's the equivalent of plain JavaScript.

```typescript
let value: any = "hello";
value = 42;        // ✅ no error
value = true;      // ✅ no error
value = { x: 1 }; // ✅ no error

// TypeScript won't catch these runtime errors:
value.foo.bar.baz; // no error at compile time 😬
value();           // no error at compile time 😬
```

**When to use `any`:**
- Migrating JavaScript codebases to TypeScript gradually
- Working with truly dynamic data (e.g., poorly typed third-party APIs)
- As a last resort when types are too complex

**The problem:** Using `any` defeats the purpose of TypeScript. Prefer `unknown` instead.

### `unknown` — The Safe Alternative to `any`

`unknown` is the type-safe counterpart to `any`. A value of type `unknown` can hold anything, but you **must narrow the type** before doing anything with it.

```typescript
let value: unknown = "hello";
value = 42;   // ✅ assigning is fine

// You CANNOT use it without narrowing:
value.toUpperCase(); // ❌ Error: Object is of type 'unknown'
value + 1;           // ❌ Error

// ✅ Narrow first, then use:
if (typeof value === "string") {
  console.log(value.toUpperCase()); // now it's safe
}

if (typeof value === "number") {
  console.log(value * 2);
}
```

**`unknown` vs `any` — the key difference:**

```typescript
function processInput(input: any) {
  input.toUpperCase(); // ✅ TypeScript trusts you (but it might crash!)
}

function processInputSafely(input: unknown) {
  input.toUpperCase(); // ❌ TypeScript forces you to check first
  
  if (typeof input === "string") {
    input.toUpperCase(); // ✅ Now TypeScript is happy
  }
}
```

> **Rule of thumb:** Prefer `unknown` over `any` when you genuinely don't know the type. It keeps safety while allowing flexibility.

### `never` — The Impossible Type

`never` represents a value that **never occurs**. It's the return type of functions that never finish (throw or loop forever), and it appears in exhaustive checks.

```typescript
// A function that always throws — it never returns
function throwError(message: string): never {
  throw new Error(message);
}

// An infinite loop — never returns
function runForever(): never {
  while (true) {
    // ...
  }
}
```

`never` in exhaustive type checking:

```typescript
type Shape = "circle" | "square" | "triangle";

function getArea(shape: Shape): number {
  switch (shape) {
    case "circle":   return Math.PI * 5 * 5;
    case "square":   return 5 * 5;
    case "triangle": return 0.5 * 5 * 5;
    default:
      // If you add a new Shape but forget to handle it here,
      // TypeScript will error because shape would be 'never'
      const exhaustiveCheck: never = shape; // ❌ Error if any case is unhandled
      return exhaustiveCheck;
  }
}
```

`never` in conditional types (advanced preview):

```typescript
// string[] filtered to remove numbers → result is string[], never number
type ExcludeNumber<T> = T extends number ? never : T;
type StringOnly = ExcludeNumber<string | number | boolean>;
// → string | boolean
```

### Quick Comparison Table

| Type | Assignable from | Assignable to | Use case |
|---|---|---|---|
| `any` | Everything | Everything | Escape hatch, JS migration |
| `unknown` | Everything | Only `unknown` or `any` | Safe dynamic values |
| `never` | Nothing | Everything | Unreachable code, exhaustive checks |

---

## 11. Week 1 Exercises

Work through these exercises to solidify your understanding. Try to complete them without looking back at the notes!

### Exercise 1 — Fix the Types
```typescript
// Fix all type errors in this code:
let productName = 123;
let inStock: boolean = "yes";
let price: string = 29.99;

function getDiscount(amount) {
  return amount * 0.1;
}
```

### Exercise 2 — Model a Product
```typescript
// Create a type alias for a Product with:
// - id (readonly, number)
// - name (string)
// - price (number)
// - description (optional string)
// - tags (array of strings)
// - status ("available" | "out_of_stock" | "discontinued")

// Then create two valid Product objects
```

### Exercise 3 — Union Narrowing
```typescript
// Write a function formatValue(value: string | number | boolean): string
// that:
// - If string: returns it in UPPER CASE
// - If number: returns it formatted with 2 decimal places
// - If boolean: returns "YES" or "NO"
```

### Exercise 4 — Interface Extension
```typescript
// Create:
// 1. An interface 'Vehicle' with: make, model, year
// 2. An interface 'Car' extending Vehicle with: doors, fuelType ("gas"|"diesel"|"electric")
// 3. An interface 'ElectricCar' extending Car with: batteryRangeKm (number)

// Create a valid ElectricCar object
```

### Exercise 5 — Safe Unknown Handling
```typescript
// Write a function safeStringLength(value: unknown): number
// that returns the length of the string if value is a string,
// or -1 if it's not a string.
// Do NOT use 'any'.
```

---

## 12. Summary Cheat Sheet

```typescript
// ─── Primitive Types ──────────────────────────────────────
let name: string = "Alice";
let age: number = 30;
let active: boolean = true;
let nothing: null = null;
let missing: undefined = undefined;

// ─── Arrays ───────────────────────────────────────────────
let nums: number[] = [1, 2, 3];
let strs: Array<string> = ["a", "b"];
let matrix: number[][] = [[1, 2], [3, 4]];
let readOnly: readonly string[] = ["x", "y"];

// ─── Tuples ───────────────────────────────────────────────
let point: [number, number] = [10, 20];
let entry: [string, number, boolean?] = ["id", 1];

// ─── Inference vs Annotation ──────────────────────────────
let inferred = "hello";          // type: string (inferred)
let explicit: string = "hello";  // type: string (annotated)

// ─── Union Types ──────────────────────────────────────────
let id: string | number = "abc";
type Status = "active" | "inactive" | "pending";

// ─── Type Aliases ─────────────────────────────────────────
type Point = { x: number; y: number };
type StringOrNum = string | number;
type Callback = (data: string) => void;
type Combined = Point & { label: string };  // intersection

// ─── Interfaces ───────────────────────────────────────────
interface User { id: number; name: string }
interface AdminUser extends User { role: "admin" }

// ─── Optional & Readonly ──────────────────────────────────
interface Profile {
  readonly id: number;
  name: string;
  bio?: string;         // optional
}

// ─── Special Types ────────────────────────────────────────
let a: any = "anything goes";
let b: unknown = "must narrow before use";
function fail(msg: string): never { throw new Error(msg); }

// ─── Narrowing ────────────────────────────────────────────
function handle(val: string | number) {
  if (typeof val === "string") { /* val is string */ }
  else                         { /* val is number */ }
}
```

---

> **Coming Up in Week 2:** Functions in depth — overloads, generics, rest parameters, and utility types like `Partial<T>`, `Required<T>`, `Pick<T>`, and `Omit<T>`.

---

*TypeScript Fundamentals · Phase 1 · Week 1 of 12*
