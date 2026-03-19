# Week 8: React Ecosystem Typing

> **Prerequisites:** Familiarity with TypeScript generics, React hooks, and basic Redux/router concepts.  
> **Goal:** Type every layer of a modern React stack — routing, state management, styling, and environment configuration.

---

## Table of Contents

1. [Typing React Router v6](#1-typing-react-router-v6)
2. [Typing Redux Toolkit Slices and Thunks](#2-typing-redux-toolkit-slices-and-thunks)
3. [Typing Zustand Stores](#3-typing-zustand-stores)
4. [Declaration Files for Untyped Packages](#4-declaration-files-for-untyped-packages)
5. [Module Augmentation](#5-module-augmentation)
6. [Typing CSS Modules](#6-typing-css-modules)
7. [Typing styled-components with Props](#7-typing-styled-components-with-props)
8. [Environment Variable Typing](#8-environment-variable-typing-importmetaenv)
9. [Putting It All Together](#9-putting-it-all-together)
10. [Exercises](#10-exercises)

---

## 1. Typing React Router v6

React Router v6 ships with its own TypeScript definitions. The main areas to type are **URL params**, **loader/action data**, and **search params**.

### 1.1 Typing Route Params with `useParams`

`useParams` returns `Readonly<Params<Key>>` where keys are strings. Without explicit typing, every param is `string | undefined`.

```tsx
// ❌ Untyped — TypeScript doesn't know what params exist
import { useParams } from "react-router-dom";

function ProductPage() {
  const { id } = useParams(); // id: string | undefined
  // Calling id.toUpperCase() would require a non-null assertion
}
```

```tsx
// ✅ Typed — pass a record of expected param names
import { useParams } from "react-router-dom";

type ProductParams = {
  id: string;
  category: string;
};

function ProductPage() {
  const { id, category } = useParams<ProductParams>();
  // id: string | undefined  (Router still marks all params optional)
  // Use optional chaining or runtime guards
  if (!id) return <div>No product ID</div>;
  return <div>Product {id} in {category}</div>;
}
```

> **Note:** Even with a typed generic, React Router marks every param as `string | undefined` because the component could theoretically be rendered outside of a matching route.

### 1.2 Typing Loaders (Data Routers)

React Router v6.4+ introduced data APIs. Loaders run on the server or in the data router before a component renders.

```tsx
// routes/product.tsx
import { LoaderFunctionArgs, useLoaderData } from "react-router-dom";

// Define the shape of the loaded data
type ProductLoaderData = {
  id: string;
  name: string;
  price: number;
};

// Type the loader function itself
export async function productLoader({
  params,
}: LoaderFunctionArgs): Promise<ProductLoaderData> {
  const { id } = params;
  if (!id) throw new Response("Not Found", { status: 404 });

  const res = await fetch(`/api/products/${id}`);
  if (!res.ok) throw new Response("Not Found", { status: 404 });

  const product: ProductLoaderData = await res.json();
  return product;
}

// The component — useLoaderData needs a type assertion
export function ProductPage() {
  // useLoaderData() returns unknown in v6 — cast it
  const product = useLoaderData() as ProductLoaderData;

  return (
    <div>
      <h1>{product.name}</h1>
      <p>${product.price.toFixed(2)}</p>
    </div>
  );
}
```

**Router setup:**

```tsx
// router.tsx
import { createBrowserRouter } from "react-router-dom";
import { ProductPage, productLoader } from "./routes/product";

export const router = createBrowserRouter([
  {
    path: "/products/:id",
    element: <ProductPage />,
    loader: productLoader,
  },
]);
```

### 1.3 Typing Actions

Actions handle form submissions in data routers.

```tsx
import { ActionFunctionArgs, redirect } from "react-router-dom";

type CreateProductInput = {
  name: string;
  price: string; // Form data is always string
};

export async function createProductAction({
  request,
}: ActionFunctionArgs): Promise<Response> {
  const formData = await request.formData();

  const input: CreateProductInput = {
    name: formData.get("name") as string,
    price: formData.get("price") as string,
  };

  const res = await fetch("/api/products", {
    method: "POST",
    body: JSON.stringify({ ...input, price: parseFloat(input.price) }),
    headers: { "Content-Type": "application/json" },
  });

  const { id } = await res.json();
  return redirect(`/products/${id}`);
}
```

### 1.4 Typing `useNavigate` and `Link`

```tsx
import { useNavigate, Link } from "react-router-dom";

function BackButton() {
  const navigate = useNavigate();

  // navigate accepts a number (go back/forward) or a path
  const goBack = () => navigate(-1);
  const goHome = () => navigate("/", { replace: true });

  return (
    <>
      <button onClick={goBack}>Back</button>
      {/* Link's `to` prop is typed as To (string | Partial<Path>) */}
      <Link to="/home" state={{ from: "product" }}>Home</Link>
    </>
  );
}
```

### 1.5 Typed Search Params

```tsx
import { useSearchParams } from "react-router-dom";

type FilterParams = {
  category: string;
  page: string;
  sort: "asc" | "desc";
};

function ProductList() {
  const [searchParams, setSearchParams] = useSearchParams();

  // Read typed values
  const category = searchParams.get("category") ?? "all";
  const page = Number(searchParams.get("page") ?? "1");

  const updateFilter = (key: keyof FilterParams, value: string) => {
    setSearchParams((prev) => {
      prev.set(key, value);
      return prev;
    });
  };

  return (
    <button onClick={() => updateFilter("sort", "asc")}>Sort Asc</button>
  );
}
```

---

## 2. Typing Redux Toolkit Slices and Thunks

Redux Toolkit (RTK) dramatically reduces Redux boilerplate and ships first-class TypeScript support.

### 2.1 Setting Up Typed Hooks

The first step is creating typed versions of `useDispatch` and `useSelector`. Do this once per project.

```ts
// store/index.ts
import { configureStore } from "@reduxjs/toolkit";
import { TypedUseSelectorHook, useDispatch, useSelector } from "react-redux";
import cartReducer from "./cartSlice";
import userReducer from "./userSlice";

export const store = configureStore({
  reducer: {
    cart: cartReducer,
    user: userReducer,
  },
});

// Infer RootState and AppDispatch from the store itself
export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;

// Typed hooks — always use these instead of plain useDispatch/useSelector
export const useAppDispatch: () => AppDispatch = useDispatch;
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
```

### 2.2 Typing a Slice

```ts
// store/cartSlice.ts
import { createSlice, PayloadAction } from "@reduxjs/toolkit";

type CartItem = {
  id: string;
  name: string;
  price: number;
  quantity: number;
};

type CartState = {
  items: CartItem[];
  totalAmount: number;
  isOpen: boolean;
};

const initialState: CartState = {
  items: [],
  totalAmount: 0,
  isOpen: false,
};

const cartSlice = createSlice({
  name: "cart",
  initialState,
  reducers: {
    // PayloadAction<T> types the action's payload
    addItem(state, action: PayloadAction<CartItem>) {
      const existing = state.items.find((i) => i.id === action.payload.id);
      if (existing) {
        existing.quantity += action.payload.quantity;
      } else {
        state.items.push(action.payload);
      }
      state.totalAmount += action.payload.price * action.payload.quantity;
    },

    removeItem(state, action: PayloadAction<string>) {
      const item = state.items.find((i) => i.id === action.payload);
      if (item) {
        state.totalAmount -= item.price * item.quantity;
        state.items = state.items.filter((i) => i.id !== action.payload);
      }
    },

    updateQuantity(
      state,
      action: PayloadAction<{ id: string; quantity: number }>
    ) {
      const item = state.items.find((i) => i.id === action.payload.id);
      if (item) {
        state.totalAmount +=
          item.price * (action.payload.quantity - item.quantity);
        item.quantity = action.payload.quantity;
      }
    },

    toggleCart(state) {
      state.isOpen = !state.isOpen;
    },

    clearCart() {
      // Returning a new state is an alternative to mutating
      return initialState;
    },
  },
});

export const { addItem, removeItem, updateQuantity, toggleCart, clearCart } =
  cartSlice.actions;
export default cartSlice.reducer;
```

### 2.3 Typing Async Thunks with `createAsyncThunk`

```ts
// store/userSlice.ts
import { createAsyncThunk, createSlice, PayloadAction } from "@reduxjs/toolkit";
import type { RootState } from "./index";

type User = {
  id: string;
  name: string;
  email: string;
  role: "admin" | "user";
};

type UserState = {
  currentUser: User | null;
  status: "idle" | "loading" | "succeeded" | "failed";
  error: string | null;
};

// createAsyncThunk<ReturnType, ArgType, ThunkApiConfig>
export const fetchUser = createAsyncThunk<
  User,          // The type returned by the thunk's payload creator
  string,        // The type of the first argument (userId)
  {
    rejectValue: string;   // The type passed to rejectWithValue
    state: RootState;      // Lets you access getState() with full types
  }
>(
  "user/fetchUser",
  async (userId, { rejectWithValue, getState }) => {
    // Access typed state if needed
    const currentUser = getState().user.currentUser;
    if (currentUser?.id === userId) {
      return currentUser; // Already fetched
    }

    try {
      const res = await fetch(`/api/users/${userId}`);
      if (!res.ok) {
        return rejectWithValue(`HTTP error: ${res.status}`);
      }
      return (await res.json()) as User;
    } catch (err) {
      return rejectWithValue("Network error");
    }
  }
);

const initialState: UserState = {
  currentUser: null,
  status: "idle",
  error: null,
};

const userSlice = createSlice({
  name: "user",
  initialState,
  reducers: {
    logout(state) {
      state.currentUser = null;
      state.status = "idle";
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchUser.pending, (state) => {
        state.status = "loading";
        state.error = null;
      })
      .addCase(fetchUser.fulfilled, (state, action) => {
        // action.payload is typed as User ✅
        state.status = "succeeded";
        state.currentUser = action.payload;
      })
      .addCase(fetchUser.rejected, (state, action) => {
        state.status = "failed";
        // action.payload is typed as string | undefined (our rejectValue type)
        state.error = action.payload ?? "Unknown error";
      });
  },
});

export const { logout } = userSlice.actions;
export default userSlice.reducer;
```

### 2.4 Using Typed Selectors

```ts
// store/selectors.ts
import type { RootState } from "./index";

// Simple selectors
export const selectCartItems = (state: RootState) => state.cart.items;
export const selectCartTotal = (state: RootState) => state.cart.totalAmount;
export const selectCurrentUser = (state: RootState) => state.user.currentUser;

// Derived selectors (memoized with createSelector from reselect)
import { createSelector } from "@reduxjs/toolkit";

export const selectCartItemCount = createSelector(
  selectCartItems,
  (items) => items.reduce((total, item) => total + item.quantity, 0)
);
```

### 2.5 Using Typed Hooks in Components

```tsx
// components/Cart.tsx
import { useAppDispatch, useAppSelector } from "../store";
import { selectCartItems, selectCartTotal } from "../store/selectors";
import { removeItem, clearCart } from "../store/cartSlice";
import { fetchUser } from "../store/userSlice";

function Cart() {
  const dispatch = useAppDispatch();
  const items = useAppSelector(selectCartItems);
  const total = useAppSelector(selectCartTotal);

  const handleRemove = (id: string) => {
    dispatch(removeItem(id)); // Fully typed
  };

  const handleFetchUser = async () => {
    // unwrap() extracts the fulfilled value or throws the rejected value
    try {
      const user = await dispatch(fetchUser("user-123")).unwrap();
      console.log("Fetched:", user.name); // user is typed as User ✅
    } catch (err) {
      console.error("Failed:", err); // err is string (our rejectValue)
    }
  };

  return (
    <div>
      {items.map((item) => (
        <div key={item.id}>
          <span>{item.name}</span>
          <button onClick={() => handleRemove(item.id)}>Remove</button>
        </div>
      ))}
      <p>Total: ${total.toFixed(2)}</p>
      <button onClick={() => dispatch(clearCart())}>Clear</button>
    </div>
  );
}
```

---

## 3. Typing Zustand Stores

Zustand is a lightweight state management library. TypeScript support is achieved by typing the store's state and actions together.

### 3.1 Basic Typed Store

```ts
// stores/counterStore.ts
import { create } from "zustand";

type CounterState = {
  count: number;
  increment: () => void;
  decrement: () => void;
  reset: () => void;
  incrementBy: (amount: number) => void;
};

export const useCounterStore = create<CounterState>((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
  reset: () => set({ count: 0 }),
  incrementBy: (amount) => set((state) => ({ count: state.count + amount })),
}));
```

```tsx
// Usage in component
import { useCounterStore } from "../stores/counterStore";

function Counter() {
  const { count, increment, decrement } = useCounterStore();
  // All properties are fully typed ✅

  return (
    <div>
      <button onClick={decrement}>-</button>
      <span>{count}</span>
      <button onClick={increment}>+</button>
    </div>
  );
}
```

### 3.2 Splitting State and Actions

For larger stores, it's useful to separate state types from action types:

```ts
// stores/authStore.ts
import { create } from "zustand";

type User = {
  id: string;
  email: string;
  role: "admin" | "user";
};

// Separate types for clarity
type AuthState = {
  user: User | null;
  token: string | null;
  isAuthenticated: boolean;
};

type AuthActions = {
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
  updateUser: (updates: Partial<User>) => void;
};

type AuthStore = AuthState & AuthActions;

const initialState: AuthState = {
  user: null,
  token: null,
  isAuthenticated: false,
};

export const useAuthStore = create<AuthStore>((set, get) => ({
  ...initialState,

  login: async (email, password) => {
    const res = await fetch("/api/auth/login", {
      method: "POST",
      body: JSON.stringify({ email, password }),
      headers: { "Content-Type": "application/json" },
    });
    const { user, token } = await res.json();
    set({ user, token, isAuthenticated: true });
  },

  logout: () => {
    set(initialState);
  },

  updateUser: (updates) => {
    const currentUser = get().user;
    if (!currentUser) return;
    set({ user: { ...currentUser, ...updates } });
  },
}));
```

### 3.3 Zustand with Middleware

Typing gets slightly more complex when middleware like `persist` or `devtools` is involved.

```ts
// stores/settingsStore.ts
import { create } from "zustand";
import { persist, createJSONStorage } from "zustand/middleware";

type Theme = "light" | "dark" | "system";
type Language = "en" | "fr" | "es" | "de";

type SettingsState = {
  theme: Theme;
  language: Language;
  notifications: boolean;
  setTheme: (theme: Theme) => void;
  setLanguage: (lang: Language) => void;
  toggleNotifications: () => void;
};

export const useSettingsStore = create<SettingsState>()(
  persist(
    (set) => ({
      theme: "system",
      language: "en",
      notifications: true,

      setTheme: (theme) => set({ theme }),
      setLanguage: (language) => set({ language }),
      toggleNotifications: () =>
        set((state) => ({ notifications: !state.notifications })),
    }),
    {
      name: "app-settings",
      storage: createJSONStorage(() => localStorage),
      // Only persist specific keys
      partialize: (state) => ({
        theme: state.theme,
        language: state.language,
      }),
    }
  )
);
```

### 3.4 Subscribing Outside Components

```ts
// Subscribe to store changes outside React (e.g., in services)
import { useAuthStore } from "./authStore";

// getState() is typed with your full store type
const { token } = useAuthStore.getState();

// Subscribe to changes
const unsubscribe = useAuthStore.subscribe((state) => {
  if (!state.isAuthenticated) {
    // Redirect to login
    window.location.href = "/login";
  }
});
```

### 3.5 Slices Pattern for Large Stores

```ts
// stores/slices/cartSlice.ts
import type { StateCreator } from "zustand";

type CartItem = { id: string; quantity: number; price: number };

export type CartSlice = {
  cartItems: CartItem[];
  addToCart: (item: CartItem) => void;
  removeFromCart: (id: string) => void;
};

export const createCartSlice: StateCreator<CartSlice> = (set) => ({
  cartItems: [],
  addToCart: (item) =>
    set((state) => ({ cartItems: [...state.cartItems, item] })),
  removeFromCart: (id) =>
    set((state) => ({
      cartItems: state.cartItems.filter((i) => i.id !== id),
    })),
});

// stores/useStore.ts — Combine slices
import { create } from "zustand";
import { createCartSlice, CartSlice } from "./slices/cartSlice";
import { createAuthSlice, AuthSlice } from "./slices/authSlice";

type AppStore = CartSlice & AuthSlice;

export const useStore = create<AppStore>()((...args) => ({
  ...createCartSlice(...args),
  ...createAuthSlice(...args),
}));
```

---

## 4. Declaration Files for Untyped Packages (`@types/`)

When a package doesn't ship its own types, you have two options: install a community `@types/` package, or write your own declaration file.

### 4.1 Installing Community Types

```bash
# Check if @types exists for a package
npm install --save-dev @types/lodash
npm install --save-dev @types/uuid
npm install --save-dev @types/node
```

After installing, TypeScript automatically picks up the types — no configuration needed.

### 4.2 Writing Your Own Declaration File

When no `@types/` package exists, create a `.d.ts` file to describe the module's shape.

**Scenario:** `some-legacy-lib` exports a class and a few utilities, but has no types.

```ts
// src/types/some-legacy-lib.d.ts

declare module "some-legacy-lib" {
  // Type the constructor and instance methods
  export class DataProcessor {
    constructor(options?: ProcessorOptions);
    process(data: unknown[]): ProcessedResult;
    reset(): void;
  }

  export interface ProcessorOptions {
    batchSize?: number;
    timeout?: number;
    onError?: (err: Error) => void;
  }

  export interface ProcessedResult {
    success: boolean;
    count: number;
    errors: Error[];
  }

  // Type utility functions
  export function formatData(input: unknown): string;
  export function validateSchema(data: unknown, schema: object): boolean;

  // Type a default export
  const processor: DataProcessor;
  export default processor;
}
```

### 4.3 Typing CommonJS Modules with Default Exports

Some older packages use `module.exports = ...` (CommonJS). TypeScript needs special handling:

```ts
// src/types/legacy-csv-parser.d.ts

declare module "legacy-csv-parser" {
  interface ParseOptions {
    delimiter?: string;
    headers?: boolean;
    encoding?: BufferEncoding;
  }

  type ParsedRow = Record<string, string>;

  // For module.exports = function() {...}
  function parseCsv(input: string, options?: ParseOptions): ParsedRow[];

  // This is required for CommonJS interop
  export = parseCsv;
}
```

```ts
// Usage with esModuleInterop: true in tsconfig
import parseCsv from "legacy-csv-parser";
```

### 4.4 Typing Packages That Augment Globals

Some packages (like polyfills or test utilities) extend global types:

```ts
// src/types/globals.d.ts

// Extend the Window interface
interface Window {
  analytics: {
    track: (event: string, properties?: Record<string, unknown>) => void;
    identify: (userId: string, traits?: Record<string, unknown>) => void;
  };
  __APP_VERSION__: string;
}

// Extend NodeJS.ProcessEnv
declare namespace NodeJS {
  interface ProcessEnv {
    NODE_ENV: "development" | "production" | "test";
    REACT_APP_API_URL: string;
    REACT_APP_FEATURE_FLAGS?: string;
  }
}
```

### 4.5 Configuring TypeScript to Find Your Declaration Files

Ensure your `tsconfig.json` includes your `types` directory:

```json
{
  "compilerOptions": {
    "typeRoots": ["./node_modules/@types", "./src/types"],
    "types": []
  },
  "include": ["src"]
}
```

---

## 5. Module Augmentation

Module augmentation lets you extend existing types from third-party libraries **without modifying their source**. This is the correct way to add properties to express `Request`, React component props, etc.

### 5.1 Augmenting a Third-Party Module

**Example: Adding custom properties to React Router's location state:**

```ts
// src/types/react-router-dom.d.ts

// IMPORTANT: Must be in a module (has import/export) to augment
import "react-router-dom";

declare module "react-router-dom" {
  // Augment the existing interface
  interface Location {
    state: {
      from?: string;
      referrer?: string;
      breadcrumbs?: string[];
    } | null;
  }
}
```

```tsx
// Now location.state is typed
import { useLocation } from "react-router-dom";

function Page() {
  const location = useLocation();
  const from = location.state?.from; // typed as string | undefined ✅
  return <div>From: {from}</div>;
}
```

### 5.2 Augmenting Express Request

```ts
// src/types/express.d.ts
import "express";

declare module "express" {
  interface Request {
    user?: {
      id: string;
      email: string;
      role: "admin" | "user";
    };
    requestId?: string;
  }
}
```

```ts
// middleware/auth.ts
import { Request, Response, NextFunction } from "express";

export function authMiddleware(req: Request, res: Response, next: NextFunction) {
  // req.user is now a recognized property with proper types ✅
  req.user = { id: "123", email: "test@example.com", role: "user" };
  next();
}
```

### 5.3 Augmenting styled-components DefaultTheme

This pattern is covered in more detail in Section 7, but the core idea:

```ts
// src/types/styled.d.ts
import "styled-components";

declare module "styled-components" {
  export interface DefaultTheme {
    colors: {
      primary: string;
      secondary: string;
      background: string;
      text: string;
    };
    spacing: (multiplier: number) => string;
    breakpoints: {
      mobile: string;
      tablet: string;
      desktop: string;
    };
  }
}
```

### 5.4 Augmenting Global Interfaces

```ts
// src/types/global.d.ts

// Augment built-in Array (use sparingly!)
interface Array<T> {
  last(): T | undefined;
}

// Augment String prototype
interface String {
  toTitleCase(): string;
}
```

### 5.5 Re-exporting with Augmentation

Sometimes you want to re-export an augmented module:

```ts
// src/router.ts — Augment and re-export
import {
  useNavigate as useNavigateOriginal,
  NavigateOptions,
} from "react-router-dom";

type TypedRoutes = "/home" | "/products" | "/cart" | "/profile";

// Wrap with stricter types
export function useNavigate() {
  const navigate = useNavigateOriginal();
  return (to: TypedRoutes | number, options?: NavigateOptions) => {
    navigate(to as string, options);
  };
}
```

---

## 6. Typing CSS Modules

CSS Modules scope styles locally by default. TypeScript needs to know the shape of the classes object.

### 6.1 The Problem

```tsx
// ❌ Without typing — TypeScript error
import styles from "./Button.module.css";

// Element implicitly has an 'any' type
const cls = styles.container; // TypeScript doesn't know what classes exist
```

### 6.2 Solution 1: Global CSS Module Declaration

The quickest fix — tells TypeScript to treat all `.module.css` files as `Record<string, string>`:

```ts
// src/types/css-modules.d.ts
declare module "*.module.css" {
  const classes: Record<string, string>;
  export default classes;
}

declare module "*.module.scss" {
  const classes: Record<string, string>;
  export default classes;
}

declare module "*.module.sass" {
  const classes: Record<string, string>;
  export default classes;
}
```

This removes the error but provides no autocompletion for class names.

### 6.3 Solution 2: `typescript-plugin-css-modules`

Install the plugin for IDE-level class name autocompletion:

```bash
npm install --save-dev typescript-plugin-css-modules
```

```json
// tsconfig.json
{
  "compilerOptions": {
    "plugins": [
      {
        "name": "typescript-plugin-css-modules"
      }
    ]
  }
}
```

> **Note:** This is a language service plugin (IDE only). Add `.vscode/settings.json`:

```json
{
  "typescript.tsdk": "node_modules/typescript/lib",
  "typescript.enablePromptUseWorkspaceTsdk": true
}
```

### 6.4 Solution 3: `typed-css-modules` (Generated Types)

Generate a `.d.ts` file for each CSS Module at build time:

```bash
npm install --save-dev typed-css-modules

# Generate types for all CSS modules
npx tcm src --pattern "**/*.module.css"
```

This creates a `Button.module.css.d.ts`:

```ts
// Button.module.css.d.ts (auto-generated — don't edit manually)
declare const styles: {
  readonly container: string;
  readonly label: string;
  readonly disabled: string;
  readonly primary: string;
  readonly secondary: string;
};
export = styles;
```

Add it to your `package.json` scripts:

```json
{
  "scripts": {
    "types:css": "tcm src --pattern '**/*.module.css' --watch"
  }
}
```

### 6.5 Using CSS Modules with `clsx` / `classnames`

```tsx
// components/Button.module.css
/*
.button { ... }
.primary { ... }
.secondary { ... }
.disabled { ... }
.large { ... }
*/

// components/Button.tsx
import clsx from "clsx";
import styles from "./Button.module.css";

type ButtonProps = {
  variant?: "primary" | "secondary";
  size?: "small" | "medium" | "large";
  disabled?: boolean;
  onClick?: () => void;
  children: React.ReactNode;
};

function Button({
  variant = "primary",
  size = "medium",
  disabled = false,
  onClick,
  children,
}: ButtonProps) {
  return (
    <button
      className={clsx(
        styles.button,
        styles[variant],        // Computed access — works if typings are correct
        size === "large" && styles.large,
        disabled && styles.disabled
      )}
      disabled={disabled}
      onClick={onClick}
    >
      {children}
    </button>
  );
}
```

### 6.6 CSS Module Composition Types

```ts
// When using CSS composes:
// .button { composes: base from './base.module.css'; }
// The composed class is merged at runtime, types just see the string
```

---

## 7. Typing styled-components with Props

styled-components supports generics for typing component props passed to template literals.

### 7.1 Basic Prop Typing

```tsx
import styled from "styled-components";

// ❌ Without typing — props are unknown
const Box = styled.div`
  background: ${(props) => props.color}; // Error: 'color' doesn't exist
`;

// ✅ With typing — pass a generic type argument
type BoxProps = {
  color?: string;
  padding?: number;
};

const Box = styled.div<BoxProps>`
  background: ${({ color }) => color ?? "white"};
  padding: ${({ padding }) => padding ?? 0}px;
`;

// Usage
<Box color="blue" padding={16}>Content</Box>
```

### 7.2 Theming with `DefaultTheme`

First, define and augment the theme type (see Section 5.3), then create the theme object:

```tsx
// theme.ts
import { DefaultTheme } from "styled-components";

export const lightTheme: DefaultTheme = {
  colors: {
    primary: "#0070f3",
    secondary: "#7928ca",
    background: "#ffffff",
    text: "#000000",
  },
  spacing: (multiplier: number) => `${multiplier * 8}px`,
  breakpoints: {
    mobile: "576px",
    tablet: "768px",
    desktop: "1200px",
  },
};

export const darkTheme: DefaultTheme = {
  colors: {
    primary: "#0070f3",
    secondary: "#7928ca",
    background: "#1a1a1a",
    text: "#ffffff",
  },
  spacing: (multiplier: number) => `${multiplier * 8}px`,
  breakpoints: {
    mobile: "576px",
    tablet: "768px",
    desktop: "1200px",
  },
};
```

```tsx
// App.tsx
import { ThemeProvider } from "styled-components";
import { lightTheme } from "./theme";

function App() {
  return (
    <ThemeProvider theme={lightTheme}>
      <Layout />
    </ThemeProvider>
  );
}
```

### 7.3 Using Theme in Styled Components

```tsx
// components/styled.ts
import styled from "styled-components";

// Theme is automatically available through DefaultTheme augmentation
export const Container = styled.div`
  max-width: 1200px;
  margin: 0 auto;
  padding: ${({ theme }) => theme.spacing(2)};

  @media (max-width: ${({ theme }) => theme.breakpoints.tablet}) {
    padding: ${({ theme }) => theme.spacing(1)};
  }
`;

export const Heading = styled.h1`
  color: ${({ theme }) => theme.colors.text};
  font-size: 2rem;
`;

export const PrimaryButton = styled.button`
  background: ${({ theme }) => theme.colors.primary};
  color: white;
  border: none;
  padding: ${({ theme }) => theme.spacing(1)} ${({ theme }) => theme.spacing(2)};
  border-radius: 4px;
  cursor: pointer;

  &:hover {
    opacity: 0.9;
  }
`;
```

### 7.4 Conditional Styles with Props and Theme

```tsx
type AlertProps = {
  variant: "success" | "warning" | "error" | "info";
  dismissible?: boolean;
};

const alertColors = {
  success: "#d4edda",
  warning: "#fff3cd",
  error: "#f8d7da",
  info: "#d1ecf1",
};

const Alert = styled.div<AlertProps>`
  padding: ${({ theme }) => theme.spacing(2)};
  background: ${({ variant }) => alertColors[variant]};
  border-radius: 4px;
  position: relative;
  display: flex;
  align-items: center;
  justify-content: space-between;

  ${({ dismissible }) =>
    dismissible &&
    `
    padding-right: 40px;
  `}
`;

// Usage
<Alert variant="success" dismissible>
  Operation completed!
</Alert>
```

### 7.5 Extending Styled Components

```tsx
// Extend an existing styled component
const BaseButton = styled.button<{ fullWidth?: boolean }>`
  padding: 8px 16px;
  border-radius: 4px;
  cursor: pointer;
  width: ${({ fullWidth }) => (fullWidth ? "100%" : "auto")};
`;

// Extend with additional props
type OutlineButtonProps = {
  color?: string;
};

const OutlineButton = styled(BaseButton)<OutlineButtonProps>`
  background: transparent;
  border: 2px solid ${({ color, theme }) => color ?? theme.colors.primary};
  color: ${({ color, theme }) => color ?? theme.colors.primary};
`;

// Use .attrs() for default props
const IconButton = styled(BaseButton).attrs({
  type: "button",
})`
  padding: 8px;
  border-radius: 50%;
`;
```

### 7.6 Typing `as` Polymorphic Prop

```tsx
// styled-components supports the `as` prop for rendering different elements
const Text = styled.p<{ bold?: boolean }>`
  font-weight: ${({ bold }) => (bold ? 700 : 400)};
`;

// Render as an h1, h2, span, etc.
<Text as="h1" bold>Page Title</Text>
<Text as="span">Inline text</Text>
<Text as="a" href="/link">A link</Text>
```

---

## 8. Environment Variable Typing (`import.meta.env`)

Vite uses `import.meta.env` for environment variables, unlike Create React App's `process.env`.

### 8.1 The Default `ImportMeta` Interface

Vite already provides basic types, but they're very loose:

```ts
// What Vite provides out of the box (simplified)
interface ImportMeta {
  env: {
    MODE: string;
    BASE_URL: string;
    PROD: boolean;
    DEV: boolean;
    SSR: boolean;
    [key: string]: string | boolean | undefined;
  };
}
```

### 8.2 Typing Your Custom Env Variables

Augment `ImportMetaEnv` in your Vite project:

```ts
// src/vite-env.d.ts (or src/env.d.ts)
/// <reference types="vite/client" />

interface ImportMetaEnv {
  // Required — will error if undefined at runtime
  readonly VITE_API_URL: string;
  readonly VITE_APP_NAME: string;

  // Optional
  readonly VITE_ANALYTICS_KEY?: string;
  readonly VITE_FEATURE_FLAGS?: string;

  // Typed with union for known values
  readonly VITE_LOG_LEVEL?: "debug" | "info" | "warn" | "error";

  // Built-in Vite vars (already declared — shown for clarity)
  readonly MODE: string;
  readonly BASE_URL: string;
  readonly PROD: boolean;
  readonly DEV: boolean;
}

// Required boilerplate to enable augmentation
interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

### 8.3 Using Typed Env Variables

```ts
// src/config/env.ts
// Centralize access and parse/validate env vars

type Config = {
  apiUrl: string;
  appName: string;
  analyticsKey: string | null;
  logLevel: "debug" | "info" | "warn" | "error";
  isDev: boolean;
  isProd: boolean;
};

function getConfig(): Config {
  const apiUrl = import.meta.env.VITE_API_URL;
  if (!apiUrl) {
    throw new Error("VITE_API_URL is required");
  }

  const rawLogLevel = import.meta.env.VITE_LOG_LEVEL;
  const validLogLevels = ["debug", "info", "warn", "error"] as const;
  const logLevel = validLogLevels.includes(rawLogLevel as any)
    ? (rawLogLevel as Config["logLevel"])
    : "info";

  return {
    apiUrl,
    appName: import.meta.env.VITE_APP_NAME ?? "My App",
    analyticsKey: import.meta.env.VITE_ANALYTICS_KEY ?? null,
    logLevel,
    isDev: import.meta.env.DEV,
    isProd: import.meta.env.PROD,
  };
}

// Export a singleton
export const config = getConfig();
```

```ts
// Usage
import { config } from "./config/env";

fetch(`${config.apiUrl}/users`); // Fully typed ✅
```

### 8.4 Feature Flags via Env Variables

```ts
// src/config/features.ts
type FeatureFlags = {
  newDashboard: boolean;
  betaCheckout: boolean;
  analyticsV2: boolean;
};

function parseFeatureFlags(raw?: string): FeatureFlags {
  const defaults: FeatureFlags = {
    newDashboard: false,
    betaCheckout: false,
    analyticsV2: false,
  };

  if (!raw) return defaults;

  try {
    const flags = raw.split(",").map((f) => f.trim());
    return {
      ...defaults,
      newDashboard: flags.includes("new-dashboard"),
      betaCheckout: flags.includes("beta-checkout"),
      analyticsV2: flags.includes("analytics-v2"),
    };
  } catch {
    return defaults;
  }
}

export const features = parseFeatureFlags(
  import.meta.env.VITE_FEATURE_FLAGS
);

// Usage
if (features.newDashboard) {
  // Render new dashboard
}
```

### 8.5 Create React App (`process.env`) Typing

For projects still using CRA, augment `NodeJS.ProcessEnv`:

```ts
// src/react-app-env.d.ts
/// <reference types="react-scripts" />

declare namespace NodeJS {
  interface ProcessEnv {
    readonly NODE_ENV: "development" | "production" | "test";
    readonly REACT_APP_API_URL: string;
    readonly REACT_APP_APP_NAME?: string;
    readonly REACT_APP_FEATURE_FLAGS?: string;
  }
}
```

```ts
// Usage in CRA
const apiUrl = process.env.REACT_APP_API_URL; // typed as string ✅
```

---

## 9. Putting It All Together

Here's a mini-application combining all concepts from this week:

```
src/
├── types/
│   ├── css-modules.d.ts
│   ├── vite-env.d.ts
│   └── styled.d.ts
├── store/
│   ├── index.ts           (RTK store + typed hooks)
│   ├── cartSlice.ts
│   └── userSlice.ts
├── stores/
│   └── authStore.ts       (Zustand)
├── config/
│   └── env.ts             (typed env vars)
├── router/
│   └── index.tsx          (typed routes + loaders)
├── components/
│   ├── Button.tsx
│   └── Button.module.css
└── theme.ts
```

### Full Example: Typed Product Page

```tsx
// routes/ProductPage.tsx
import { useParams, useLoaderData, Link } from "react-router-dom";
import { LoaderFunctionArgs } from "react-router-dom";
import styled from "styled-components";
import { useAppDispatch } from "../store";
import { addItem } from "../store/cartSlice";
import { config } from "../config/env";
import styles from "./ProductPage.module.css";

// Types
type Product = {
  id: string;
  name: string;
  price: number;
  description: string;
  category: string;
  imageUrl: string;
};

// Loader
export async function loader({
  params,
}: LoaderFunctionArgs): Promise<Product> {
  const res = await fetch(`${config.apiUrl}/products/${params.id}`);
  if (!res.ok) throw new Response("Not Found", { status: 404 });
  return res.json();
}

// Styled components
const HeroImage = styled.img<{ featured?: boolean }>`
  width: 100%;
  max-width: ${({ featured }) => (featured ? "800px" : "400px")};
  border-radius: ${({ theme }) => theme.spacing(1)};
`;

const Price = styled.span`
  font-size: 1.5rem;
  font-weight: bold;
  color: ${({ theme }) => theme.colors.primary};
`;

// Component
export function ProductPage() {
  const product = useLoaderData() as Product;
  const dispatch = useAppDispatch();

  const handleAddToCart = () => {
    dispatch(
      addItem({
        id: product.id,
        name: product.name,
        price: product.price,
        quantity: 1,
      })
    );
  };

  return (
    <div className={styles.container}>
      <Link to="/products">← Back to Products</Link>

      <HeroImage
        src={product.imageUrl}
        alt={product.name}
        featured
      />

      <h1 className={styles.title}>{product.name}</h1>
      <Price>${product.price.toFixed(2)}</Price>

      <p className={styles.description}>{product.description}</p>

      <button className={styles.addButton} onClick={handleAddToCart}>
        Add to Cart
      </button>
    </div>
  );
}
```

---

## 10. Exercises

### Exercise 1: React Router

Create a blog application with these typed routes:

- `/posts` — lists posts (loader fetches from API)
- `/posts/:slug` — displays a single post (loader fetches by slug)
- `/posts/new` — form with a typed action to create a post

**Bonus:** Add search params for pagination (`?page=1&limit=10`) with full types.

### Exercise 2: Redux Toolkit

Build a notifications system:

- A `notificationsSlice` with `add`, `dismiss`, and `clearAll` actions
- An async thunk `fetchNotifications` that fetches from an API
- Typed selectors for unread count and notifications by type
- A component using `useAppSelector` and `useAppDispatch`

### Exercise 3: Zustand

Create a shopping preferences store using Zustand with:

- `persist` middleware (saved to localStorage)
- `devtools` middleware
- State: `currency`, `preferredPayment`, `savedAddresses`
- Actions with correct TypeScript generics throughout

### Exercise 4: Declaration Files

Find an npm package without TypeScript support (check for `"types": false` in package.json or missing type definitions), and write a complete `.d.ts` file covering at least 5 exported functions/classes.

### Exercise 5: Module Augmentation

Augment the React Router `useLoaderData` return type to avoid the `as` cast, using a generic wrapper:

```ts
// Goal: make this work without a type assertion
export function useTypedLoaderData<T>(): T;
```

### Exercise 6: Environment Variables

Set up a Vite project with:

- 5 typed env variables (mix of required and optional)
- A `config.ts` that validates all required vars at startup
- A typed feature flags system parsed from a single `VITE_FEATURES` env var
- Tests that verify config throws when required vars are missing

---

## Summary

| Topic | Key Takeaway |
|---|---|
| React Router v6 | Type `useParams` with a generic, cast `useLoaderData`, type loaders with `LoaderFunctionArgs` |
| Redux Toolkit | Create `RootState` and `AppDispatch` types; use `PayloadAction<T>` in slices; use `createAsyncThunk<Return, Arg, Config>` |
| Zustand | Type the full state + actions as one type and pass it to `create<T>` |
| Declaration Files | Use `declare module` for untyped packages; match the package's exact export structure |
| Module Augmentation | Import the module first, then redeclare it in a `.d.ts` file to add properties |
| CSS Modules | Use `typed-css-modules` for generated types or `typescript-plugin-css-modules` for IDE support |
| styled-components | Use generics `styled.div<Props>` and augment `DefaultTheme` for global theme typing |
| Environment Variables | Augment `ImportMetaEnv` in `vite-env.d.ts`; centralize access in a `config.ts` with validation |

---

*Next week: Advanced TypeScript Patterns — conditional types, mapped types, template literal types, and `infer`.*
