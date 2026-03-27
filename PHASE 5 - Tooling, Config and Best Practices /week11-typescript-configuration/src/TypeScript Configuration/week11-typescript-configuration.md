# Week 11: TypeScript Configuration

> A complete, practical guide to mastering TypeScript project setup, tooling, and type safety infrastructure.

---

## Table of Contents

1. [tsconfig.json Deep Dive](#1-tsconfigjson-deep-dive)
2. [Project References and Monorepos](#2-project-references-and-monorepos)
3. [Path Aliases and baseUrl](#3-path-aliases-and-baseurl)
4. [Declaration Files (.d.ts)](#4-declaration-files-dts)
5. [Writing Your Own .d.ts for JS Libraries](#5-writing-your-own-dts-for-js-libraries)
6. [ESLint with TypeScript](#6-eslint-with-typescript)
7. [Prettier + TypeScript](#7-prettier--typescript)
8. [ts-prune and Type Coverage Tools](#8-ts-prune-and-type-coverage-tools)

---

## 1. tsconfig.json Deep Dive

The `tsconfig.json` file is the heart of every TypeScript project. It controls how TypeScript compiles your code, what JavaScript features are available, and how strict type checking should be.

### Basic Structure

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "lib": ["ES2022", "DOM"],
    "strict": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

---

### `target` — JavaScript Output Version

The `target` option tells TypeScript which version of JavaScript to compile your code **down to**.

| Value | Use Case |
|-------|----------|
| `ES5` | Maximum browser compatibility (legacy) |
| `ES2017` | Async/await support without polyfills |
| `ES2020` | Optional chaining, nullish coalescing |
| `ES2022` | Top-level await, class fields |
| `ESNext` | Always the latest (risky for production) |

```json
// For a Node.js 18+ backend
{ "target": "ES2022" }

// For a browser app supporting older browsers
{ "target": "ES2017" }
```

> **Key insight:** `target` affects what syntax gets *downleveled*. If you target `ES5`, TypeScript rewrites arrow functions and classes. If you target `ES2022`, they pass through unchanged.

---

### `lib` — Type Definitions to Include

`lib` controls which **built-in type definitions** TypeScript includes. These are NOT polyfills — they are just the *type shapes* of runtime APIs.

```json
{
  "lib": ["ES2022", "DOM", "DOM.Iterable"]
}
```

Common `lib` values:

- `ES2022` — Standard JS globals (`Promise`, `Map`, `Set`, `Array.at()`, etc.)
- `DOM` — Browser APIs (`document`, `window`, `fetch`, `HTMLElement`)
- `DOM.Iterable` — Makes `NodeList`, `HTMLCollection` iterable
- `WebWorker` — Service worker / web worker APIs
- `ESNext.Array` — Bleeding-edge array methods

```typescript
// Without "DOM" in lib, this gives an error:
document.getElementById("app"); // Error: Cannot find name 'document'

// Without "ES2022" in lib, this gives an error:
const arr = [1, 2, 3];
arr.at(-1); // Error: Property 'at' does not exist on type 'number[]'
```

> **Node.js tip:** For Node.js projects, omit `DOM` entirely. Use `@types/node` instead.

---

### `module` — Module System

`module` controls the **output module format** TypeScript emits.

| Value | Use Case |
|-------|----------|
| `CommonJS` | Node.js (require/module.exports) |
| `ESNext` | Bundlers (Vite, webpack, esbuild) |
| `NodeNext` | Modern Node.js with native ESM |
| `Preserve` | Don't transform imports (for bundlers) |

```json
// For a Node.js project using package.json "type": "module"
{ "module": "NodeNext", "moduleResolution": "NodeNext" }

// For a project using a bundler (Vite, webpack)
{ "module": "ESNext", "moduleResolution": "Bundler" }

// For a classic Node.js project (CommonJS)
{ "module": "CommonJS", "moduleResolution": "Node" }
```

> **Critical:** `module` and `moduleResolution` should always be kept in sync. The `NodeNext` and `Bundler` moduleResolution modes are the modern choices.

---

### `strict` — The Strict Mode Flag

Setting `"strict": true` enables a collection of strict type-checking flags all at once:

| Flag | What It Catches |
|------|-----------------|
| `strictNullChecks` | Prevents `null`/`undefined` from being used as any type |
| `noImplicitAny` | Disallows implicit `any` type inference |
| `strictFunctionTypes` | Stricter function parameter checking |
| `strictPropertyInitialization` | Class properties must be initialized |
| `strictBindCallApply` | Stricter `bind`, `call`, `apply` checking |
| `useUnknownInCatchVariables` | Catch variables are `unknown` not `any` |

```typescript
// strictNullChecks example
function getLength(str: string | null) {
  return str.length; // Error! str could be null
  return str?.length ?? 0; // Correct
}

// noImplicitAny example
function process(data) { // Error! 'data' implicitly has 'any' type
  return data;
}
function process(data: unknown) { // Correct
  return data;
}

// useUnknownInCatchVariables example
try {
  // ...
} catch (e) {
  console.log(e.message); // Error! 'e' is 'unknown'
  if (e instanceof Error) {
    console.log(e.message); // Correct
  }
}
```

> **Best practice:** Always start new projects with `"strict": true`. Disabling strict mode is technical debt that compounds quickly.

---

### Other Important Compiler Options

```json
{
  "compilerOptions": {
    // Emit
    "outDir": "./dist",            // Where to put compiled output
    "rootDir": "./src",            // Root of your source files
    "declaration": true,           // Generate .d.ts files
    "declarationMap": true,        // Source maps for .d.ts files
    "sourceMap": true,             // Generate .js.map files
    "removeComments": false,       // Keep comments in output

    // Strictness (beyond "strict")
    "noUncheckedIndexedAccess": true,  // arr[0] is T | undefined
    "exactOptionalPropertyTypes": true, // undefined !== missing
    "noImplicitReturns": true,     // Functions must always return
    "noFallthroughCasesInSwitch": true, // Switch cases must break

    // Module Resolution
    "esModuleInterop": true,       // Allows default imports from CJS
    "allowSyntheticDefaultImports": true, // Allows: import fs from 'fs'
    "resolveJsonModule": true,     // Import .json files
    "forceConsistentCasingInFileNames": true, // Prevents case bugs

    // Performance
    "skipLibCheck": true,          // Skip type checking of .d.ts files
    "incremental": true,           // Cache for faster rebuilds
    "tsBuildInfoFile": ".tsbuildinfo"
  }
}
```

#### `noUncheckedIndexedAccess` — Highly Recommended

This is not included in `strict` but is very valuable:

```typescript
// Without noUncheckedIndexedAccess:
const arr = [1, 2, 3];
const first: number = arr[0]; // Fine, but could be undefined at runtime!

// With noUncheckedIndexedAccess:
const first: number | undefined = arr[0]; // TypeScript forces you to handle it
if (first !== undefined) {
  console.log(first + 1); // Safe
}
```

---

### Full Recommended tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",

    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "exactOptionalPropertyTypes": true,
    "forceConsistentCasingInFileNames": true,

    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "skipLibCheck": true,
    "incremental": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

---

## 2. Project References and Monorepos

Project references allow you to split a TypeScript project into smaller sub-projects that can reference each other — essential for monorepos.

### Why Project References?

Without project references in a monorepo:
- TypeScript type-checks **everything together** — slow
- `tsc --watch` rebuilds the entire codebase on every change
- No incremental building between packages

With project references:
- Each package is compiled **independently**
- TypeScript caches results — only rebuilds changed packages
- Circular dependencies are detected at the project level

---

### Monorepo Structure

```
my-monorepo/
├── tsconfig.base.json          # Shared base config
├── tsconfig.json               # Root config with references
├── packages/
│   ├── core/
│   │   ├── src/
│   │   │   └── index.ts
│   │   └── tsconfig.json
│   ├── utils/
│   │   ├── src/
│   │   │   └── index.ts
│   │   └── tsconfig.json
│   └── app/
│       ├── src/
│       │   └── index.ts
│       └── tsconfig.json
└── package.json
```

---

### Step 1: Base Config (`tsconfig.base.json`)

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "composite": true
  }
}
```

> **`composite: true`** is required for any tsconfig that will be *referenced* by another. It enables incremental builds and generates the necessary `.tsbuildinfo` files.

---

### Step 2: Package Configs

**`packages/core/tsconfig.json`:**
```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "rootDir": "./src",
    "outDir": "./dist"
  },
  "include": ["src"]
}
```

**`packages/utils/tsconfig.json`:**
```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "rootDir": "./src",
    "outDir": "./dist"
  },
  "include": ["src"]
}
```

**`packages/app/tsconfig.json`** (references other packages):
```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "rootDir": "./src",
    "outDir": "./dist"
  },
  "include": ["src"],
  "references": [
    { "path": "../core" },
    { "path": "../utils" }
  ]
}
```

---

### Step 3: Root Config (`tsconfig.json`)

```json
{
  "files": [],
  "references": [
    { "path": "./packages/core" },
    { "path": "./packages/utils" },
    { "path": "./packages/app" }
  ]
}
```

> The root config has `"files": []` because it doesn't compile anything directly — it just orchestrates the references.

---

### Building the Monorepo

```bash
# Build all projects in dependency order
tsc --build

# Build a specific project (and its dependencies)
tsc --build packages/app

# Watch mode for all projects
tsc --build --watch

# Force rebuild everything (ignore cache)
tsc --build --force

# Clean all build artifacts
tsc --build --clean
```

---

## 3. Path Aliases and baseUrl

Path aliases eliminate deep relative imports and make your code much more readable.

### The Problem

```typescript
// Without path aliases — ugly and fragile
import { UserService } from "../../../services/UserService";
import { Logger } from "../../../../utils/Logger";
import { config } from "../../config";
```

### The Solution

```typescript
// With path aliases — clean and refactor-friendly
import { UserService } from "@/services/UserService";
import { Logger } from "@/utils/Logger";
import { config } from "@/config";
```

---

### Configuring Path Aliases

**`tsconfig.json`:**
```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@utils/*": ["src/utils/*"],
      "@types/*": ["src/types/*"],
      "@config": ["src/config/index.ts"]
    }
  }
}
```

**Usage:**
```typescript
import { Button } from "@components/Button";
import { formatDate } from "@utils/date";
import type { User } from "@types/user";
import { config } from "@config";
```

---

### `baseUrl` Explained

`baseUrl` sets the root directory for resolving **non-relative** module names. When set to `"."` (project root), you can import by absolute path:

```typescript
// baseUrl: "."
// This resolves to ./src/utils/logger.ts (from project root)
import { logger } from "src/utils/logger";
```

However, explicit `paths` aliases (using `@`) are cleaner and less ambiguous.

---

### Bundler Configuration for Path Aliases

> **Important:** TypeScript's `paths` only affect **type checking**. Your bundler or runtime also needs to be configured separately.

#### Vite

```typescript
// vite.config.ts
import { defineConfig } from "vite";
import { resolve } from "path";

export default defineConfig({
  resolve: {
    alias: {
      "@": resolve(__dirname, "./src"),
      "@components": resolve(__dirname, "./src/components"),
      "@utils": resolve(__dirname, "./src/utils"),
    },
  },
});
```

#### Node.js (with `tsconfig-paths`)

```bash
npm install --save-dev tsconfig-paths
```

```bash
# Run with ts-node
ts-node -r tsconfig-paths/register src/index.ts

# Or in package.json scripts:
# "start": "ts-node -r tsconfig-paths/register src/index.ts"
```

#### webpack

```javascript
// webpack.config.js
const path = require("path");

module.exports = {
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "src"),
    },
  },
};
```

#### Jest

```json
// jest.config.json
{
  "moduleNameMapper": {
    "^@/(.*)$": "<rootDir>/src/$1",
    "^@components/(.*)$": "<rootDir>/src/components/$1"
  }
}
```

---

## 4. Declaration Files (.d.ts)

Declaration files are TypeScript's way of describing the **types** of existing JavaScript code without including the implementation.

### What Are .d.ts Files?

```
// math.ts (TypeScript source)       // math.d.ts (generated declaration)
export function add(                  export declare function add(
  a: number,                            a: number,
  b: number                             b: number
): number {                           ): number;
  return a + b;
}
```

`.d.ts` files contain **only type information** — no runtime code. They allow TypeScript to understand the shape of code without seeing the original source.

---

### How .d.ts Files Are Generated

Enable declaration output in `tsconfig.json`:

```json
{
  "compilerOptions": {
    "declaration": true,        // Generate .d.ts files
    "declarationMap": true,     // Generate .d.ts.map for IDE "Go to Source"
    "declarationDir": "./types" // Optional: separate folder for .d.ts files
  }
}
```

After running `tsc`, for every `.ts` file you get:
- `dist/math.js` — compiled JavaScript
- `dist/math.d.ts` — type declarations
- `dist/math.d.ts.map` — source map linking .d.ts back to .ts

---

### Ambient Declarations

`.d.ts` files can also declare **global types** and **ambient modules** — things that exist at runtime but TypeScript doesn't know about.

```typescript
// global.d.ts — adds types to the global scope

// Declare a global variable injected at runtime (e.g., by webpack DefinePlugin)
declare const __APP_VERSION__: string;
declare const __DEV__: boolean;

// Declare a global interface (augments existing global)
interface Window {
  analytics: {
    track(event: string, properties?: Record<string, unknown>): void;
  };
}

// Declare a module for file types TypeScript can't understand
declare module "*.svg" {
  const content: string;
  export default content;
}

declare module "*.png" {
  const content: string;
  export default content;
}

declare module "*.css" {
  const styles: Record<string, string>;
  export default styles;
}
```

---

### Triple-Slash References

Triple-slash references are a legacy way to include other type files, still commonly seen:

```typescript
/// <reference types="node" />          // Includes @types/node
/// <reference types="vite/client" />   // Includes Vite client types
/// <reference path="./custom.d.ts" />  // Includes a specific file
```

Modern projects put these in `tsconfig.json` instead:

```json
{
  "compilerOptions": {
    "types": ["node", "jest"]
  }
}
```

---

## 5. Writing Your Own .d.ts for JS Libraries

When a JavaScript library has no TypeScript types (no `@types/` package), you can write your own declaration file.

### Scenario: A Library With No Types

```javascript
// some-math-lib/index.js (no types available)
exports.add = function(a, b) { return a + b; };
exports.multiply = function(a, b) { return a * b; };
exports.VERSION = "1.0.0";
```

---

### Writing the Declaration File

Create a `.d.ts` file in your project:

```typescript
// types/some-math-lib.d.ts

declare module "some-math-lib" {
  /**
   * Adds two numbers together.
   */
  export function add(a: number, b: number): number;

  /**
   * Multiplies two numbers.
   */
  export function multiply(a: number, b: number): number;

  /** Library version string */
  export const VERSION: string;
}
```

Tell TypeScript where to find it in `tsconfig.json`:

```json
{
  "compilerOptions": {
    "typeRoots": ["./types", "./node_modules/@types"]
  }
}
```

Or reference it with a triple-slash directive:

```typescript
// In any .ts file that uses the library
/// <reference path="../types/some-math-lib.d.ts" />
import { add, multiply } from "some-math-lib";
```

---

### Real-World Examples

#### Declaring a class-based library

```typescript
declare module "event-emitter-lib" {
  type Listener<T> = (data: T) => void;

  class EventEmitter<Events extends Record<string, unknown>> {
    on<K extends keyof Events>(event: K, listener: Listener<Events[K]>): this;
    off<K extends keyof Events>(event: K, listener: Listener<Events[K]>): this;
    emit<K extends keyof Events>(event: K, data: Events[K]): boolean;
  }

  export = EventEmitter;
}
```

#### Declaring a function with overloads

```typescript
declare module "parse-lib" {
  function parse(input: string): string[];
  function parse(input: string, options: { asObject: true }): Record<string, string>;
  function parse(input: string, options?: { asObject?: boolean }): string[] | Record<string, string>;

  export = parse;
}
```

#### Declaring a namespace (IIFE / UMD library)

```typescript
declare namespace MyGlobalLib {
  interface Options {
    timeout?: number;
    retries?: number;
  }

  function request(url: string, options?: Options): Promise<Response>;
  function get(url: string): Promise<Response>;
  function post(url: string, body: unknown): Promise<Response>;
}
```

---

### Module Augmentation

You can extend existing library types without modifying their source:

```typescript
// Augmenting Express's Request type to add a 'user' property
import "express";

declare module "express-serve-static-core" {
  interface Request {
    user?: {
      id: string;
      email: string;
      role: "admin" | "user";
    };
  }
}
```

```typescript
// Usage in an Express route handler
app.get("/profile", (req, res) => {
  if (!req.user) {
    return res.status(401).json({ error: "Unauthorized" });
  }
  res.json({ email: req.user.email }); // Fully typed!
});
```

---

## 6. ESLint with TypeScript

ESLint catches code quality issues beyond what TypeScript's type checker finds. The `@typescript-eslint` plugin adds TypeScript-aware rules.

### Installation

```bash
npm install --save-dev \
  eslint \
  @eslint/js \
  typescript-eslint \
  @typescript-eslint/eslint-plugin \
  @typescript-eslint/parser
```

---

### Modern Configuration (ESLint Flat Config)

**`eslint.config.mjs`** (ESLint v9+ flat config):

```javascript
import eslint from "@eslint/js";
import tseslint from "typescript-eslint";

export default tseslint.config(
  // Base ESLint recommended rules
  eslint.configs.recommended,

  // TypeScript recommended rules
  ...tseslint.configs.recommended,

  // TypeScript rules that require type information (slower but more powerful)
  ...tseslint.configs.recommendedTypeChecked,

  {
    languageOptions: {
      parserOptions: {
        project: true,           // Use the nearest tsconfig.json
        tsconfigRootDir: import.meta.dirname,
      },
    },
  },

  {
    // Your custom rules
    rules: {
      // Disable base rule, use TypeScript-aware version
      "no-unused-vars": "off",
      "@typescript-eslint/no-unused-vars": [
        "error",
        {
          argsIgnorePattern: "^_",
          varsIgnorePattern: "^_",
        },
      ],

      // Require explicit return types on exported functions
      "@typescript-eslint/explicit-module-boundary-types": "warn",

      // Disallow 'any'
      "@typescript-eslint/no-explicit-any": "error",

      // Require awaiting promises
      "@typescript-eslint/no-floating-promises": "error",

      // Enforce consistent type assertions
      "@typescript-eslint/consistent-type-assertions": [
        "error",
        { assertionStyle: "as" },
      ],

      // Enforce type imports
      "@typescript-eslint/consistent-type-imports": [
        "error",
        { prefer: "type-imports" },
      ],

      // Disallow non-null assertions (!)
      "@typescript-eslint/no-non-null-assertion": "warn",

      // Require proper nullish handling
      "@typescript-eslint/prefer-nullish-coalescing": "error",
      "@typescript-eslint/prefer-optional-chain": "error",
    },
  },

  {
    // Ignore patterns
    ignores: ["dist/**", "node_modules/**", "*.js", "*.mjs"],
  }
);
```

---

### Legacy Configuration (.eslintrc.json)

For projects still using ESLint v8:

```json
{
  "root": true,
  "parser": "@typescript-eslint/parser",
  "parserOptions": {
    "project": "./tsconfig.json",
    "ecmaVersion": "latest",
    "sourceType": "module"
  },
  "plugins": ["@typescript-eslint"],
  "extends": [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:@typescript-eslint/recommended-type-checked"
  ],
  "rules": {
    "no-unused-vars": "off",
    "@typescript-eslint/no-unused-vars": ["error", { "argsIgnorePattern": "^_" }],
    "@typescript-eslint/no-explicit-any": "error",
    "@typescript-eslint/no-floating-promises": "error",
    "@typescript-eslint/consistent-type-imports": ["error", { "prefer": "type-imports" }]
  },
  "ignorePatterns": ["dist", "node_modules"]
}
```

---

### Key @typescript-eslint Rules Explained

#### `no-floating-promises` — Prevents Unhandled Promises

```typescript
// ❌ BAD — promise result is ignored, errors are silently swallowed
async function saveUser(user: User) {
  db.save(user); // Error! This promise is not awaited or handled
}

// ✅ GOOD
async function saveUser(user: User) {
  await db.save(user);
}

// ✅ ALSO GOOD — explicit void for fire-and-forget
void analytics.track("user_saved");
```

#### `consistent-type-imports` — Type-Only Imports

```typescript
// ❌ BAD — value import used only for type
import { User } from "./types";

// ✅ GOOD — erased at compile time, better performance
import type { User } from "./types";
```

#### `no-misused-promises` — Prevents Passing Async Functions Where Sync Is Expected

```typescript
// ❌ BAD — forEach doesn't await the async callback
[1, 2, 3].forEach(async (num) => {
  await processNumber(num); // These run concurrently and errors are lost!
});

// ✅ GOOD
for (const num of [1, 2, 3]) {
  await processNumber(num);
}
// or:
await Promise.all([1, 2, 3].map(async (num) => processNumber(num)));
```

---

### Package.json Scripts

```json
{
  "scripts": {
    "lint": "eslint src",
    "lint:fix": "eslint src --fix",
    "lint:strict": "eslint src --max-warnings 0"
  }
}
```

---

## 7. Prettier + TypeScript

Prettier handles **code formatting** while ESLint handles **code quality**. They complement each other perfectly.

### Installation

```bash
npm install --save-dev prettier eslint-config-prettier
```

> `eslint-config-prettier` disables ESLint rules that would conflict with Prettier's formatting.

---

### Prettier Configuration

**`.prettierrc`:**
```json
{
  "semi": true,
  "singleQuote": false,
  "quoteProps": "as-needed",
  "trailingComma": "all",
  "tabWidth": 2,
  "useTabs": false,
  "printWidth": 100,
  "bracketSpacing": true,
  "arrowParens": "always",
  "endOfLine": "lf"
}
```

**`.prettierignore`:**
```
node_modules
dist
build
coverage
*.min.js
```

---

### Integrating Prettier with ESLint

Add `eslint-config-prettier` **last** in your extends array to override any formatting rules:

```json
{
  "extends": [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "prettier"
  ]
}
```

In flat config (`eslint.config.mjs`):

```javascript
import eslint from "@eslint/js";
import tseslint from "typescript-eslint";
import prettier from "eslint-config-prettier";

export default tseslint.config(
  eslint.configs.recommended,
  ...tseslint.configs.recommended,
  prettier, // Must be last!
);
```

---

### Package.json Scripts

```json
{
  "scripts": {
    "format": "prettier --write \"src/**/*.{ts,tsx,json,md}\"",
    "format:check": "prettier --check \"src/**/*.{ts,tsx,json,md}\"",
    "lint": "eslint src",
    "lint:fix": "eslint src --fix"
  }
}
```

---

### VS Code Integration

**`.vscode/settings.json`:**
```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },
  "[typescript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[typescriptreact]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  }
}
```

---

### Recommended VS Code Extensions

```json
// .vscode/extensions.json
{
  "recommendations": [
    "esbenp.prettier-vscode",
    "dbaeumer.vscode-eslint",
    "ms-vscode.vscode-typescript-next"
  ]
}
```

---

### Git Hooks with Husky + lint-staged

Automatically format and lint only staged files before each commit:

```bash
npm install --save-dev husky lint-staged
npx husky init
```

**`.husky/pre-commit`:**
```bash
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"
npx lint-staged
```

**`package.json`:**
```json
{
  "lint-staged": {
    "*.{ts,tsx}": [
      "prettier --write",
      "eslint --fix",
      "eslint --max-warnings 0"
    ],
    "*.{json,md,css}": [
      "prettier --write"
    ]
  }
}
```

---

## 8. ts-prune and Type Coverage Tools

Dead code and poor type coverage silently degrade your codebase. These tools help you find and fix both.

### ts-prune — Finding Unused Exports

`ts-prune` finds exports that are never imported anywhere in your project.

```bash
npm install --save-dev ts-prune
```

**`package.json`:**
```json
{
  "scripts": {
    "prune": "ts-prune",
    "prune:error": "ts-prune | grep -v '(used in module)' | wc -l | xargs -I{} test {} -eq 0"
  }
}
```

**Example output:**
```
src/utils/oldHelper.ts:5 - formatDate
src/types/legacy.ts:12 - LegacyUser
src/services/deprecated.ts:3 - fetchOldData (used in module)
```

Lines marked `(used in module)` are self-referential imports — often fine. The others are true dead code.

**Configuration (`.ts-prunerc`):**
```json
{
  "ignore": "(test|spec|stories|index)\\.ts$"
}
```

---

### type-coverage — Measuring Your Type Safety

`type-coverage` measures what percentage of your TypeScript code is fully typed (not `any`).

```bash
npm install --save-dev type-coverage
```

```bash
# Check type coverage
npx type-coverage

# Show specific locations of 'any' usage
npx type-coverage --detail

# Fail if coverage drops below a threshold
npx type-coverage --at-least 95

# Ignore specific patterns
npx type-coverage --ignore-catch --ignore-files "**/*.test.ts"
```

**Example output:**
```
14445 / 14982 98.41%
```

**`package.json` script:**
```json
{
  "scripts": {
    "type-coverage": "type-coverage --at-least 95 --strict"
  }
}
```

---

### TypeScript's Built-in `noUnusedLocals` and `noUnusedParameters`

These built-in flags catch unused variables and parameters at compile time:

```json
{
  "compilerOptions": {
    "noUnusedLocals": true,
    "noUnusedParameters": true
  }
}
```

```typescript
// noUnusedLocals: catches this
function process() {
  const unusedVar = "hello"; // Error: 'unusedVar' is declared but never read
  return 42;
}

// noUnusedParameters: catches this
function greet(name: string, _title: string) { // OK — prefixed with _
  return `Hello, ${name}`;
  //      ^ Error without the underscore prefix
}
```

> **Convention:** Prefix intentionally unused parameters with `_` to suppress the warning.

---

### knip — The All-in-One Dead Code Finder

`knip` is a more powerful, modern alternative to `ts-prune` that finds unused files, exports, dependencies, and more:

```bash
npm install --save-dev knip
```

**`knip.json`:**
```json
{
  "entry": ["src/index.ts"],
  "project": ["src/**/*.ts"],
  "ignore": ["src/**/*.test.ts"],
  "ignoreDependencies": ["ts-node"]
}
```

```bash
# Run knip
npx knip

# Output:
# Unused files (2)
#   src/utils/oldFormatter.ts
#   src/types/legacyTypes.ts
#
# Unused exports (3)
#   src/utils/date.ts: formatRelativeDate
#   src/services/api.ts: deprecatedFetch
#
# Unused dependencies (1)
#   lodash
```

---

### Putting It All Together — CI Pipeline

**`.github/workflows/quality.yml`:**
```yaml
name: Code Quality

on: [push, pull_request]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - run: npm ci

      - name: TypeScript type check
        run: npx tsc --noEmit

      - name: ESLint
        run: npm run lint:strict

      - name: Prettier
        run: npm run format:check

      - name: Type coverage
        run: npx type-coverage --at-least 95

      - name: Dead code check
        run: npx knip
```

---

## Summary

Here's a quick reference for everything covered this week:

| Tool / Feature | Purpose | Key Config |
|---------------|---------|-----------|
| `tsconfig.json` | TypeScript compiler settings | `strict`, `target`, `module`, `lib` |
| Project References | Monorepo incremental builds | `composite: true`, `references: []` |
| Path Aliases | Clean imports | `baseUrl`, `paths` in tsconfig + bundler config |
| `.d.ts` files | Type declarations for JS | `declaration: true` or handwritten |
| Module Augmentation | Extend library types | `declare module "..."` |
| `@typescript-eslint` | Type-aware linting | `parserOptions.project`, rules |
| Prettier | Code formatting | `.prettierrc`, `eslint-config-prettier` |
| `ts-prune` / `knip` | Unused export detection | CI integration |
| `type-coverage` | Measure type safety % | `--at-least 95` |

---

## Practice Exercises

1. **tsconfig audit**: Take an existing project and add `noUncheckedIndexedAccess` and `exactOptionalPropertyTypes`. Fix all the resulting errors.

2. **Monorepo setup**: Create a monorepo with three packages — `core` (shared types), `api` (backend), and `web` (frontend). Set up project references so `api` and `web` both reference `core`.

3. **Write a .d.ts file**: Find a JavaScript utility library on npm that has no TypeScript types. Write a complete `.d.ts` declaration file for it.

4. **ESLint rule exploration**: Enable `@typescript-eslint/no-floating-promises` and `@typescript-eslint/no-misused-promises` on a real codebase. Document every bug you find.

5. **Type coverage goal**: Run `type-coverage` on a project. If it's below 90%, bring it to 95% by eliminating `any` types systematically.

---

*Next Week: Advanced TypeScript Patterns — Conditional Types, Template Literal Types, and Type-Level Programming*
