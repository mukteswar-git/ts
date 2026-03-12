# Week 4: Advanced Type Manipulation in TypeScript

> **Prerequisites:** Familiarity with TypeScript generics, interfaces, and basic types.  
> **Goal:** Master TypeScript's most powerful type-level programming tools.

---

## Table of Contents

1. [keyof and typeof Operators](#1-keyof-and-typeof-operators)
2. [Index Access Types — T[K]](#2-index-access-types--tk)
3. [Mapped Types](#3-mapped-types)
4. [Conditional Types](#4-conditional-types)
5. [The infer Keyword](#5-the-infer-keyword)
6. [Template Literal Types](#6-template-literal-types)
7. [Recursive Types](#7-recursive-types)
8. [Distributive Conditional Types](#8-distributive-conditional-types)
9. [Combining Everything: Real-World Patterns](#9-combining-everything-real-world-patterns)
10. [Exercises](#10-exercises)

---

## 1. `keyof` and `typeof` Operators

### `keyof` — Extract the Keys of a Type

`keyof T` produces a union of the **string (or number/symbol) literal types** that are the keys of `T`.

```typescript
interface User {
  id: number;
  name: string;
  email: string;
}

type UserKeys = keyof User;
// => "id" | "name" | "email"
```

**Why is this useful?** It lets you write functions that are constrained to valid property names:

```typescript
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user: User = { id: 1, name: "Alice", email: "alice@example.com" };

const name = getProperty(user, "name");   // ✅ string
const id   = getProperty(user, "id");     // ✅ number
// getProperty(user, "age");             // ❌ Error: "age" not in keyof User
```

---

### `typeof` — Capture the Type of a Value

`typeof` at the **type level** extracts the TypeScript type of a JavaScript value or variable.

```typescript
const config = {
  host: "localhost",
  port: 3000,
  debug: true,
};

type Config = typeof config;
// => { host: string; port: number; debug: boolean }
```

**Combining `keyof` and `typeof`:**

```typescript
type ConfigKeys = keyof typeof config;
// => "host" | "port" | "debug"

function setConfig<K extends keyof typeof config>(
  key: K,
  value: (typeof config)[K]
) {
  config[key] = value;
}

setConfig("port", 8080);   // ✅
setConfig("debug", false); // ✅
// setConfig("port", "8080"); // ❌ Error: string not assignable to number
```

> **Key Insight:** `typeof` works on variables/values; `keyof` works on types. Use them together to derive types from runtime data structures.

---

## 2. Index Access Types — `T[K]`

Index access types let you look up the type of a specific **property** within another type, just like bracket notation in JavaScript.

```typescript
interface Product {
  id: number;
  name: string;
  price: number;
  tags: string[];
}

type ProductName  = Product["name"];   // => string
type ProductPrice = Product["price"];  // => number
type ProductTags  = Product["tags"];   // => string[]
type TagItem      = Product["tags"][number]; // => string  (element of the array)
```

### Union Keys

You can pass a **union** as the key to get a union of types:

```typescript
type IdOrName = Product["id" | "name"];
// => number | string

type AllValues = Product[keyof Product];
// => number | string | string[]
```

### Practical Example: Extracting Nested Types

```typescript
interface ApiResponse {
  data: {
    users: {
      id: number;
      profile: {
        avatar: string;
        bio: string;
      };
    }[];
  };
  status: number;
}

type Users        = ApiResponse["data"]["users"];            // => { id: number; profile: {...} }[]
type SingleUser   = ApiResponse["data"]["users"][number];    // => { id: number; profile: {...} }
type UserProfile  = ApiResponse["data"]["users"][number]["profile"]; // => { avatar: string; bio: string }
type AvatarType   = UserProfile["avatar"];                   // => string
```

---

## 3. Mapped Types

Mapped types let you **transform every property** of an existing type into something new. The syntax is:

```typescript
{ [K in keyof T]: NewType }
```

### The Building Blocks

```typescript
interface Point {
  x: number;
  y: number;
}

// Make all properties optional
type PartialPoint = { [K in keyof Point]?: Point[K] };
// => { x?: number; y?: number }

// Make all properties readonly
type ReadonlyPoint = { readonly [K in keyof Point]: Point[K] };
// => { readonly x: number; readonly y: number }

// Make all properties nullable
type NullablePoint = { [K in keyof Point]: Point[K] | null };
// => { x: number | null; y: number | null }
```

These patterns are so common that TypeScript ships them as **built-in utility types**: `Partial<T>`, `Readonly<T>`, `Required<T>`, `Record<K, V>`.

---

### Remapping Keys with `as`

TypeScript 4.1+ lets you **rename keys** inside a mapped type using `as`:

```typescript
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

interface Person {
  name: string;
  age: number;
}

type PersonGetters = Getters<Person>;
// => {
//   getName: () => string;
//   getAge:  () => number;
// }
```

### Filtering Properties with `as never`

Map a key to `never` to **remove** it:

```typescript
type OnlyStrings<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K];
};

interface Mixed {
  id: number;
  name: string;
  active: boolean;
  title: string;
}

type StringOnly = OnlyStrings<Mixed>;
// => { name: string; title: string }
```

### Mapped Type Modifiers: `+` and `-`

Use `-` to **remove** `optional` or `readonly` modifiers:

```typescript
// Remove optional from all properties
type Required<T> = { [K in keyof T]-?: T[K] };

// Remove readonly from all properties
type Mutable<T> = { -readonly [K in keyof T]: T[K] };
```

---

## 4. Conditional Types

Conditional types express **type-level if/else** logic:

```typescript
T extends U ? TrueType : FalseType
```

If `T` is assignable to `U`, the result is `TrueType`; otherwise `FalseType`.

### Basic Examples

```typescript
type IsString<T> = T extends string ? true : false;

type A = IsString<string>;   // => true
type B = IsString<number>;   // => false
type C = IsString<"hello">;  // => true  (literal string extends string)
```

### Practical Use Case: Unwrapping Promises

```typescript
type Awaited<T> = T extends Promise<infer R> ? R : T;

type Resolved = Awaited<Promise<string>>;  // => string
type Plain    = Awaited<number>;           // => number
```

> `Awaited<T>` is now a built-in TypeScript utility. Understanding how it works under the hood is key.

### Nested Conditional Types

```typescript
type TypeName<T> =
  T extends string  ? "string"  :
  T extends number  ? "number"  :
  T extends boolean ? "boolean" :
  T extends null    ? "null"    :
  T extends undefined ? "undefined" :
  "object";

type T1 = TypeName<string>;    // => "string"
type T2 = TypeName<42>;        // => "number"
type T3 = TypeName<string[]>;  // => "object"
```

---

## 5. The `infer` Keyword

`infer` is used **inside conditional types** to declare a type variable that TypeScript will **infer** from the matched type. Think of it as a capture group for types.

### Syntax

```typescript
T extends SomeType<infer R> ? R : never
```

### Extracting the Return Type of a Function

```typescript
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

function greet(name: string): string {
  return `Hello, ${name}`;
}

type GreetReturn = ReturnType<typeof greet>;
// => string
```

### Extracting Function Parameters

```typescript
type Parameters<T> = T extends (...args: infer P) => any ? P : never;

function createUser(name: string, age: number, admin: boolean): void {}

type CreateUserParams = Parameters<typeof createUser>;
// => [name: string, age: number, admin: boolean]
```

### Extracting from Arrays and Promises

```typescript
// Array element type
type ElementType<T> = T extends (infer E)[] ? E : never;

type Elem = ElementType<string[]>;   // => string
type Elem2 = ElementType<number[]>;  // => number

// Unwrap nested promises (recursive with infer)
type DeepAwaited<T> = T extends Promise<infer R> ? DeepAwaited<R> : T;

type Result = DeepAwaited<Promise<Promise<string>>>;  // => string
```

### Inferring from Object Properties

```typescript
type FirstArgument<T> = T extends (first: infer F, ...rest: any[]) => any
  ? F
  : never;

type FA = FirstArgument<(x: number, y: string) => void>;  // => number
```

> **Remember:** `infer` only works inside the `extends` clause of a conditional type. You cannot use it elsewhere.

---

## 6. Template Literal Types

Introduced in TypeScript 4.1, template literal types let you **construct string literal types** using interpolation — just like JavaScript template literals, but at the type level.

### Basic Syntax

```typescript
type Greeting = `Hello, ${string}`;
// Any string starting with "Hello, "

type EventName = `on${string}`;
// "onClick", "onChange", "onSubmit", etc.
```

### Combining with Unions

When a union is used inside a template literal, TypeScript **distributes** it to produce all combinations:

```typescript
type Direction = "top" | "right" | "bottom" | "left";
type Property  = "margin" | "padding";

type SpacingProp = `${Property}-${Direction}`;
// => "margin-top" | "margin-right" | "margin-bottom" | "margin-left"
//  | "padding-top" | "padding-right" | "padding-bottom" | "padding-left"
```

### Built-in String Manipulation Types

TypeScript ships four utility types for template literals:

```typescript
type Upper = Uppercase<"hello">;         // => "HELLO"
type Lower = Lowercase<"HELLO">;         // => "hello"
type Cap   = Capitalize<"hello">;        // => "Hello"
type Uncap = Uncapitalize<"Hello">;      // => "hello"
```

### Real-World Pattern: Typed Event System

```typescript
type EventMap = {
  click: MouseEvent;
  keydown: KeyboardEvent;
  resize: UIEvent;
};

type EventHandler<T extends keyof EventMap> = `on${Capitalize<T>}`;

type ClickHandler  = EventHandler<"click">;   // => "onClick"
type KeydownHandler = EventHandler<"keydown">; // => "onKeydown"

// Build a typed props object for all events
type EventProps = {
  [K in keyof EventMap as `on${Capitalize<K>}`]: (event: EventMap[K]) => void;
};
// => {
//   onClick:   (event: MouseEvent) => void;
//   onKeydown: (event: KeyboardEvent) => void;
//   onResize:  (event: UIEvent) => void;
// }
```

### Extracting Parts of String Literals with `infer`

```typescript
type ExtractRouteParams<T extends string> =
  T extends `${infer _Start}:${infer Param}/${infer Rest}`
    ? Param | ExtractRouteParams<`/${Rest}`>
    : T extends `${infer _Start}:${infer Param}`
    ? Param
    : never;

type Params = ExtractRouteParams<"/users/:userId/posts/:postId">;
// => "userId" | "postId"
```

---

## 7. Recursive Types

TypeScript supports types that **reference themselves**, enabling you to model deeply nested or tree-structured data.

### Simple Recursive Type

```typescript
type NestedArray<T> = T | NestedArray<T>[];

const flat: NestedArray<number> = 1;
const nested: NestedArray<number> = [1, [2, [3, [4]]]];
```

### JSON Value Type

```typescript
type JSONValue =
  | string
  | number
  | boolean
  | null
  | JSONValue[]
  | { [key: string]: JSONValue };

const data: JSONValue = {
  name: "Alice",
  scores: [10, 20, 30],
  meta: {
    active: true,
    address: null,
  },
};
```

### Recursive Mapped Type: DeepReadonly

```typescript
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object
    ? DeepReadonly<T[K]>
    : T[K];
};

interface Config {
  server: {
    host: string;
    port: number;
    tls: {
      enabled: boolean;
      cert: string;
    };
  };
  debug: boolean;
}

type FrozenConfig = DeepReadonly<Config>;
// All nested properties are readonly — even config.server.tls.enabled
```

### Recursive Conditional Type: DeepPartial

```typescript
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K];
};

type PartialConfig = DeepPartial<Config>;
// Every property at every depth is optional
```

### Tree Structure

```typescript
interface TreeNode<T> {
  value: T;
  children?: TreeNode<T>[];
}

const tree: TreeNode<string> = {
  value: "root",
  children: [
    { value: "branch1", children: [{ value: "leaf1" }] },
    { value: "branch2" },
  ],
};
```

> **Caution:** TypeScript has a recursion depth limit. Extremely deep recursive types may cause performance issues or compiler errors. Use with intention.

---

## 8. Distributive Conditional Types

When a **bare type parameter** is used in a conditional type, TypeScript **distributes** the condition over each member of a union automatically.

### Understanding Distribution

```typescript
type ToArray<T> = T extends any ? T[] : never;

type A = ToArray<string | number>;
// Distributed as: ToArray<string> | ToArray<number>
// => string[] | number[]
```

Compare to a **non-distributive** version (wrap in a tuple):

```typescript
type ToArrayNonDist<T> = [T] extends [any] ? T[] : never;

type B = ToArrayNonDist<string | number>;
// NOT distributed — treats the whole union as one type
// => (string | number)[]
```

### Filtering with Distribution

Distribution combined with `never` is a powerful **filter**:

```typescript
// Keep only types assignable to U
type Filter<T, U> = T extends U ? T : never;

type Strings = Filter<string | number | boolean | null, string | null>;
// => string | null

// Exclude types assignable to U (same as built-in Exclude<T, U>)
type Exclude<T, U> = T extends U ? never : T;

type NonNull = Exclude<string | number | null | undefined, null | undefined>;
// => string | number
```

### Built-in Utilities That Use Distribution

```typescript
// Extract<T, U>  — keep members of T assignable to U
type Extract<T, U> = T extends U ? T : never;

// NonNullable<T> — remove null and undefined
type NonNullable<T> = T extends null | undefined ? never : T;

// ReturnType<T>
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
```

### Careful: Distribution Only on Bare Parameters

```typescript
// T is "bare" — distribution happens
type Distributed<T> = T extends string ? "yes" : "no";
type D = Distributed<string | number>;  // => "yes" | "no"

// T is "wrapped" — no distribution
type NotDistributed<T> = [T] extends [string] ? "yes" : "no";
type ND = NotDistributed<string | number>;  // => "no"  (union not assignable to string)
```

---

## 9. Combining Everything: Real-World Patterns

### Pattern 1: Strongly-Typed Event Emitter

```typescript
type EventMap = Record<string, unknown>;

class TypedEmitter<Events extends EventMap> {
  private listeners: {
    [K in keyof Events]?: Array<(data: Events[K]) => void>;
  } = {};

  on<K extends keyof Events>(event: K, listener: (data: Events[K]) => void): void {
    if (!this.listeners[event]) this.listeners[event] = [];
    this.listeners[event]!.push(listener);
  }

  emit<K extends keyof Events>(event: K, data: Events[K]): void {
    this.listeners[event]?.forEach((fn) => fn(data));
  }
}

// Usage
interface AppEvents {
  login:  { userId: string; timestamp: number };
  logout: { userId: string };
  error:  { message: string; code: number };
}

const emitter = new TypedEmitter<AppEvents>();
emitter.on("login", ({ userId, timestamp }) => console.log(userId, timestamp));
emitter.emit("login", { userId: "u1", timestamp: Date.now() }); // ✅ fully typed
```

---

### Pattern 2: Deep Object Path Access

```typescript
type Paths<T, Prefix extends string = ""> = {
  [K in keyof T & string]:
    T[K] extends object
      ? `${Prefix}${K}` | Paths<T[K], `${Prefix}${K}.`>
      : `${Prefix}${K}`;
}[keyof T & string];

interface AppState {
  user: {
    name: string;
    address: {
      city: string;
      zip: string;
    };
  };
  theme: string;
}

type AppPaths = Paths<AppState>;
// => "user" | "user.name" | "user.address" | "user.address.city"
//  | "user.address.zip" | "theme"
```

---

### Pattern 3: API Response Transformer

```typescript
type ApiSuccess<T> = { status: "success"; data: T };
type ApiError      = { status: "error"; message: string };
type ApiResponse<T> = ApiSuccess<T> | ApiError;

// Extract data type from a success response
type UnwrapSuccess<T> = T extends ApiSuccess<infer D> ? D : never;

type UsersResponse = ApiResponse<{ id: number; name: string }[]>;
type UsersData     = UnwrapSuccess<UsersResponse>;
// => { id: number; name: string }[]

// Map an object of endpoint definitions to their response data types
type EndpointMap = {
  getUser:   ApiResponse<{ id: number; name: string }>;
  listPosts: ApiResponse<{ id: number; title: string }[]>;
  deletePost: ApiResponse<void>;
};

type SuccessDataMap = {
  [K in keyof EndpointMap]: UnwrapSuccess<EndpointMap[K]>;
};
// => {
//   getUser:   { id: number; name: string };
//   listPosts: { id: number; title: string }[];
//   deletePost: void;
// }
```

---

## 10. Exercises

Work through these in order. Each builds on previous concepts.

---

**Exercise 1 — `keyof` + Index Access**

Given the type below, create a generic `pick` function that returns a new object containing only the specified keys.

```typescript
interface Car {
  make: string;
  model: string;
  year: number;
  color: string;
}

function pick<T, K extends keyof T>(obj: T, keys: K[]): Pick<T, K> {
  // Your implementation here
}

const car: Car = { make: "Toyota", model: "Camry", year: 2022, color: "blue" };
const result = pick(car, ["make", "year"]);
// result should be: { make: "Toyota", year: 2022 }
```

---

**Exercise 2 — Mapped Types**

Create a `Nullable<T>` mapped type that makes every property of `T` nullable (i.e., `T[K] | null`). Then create a `DeepNullable<T>` that applies this recursively.

---

**Exercise 3 — Conditional Types**

Write a conditional type `IsArray<T>` that returns `true` if `T` is an array, and `false` otherwise. Then write `UnwrapArray<T>` that returns the element type if `T` is an array, otherwise returns `T`.

```typescript
type IsArray<T> = /* your answer */
type UnwrapArray<T> = /* your answer */

type A = IsArray<string[]>;   // => true
type B = IsArray<string>;     // => false
type C = UnwrapArray<number[]>; // => number
type D = UnwrapArray<boolean>;  // => boolean
```

---

**Exercise 4 — `infer`**

Write a type `ConstructorParameters<T>` that extracts the parameter types of a class constructor.

```typescript
class Database {
  constructor(host: string, port: number, name: string) {}
}

type DBParams = ConstructorParameters<typeof Database>;
// => [host: string, port: number, name: string]
```

---

**Exercise 5 — Template Literal Types**

Given a union of HTTP methods, create a type that produces all combinations of method + path:

```typescript
type Method = "GET" | "POST" | "PUT" | "DELETE";
type Resource = "users" | "posts" | "comments";

type Endpoint = /* your answer */
// => "GET /users" | "GET /posts" | "GET /comments"
//  | "POST /users" | ...  (all 12 combinations)
```

---

**Exercise 6 — Recursive + Conditional (Challenge)**

Write a type `Flatten<T>` that recursively unwraps nested arrays:

```typescript
type Flatten<T> = /* your answer */

type A = Flatten<number>;           // => number
type B = Flatten<number[]>;         // => number
type C = Flatten<number[][]>;       // => number
type D = Flatten<string[][][]>;     // => string
```

---

## Quick Reference Cheat Sheet

| Feature | Syntax | Use Case |
|---|---|---|
| `keyof` | `keyof T` | Get union of keys of a type |
| `typeof` | `typeof value` | Get type of a value/variable |
| Index Access | `T[K]` | Look up property type |
| Mapped Type | `{ [K in keyof T]: ... }` | Transform all properties |
| Remap Keys | `{ [K in keyof T as NewK]: ... }` | Rename/filter properties |
| Conditional | `T extends U ? X : Y` | Type-level if/else |
| `infer` | `T extends F<infer R> ? R : never` | Capture and extract types |
| Template Literal | `` `prefix_${T}` `` | Construct string literal types |
| Recursive | `type T = ... \| T[]` | Model nested/tree data |
| Distributive | `T extends U ? ...` (bare T) | Apply condition across union |

---

*End of Week 4: Advanced Type Manipulation*
