# Week 7: Advanced React Typing with TypeScript

> A complete, hands-on guide to typing advanced React patterns with TypeScript.

---

## Table of Contents

1. [Typing `useReducer` with Discriminated Union Actions](#1-typing-usereducer-with-discriminated-union-actions)
2. [Typing Context API with Generics](#2-typing-context-api-with-generics)
3. [Typing Custom Hooks (Return Types)](#3-typing-custom-hooks-return-types)
4. [Typing `forwardRef` and `useImperativeHandle`](#4-typing-forwardref-and-useimperativehandle)
5. [Component Prop Patterns: Polymorphic Components](#5-component-prop-patterns-polymorphic-components)
6. [Typing Higher Order Components (HOCs)](#6-typing-higher-order-components-hocs)
7. [Typing Render Props](#7-typing-render-props)
8. [`React.ComponentProps` and `React.HTMLAttributes`](#8-reactcomponentprops-and-reacthtmlattributes)
9. [Typing React Query (TanStack Query)](#9-typing-react-query-tanstack-query)
10. [Typing Form Libraries: React Hook Form + Zod](#10-typing-form-libraries-react-hook-form--zod)

---

## 1. Typing `useReducer` with Discriminated Union Actions

Discriminated unions give the TypeScript compiler full knowledge of what payload is available for each action type, eliminating entire categories of runtime bugs.

### The Problem with Loose Typing

```typescript
// ❌ Bad — any action can carry any payload
type Action = {
  type: string;
  payload?: any;
};
```

### Defining Discriminated Union Actions

```typescript
// ✅ Good — each action type narrows its own payload
type CounterAction =
  | { type: "INCREMENT" }
  | { type: "DECREMENT" }
  | { type: "RESET" }
  | { type: "SET_VALUE"; payload: number };
```

Every branch carries exactly the data it needs — nothing more, nothing less.

### State Interface

```typescript
interface CounterState {
  count: number;
  history: number[];
}

const initialState: CounterState = {
  count: 0,
  history: [],
};
```

### Fully-Typed Reducer

```typescript
function counterReducer(
  state: CounterState,
  action: CounterAction
): CounterState {
  switch (action.type) {
    case "INCREMENT":
      return { ...state, count: state.count + 1, history: [...state.history, state.count] };

    case "DECREMENT":
      return { ...state, count: state.count - 1, history: [...state.history, state.count] };

    case "RESET":
      return initialState;

    case "SET_VALUE":
      // TypeScript knows `action.payload` is `number` here
      return { ...state, count: action.payload, history: [...state.history, state.count] };

    default:
      // Exhaustiveness check — compile error if a case is missing
      const _exhaustive: never = action;
      return _exhaustive;
  }
}
```

### Using the Reducer in a Component

```typescript
import { useReducer } from "react";

function Counter() {
  const [state, dispatch] = useReducer(counterReducer, initialState);

  return (
    <div>
      <p>Count: {state.count}</p>
      <button onClick={() => dispatch({ type: "INCREMENT" })}>+</button>
      <button onClick={() => dispatch({ type: "DECREMENT" })}>-</button>
      <button onClick={() => dispatch({ type: "SET_VALUE", payload: 100 })}>
        Set 100
      </button>
      {/* ❌ Compile error: payload is required for SET_VALUE */}
      {/* dispatch({ type: "SET_VALUE" }) */}
    </div>
  );
}
```

### Complex Real-World Example: Shopping Cart

```typescript
type Product = { id: string; name: string; price: number };

type CartItem = Product & { quantity: number };

type CartAction =
  | { type: "ADD_ITEM"; payload: Product }
  | { type: "REMOVE_ITEM"; payload: { id: string } }
  | { type: "UPDATE_QUANTITY"; payload: { id: string; quantity: number } }
  | { type: "CLEAR_CART" }
  | { type: "APPLY_DISCOUNT"; payload: { code: string; percent: number } };

interface CartState {
  items: CartItem[];
  discountPercent: number;
  discountCode: string | null;
}

const cartInitialState: CartState = {
  items: [],
  discountPercent: 0,
  discountCode: null,
};

function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case "ADD_ITEM": {
      const existing = state.items.find((i) => i.id === action.payload.id);
      if (existing) {
        return {
          ...state,
          items: state.items.map((i) =>
            i.id === action.payload.id ? { ...i, quantity: i.quantity + 1 } : i
          ),
        };
      }
      return {
        ...state,
        items: [...state.items, { ...action.payload, quantity: 1 }],
      };
    }

    case "REMOVE_ITEM":
      return {
        ...state,
        items: state.items.filter((i) => i.id !== action.payload.id),
      };

    case "UPDATE_QUANTITY":
      return {
        ...state,
        items: state.items.map((i) =>
          i.id === action.payload.id
            ? { ...i, quantity: action.payload.quantity }
            : i
        ),
      };

    case "CLEAR_CART":
      return cartInitialState;

    case "APPLY_DISCOUNT":
      return {
        ...state,
        discountCode: action.payload.code,
        discountPercent: action.payload.percent,
      };

    default:
      const _exhaustive: never = action;
      return _exhaustive;
  }
}
```

> **Key Takeaways:**
> - Each action type's `payload` is narrowed automatically inside the `case` block.
> - The `never` exhaustiveness check becomes a compile-time safety net.
> - Wrap `case` bodies in `{}` blocks to avoid variable hoisting across cases.

---

## 2. Typing Context API with Generics

Context without types is a common source of `undefined` errors. Generic context wrappers solve this cleanly.

### Naive Approach and Its Problems

```typescript
// ❌ Problematic — consumers must check for undefined everywhere
const ThemeContext = React.createContext<string | undefined>(undefined);

function useTheme() {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error("useTheme must be used within ThemeProvider");
  return ctx;
}
```

### Generic Context Factory

```typescript
import { createContext, useContext, ReactNode } from "react";

/**
 * Creates a typed context + hook pair.
 * The hook throws if used outside the provider, so consumers never get `undefined`.
 */
function createTypedContext<T>(displayName: string) {
  const Context = createContext<T | undefined>(undefined);
  Context.displayName = displayName;

  function useTypedContext(): T {
    const ctx = useContext(Context);
    if (ctx === undefined) {
      throw new Error(`use${displayName} must be used within ${displayName}Provider`);
    }
    return ctx;
  }

  return [Context, useTypedContext] as const;
}
```

### Auth Context Example

```typescript
interface User {
  id: string;
  email: string;
  role: "admin" | "user" | "guest";
}

interface AuthContextValue {
  user: User | null;
  isLoading: boolean;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
}

const [AuthContext, useAuth] = createTypedContext<AuthContextValue>("Auth");

function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(false);

  const login = async (email: string, password: string) => {
    setIsLoading(true);
    try {
      const user = await fakeLoginApi(email, password);
      setUser(user);
    } finally {
      setIsLoading(false);
    }
  };

  const logout = () => setUser(null);

  return (
    <AuthContext.Provider value={{ user, isLoading, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

// Consumer — fully typed, no undefined checks needed
function UserProfile() {
  const { user, logout } = useAuth(); // TypeScript knows the full shape
  if (!user) return <p>Not logged in</p>;
  return (
    <div>
      <p>{user.email}</p>
      <button onClick={logout}>Log out</button>
    </div>
  );
}
```

### Generic State Context

```typescript
interface StateContextValue<T> {
  state: T;
  setState: React.Dispatch<React.SetStateAction<T>>;
}

function createStateContext<T>(defaultValue: T) {
  const [Context, useCtx] = createTypedContext<StateContextValue<T>>("State");

  function StateProvider({ children }: { children: ReactNode }) {
    const [state, setState] = useState<T>(defaultValue);
    return (
      <Context.Provider value={{ state, setState }}>
        {children}
      </Context.Provider>
    );
  }

  return { StateProvider, useCtx };
}

// Usage
const { StateProvider: SettingsProvider, useCtx: useSettings } =
  createStateContext({ theme: "dark", language: "en" });
```

> **Key Takeaways:**
> - The factory pattern avoids repeating the `undefined` guard for every context.
> - Always set `displayName` for React DevTools clarity.
> - Return `as const` from the factory to preserve tuple types.

---

## 3. Typing Custom Hooks (Return Types)

Explicit return types on custom hooks prevent accidental API changes and make hooks self-documenting.

### Typing Named Return Objects

```typescript
interface UsePaginationOptions {
  totalItems: number;
  itemsPerPage?: number;
  initialPage?: number;
}

interface UsePaginationReturn {
  currentPage: number;
  totalPages: number;
  hasNext: boolean;
  hasPrev: boolean;
  goToPage: (page: number) => void;
  nextPage: () => void;
  prevPage: () => void;
}

function usePagination({
  totalItems,
  itemsPerPage = 10,
  initialPage = 1,
}: UsePaginationOptions): UsePaginationReturn {
  const [currentPage, setCurrentPage] = useState(initialPage);
  const totalPages = Math.ceil(totalItems / itemsPerPage);

  const goToPage = (page: number) => {
    setCurrentPage(Math.max(1, Math.min(page, totalPages)));
  };

  return {
    currentPage,
    totalPages,
    hasNext: currentPage < totalPages,
    hasPrev: currentPage > 1,
    goToPage,
    nextPage: () => goToPage(currentPage + 1),
    prevPage: () => goToPage(currentPage - 1),
  };
}
```

### Typing Tuple Returns

```typescript
// Use `as const` to preserve the tuple type instead of widening to an array
function useToggle(initial = false): [boolean, () => void, (v: boolean) => void] {
  const [value, setValue] = useState(initial);
  const toggle = () => setValue((v) => !v);
  return [value, toggle, setValue] as const;
}

// With const assertion on the return type annotation instead:
function useCounter(initial = 0) {
  const [count, setCount] = useState(initial);
  return {
    count,
    increment: () => setCount((c) => c + 1),
    decrement: () => setCount((c) => c - 1),
    reset: () => setCount(initial),
  } as const; // prevents callers from mutating the returned object
}
```

### Generic Async Data Hook

```typescript
type AsyncState<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: Error };

interface UseAsyncReturn<T> {
  state: AsyncState<T>;
  execute: (...args: Parameters<() => Promise<T>>) => Promise<void>;
  reset: () => void;
}

function useAsync<T>(asyncFn: () => Promise<T>): UseAsyncReturn<T> {
  const [state, setState] = useState<AsyncState<T>>({ status: "idle" });

  const execute = async () => {
    setState({ status: "loading" });
    try {
      const data = await asyncFn();
      setState({ status: "success", data });
    } catch (err) {
      setState({ status: "error", error: err instanceof Error ? err : new Error(String(err)) });
    }
  };

  return {
    state,
    execute,
    reset: () => setState({ status: "idle" }),
  };
}

// Consumer — discriminated union narrows correctly in JSX
function DataDisplay() {
  const { state, execute } = useAsync(() => fetch("/api/data").then(r => r.json()));

  if (state.status === "loading") return <p>Loading…</p>;
  if (state.status === "error") return <p>Error: {state.error.message}</p>;
  if (state.status === "success") return <pre>{JSON.stringify(state.data, null, 2)}</pre>;
  return <button onClick={execute}>Fetch Data</button>;
}
```

### Typing Hooks That Accept Callbacks

```typescript
function useDebounce<T extends (...args: any[]) => any>(
  fn: T,
  delay: number
): (...args: Parameters<T>) => void {
  const timeoutRef = useRef<ReturnType<typeof setTimeout>>();

  return useCallback(
    (...args: Parameters<T>) => {
      clearTimeout(timeoutRef.current);
      timeoutRef.current = setTimeout(() => fn(...args), delay);
    },
    [fn, delay]
  );
}
```

> **Key Takeaways:**
> - Always annotate the return type explicitly — it's the hook's public API contract.
> - Use `as const` on tuple returns to prevent TypeScript from widening to `Array`.
> - `Parameters<T>` and `ReturnType<T>` are your best friends for generic callback hooks.

---

## 4. Typing `forwardRef` and `useImperativeHandle`

`forwardRef` and `useImperativeHandle` need explicit generic parameters to stay fully typed.

### Basic `forwardRef`

```typescript
import { forwardRef, useRef } from "react";

interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string;
  error?: string;
}

// Generic parameters: <RefType, PropsType>
const Input = forwardRef<HTMLInputElement, InputProps>(
  ({ label, error, ...rest }, ref) => {
    return (
      <div>
        <label>{label}</label>
        <input ref={ref} {...rest} className={error ? "input-error" : ""} />
        {error && <span className="error-message">{error}</span>}
      </div>
    );
  }
);

Input.displayName = "Input"; // Required for React DevTools

// Usage
function Form() {
  const inputRef = useRef<HTMLInputElement>(null);

  const focusInput = () => inputRef.current?.focus();

  return (
    <>
      <Input ref={inputRef} label="Email" type="email" />
      <button onClick={focusInput}>Focus</button>
    </>
  );
}
```

### `useImperativeHandle` — Exposing a Custom API

```typescript
import { forwardRef, useImperativeHandle, useRef } from "react";

// Define exactly what the parent can call
interface VideoPlayerHandle {
  play: () => void;
  pause: () => void;
  seek: (seconds: number) => void;
  getCurrentTime: () => number;
}

interface VideoPlayerProps {
  src: string;
  autoPlay?: boolean;
}

const VideoPlayer = forwardRef<VideoPlayerHandle, VideoPlayerProps>(
  ({ src, autoPlay }, ref) => {
    const videoRef = useRef<HTMLVideoElement>(null);

    // Expose a controlled subset of the DOM API
    useImperativeHandle(ref, () => ({
      play: () => videoRef.current?.play(),
      pause: () => videoRef.current?.pause(),
      seek: (seconds) => {
        if (videoRef.current) videoRef.current.currentTime = seconds;
      },
      getCurrentTime: () => videoRef.current?.currentTime ?? 0,
    }));

    return <video ref={videoRef} src={src} autoPlay={autoPlay} />;
  }
);

VideoPlayer.displayName = "VideoPlayer";

// Parent — can only call the exposed API, not the full HTMLVideoElement
function App() {
  const playerRef = useRef<VideoPlayerHandle>(null);

  return (
    <>
      <VideoPlayer ref={playerRef} src="/video.mp4" />
      <button onClick={() => playerRef.current?.play()}>Play</button>
      <button onClick={() => playerRef.current?.seek(30)}>Skip 30s</button>
      {/* ❌ Compile error — `.volume` is not on VideoPlayerHandle */}
      {/* playerRef.current?.volume */}
    </>
  );
}
```

### Combining Multiple Refs Internally

```typescript
function useCombinedRef<T>(...refs: React.Ref<T>[]) {
  return useCallback((element: T) => {
    refs.forEach((ref) => {
      if (typeof ref === "function") {
        ref(element);
      } else if (ref && "current" in ref) {
        (ref as React.MutableRefObject<T>).current = element;
      }
    });
  }, refs);
}

// Usage: exposing ref while also keeping an internal one
const FancyInput = forwardRef<HTMLInputElement, InputProps>((props, externalRef) => {
  const internalRef = useRef<HTMLInputElement>(null);
  const combinedRef = useCombinedRef(internalRef, externalRef);

  useEffect(() => {
    // Can still use internalRef for internal logic
    internalRef.current?.focus();
  }, []);

  return <input ref={combinedRef} {...props} />;
});
```

> **Key Takeaways:**
> - The first generic on `forwardRef<RefType, Props>` is the ref target, not the component.
> - `useImperativeHandle` lets you expose a minimal, safe API instead of the raw DOM node.
> - Always set `displayName` — it greatly improves the React DevTools experience.

---

## 5. Component Prop Patterns: Polymorphic Components

Polymorphic (or "as-prop") components render as different HTML elements while staying fully typed.

### The Challenge

```typescript
// We want: <Text as="h1">, <Text as="p">, <Text as="span">
// Each `as` value should bring its own HTML attributes into scope.
```

### The Polymorphic Helpers

```typescript
type AsProp<C extends React.ElementType> = { as?: C };

type PropsToOmit<C extends React.ElementType, P> = keyof (AsProp<C> & P);

// Merges your custom props with the native element's props, minus conflicts
type PolymorphicComponentProp<
  C extends React.ElementType,
  Props = {}
> = React.PropsWithChildren<Props & AsProp<C>> &
  Omit<React.ComponentPropsWithoutRef<C>, PropsToOmit<C, Props>>;

// Adds ref support
type PolymorphicComponentPropWithRef<
  C extends React.ElementType,
  Props = {}
> = PolymorphicComponentProp<C, Props> & { ref?: PolymorphicRef<C> };

type PolymorphicRef<C extends React.ElementType> =
  React.ComponentPropsWithRef<C>["ref"];
```

### Polymorphic `Text` Component

```typescript
type TextProps<C extends React.ElementType> = PolymorphicComponentPropWithRef<
  C,
  {
    color?: "primary" | "secondary" | "muted";
    size?: "sm" | "md" | "lg" | "xl";
  }
>;

type TextComponent = <C extends React.ElementType = "span">(
  props: TextProps<C>
) => React.ReactElement | null;

const Text: TextComponent = forwardRef(
  <C extends React.ElementType = "span">(
    { as, color = "primary", size = "md", children, ...rest }: TextProps<C>,
    ref: PolymorphicRef<C>
  ) => {
    const Tag = as ?? "span";
    return (
      <Tag
        ref={ref}
        className={`text-${color} text-${size}`}
        {...rest}
      >
        {children}
      </Tag>
    );
  }
);

// Usage — TypeScript enforces correct props per element
<Text as="h1" size="xl">Heading</Text>         // ✅ h1 attrs available
<Text as="a" href="/about">Link</Text>         // ✅ href is valid on <a>
<Text as="button" onClick={() => {}}>Btn</Text> // ✅ onClick with button event type
<Text as="h1" href="/about">Bad</Text>         // ❌ href doesn't exist on h1
```

### Simpler Variant: Component Polymorphism via Union

```typescript
// For a smaller set of allowed elements, a union is often cleaner
type HeadingLevel = "h1" | "h2" | "h3" | "h4" | "h5" | "h6";

interface HeadingProps extends React.HTMLAttributes<HTMLHeadingElement> {
  level?: HeadingLevel;
}

function Heading({ level = "h2", children, ...rest }: HeadingProps) {
  const Tag = level;
  return <Tag {...rest}>{children}</Tag>;
}
```

> **Key Takeaways:**
> - The `as` prop pattern is powerful but adds generic complexity — use a union type when the set of elements is small.
> - `React.ComponentPropsWithoutRef<C>` excludes the `ref` prop, which you handle separately.
> - Wrapping with `forwardRef` requires a separate named type for the component rather than inline inference.

---

## 6. Typing Higher Order Components (HOCs)

HOCs wrap a component and return a new one. The key is preserving the inner component's props while potentially injecting or removing some.

### HOC That Injects Props

```typescript
// The HOC injects `user` — consumers don't pass it
interface WithAuthProps {
  user: User;
}

function withAuth<P extends WithAuthProps>(
  WrappedComponent: React.ComponentType<P>
): React.ComponentType<Omit<P, keyof WithAuthProps>> {
  const displayName =
    WrappedComponent.displayName ?? WrappedComponent.name ?? "Component";

  function WithAuthHOC(props: Omit<P, keyof WithAuthProps>) {
    const { user } = useAuth(); // from context (see Section 2)
    if (!user) return <Redirect to="/login" />;

    // Re-cast is necessary because TypeScript can't verify the merge is safe
    return <WrappedComponent {...(props as P)} user={user} />;
  }

  WithAuthHOC.displayName = `withAuth(${displayName})`;
  return WithAuthHOC;
}

// Usage
interface DashboardProps extends WithAuthProps {
  title: string;
}

function Dashboard({ user, title }: DashboardProps) {
  return <h1>{title} — logged in as {user.email}</h1>;
}

const ProtectedDashboard = withAuth(Dashboard);

// `user` is now injected — consumer only provides `title`
<ProtectedDashboard title="Admin Panel" />
```

### HOC That Adds Loading State

```typescript
interface WithLoadingProps {
  isLoading: boolean;
}

function withLoading<P extends WithLoadingProps>(
  WrappedComponent: React.ComponentType<P>,
  LoadingComponent: React.ComponentType = () => <p>Loading…</p>
): React.ComponentType<P> {
  function WithLoading({ isLoading, ...rest }: P) {
    if (isLoading) return <LoadingComponent />;
    return <WrappedComponent {...(rest as P)} isLoading={isLoading} />;
  }

  WithLoading.displayName = `withLoading(${WrappedComponent.displayName ?? WrappedComponent.name})`;
  return WithLoading;
}
```

### Composing Multiple HOCs

```typescript
// Utility to compose HOCs from right to left
function compose<T>(...fns: Array<(arg: T) => T>) {
  return fns.reduce((f, g) => (x: T) => f(g(x)));
}

const enhance = compose(
  withAuth,
  withLoading
) as <P>(c: React.ComponentType<P>) => React.ComponentType<P>;
```

> **Key Takeaways:**
> - Use `Omit<P, keyof InjectedProps>` to strip injected props from the outer component's signature.
> - Always forward `displayName` for debugging.
> - When type casts are needed (as P), add a comment explaining why TypeScript can't infer it.
> - Custom hooks are usually a simpler alternative to HOCs — prefer hooks when possible.

---

## 7. Typing Render Props

Render props pass a function as a prop, letting the parent control rendering while the child controls logic.

### Basic Render Prop

```typescript
interface MousePosition {
  x: number;
  y: number;
}

interface MouseTrackerProps {
  render: (position: MousePosition) => React.ReactNode;
}

function MouseTracker({ render }: MouseTrackerProps) {
  const [position, setPosition] = useState<MousePosition>({ x: 0, y: 0 });

  const handleMouseMove = (e: React.MouseEvent) => {
    setPosition({ x: e.clientX, y: e.clientY });
  };

  return (
    <div style={{ width: "100%", height: "100vh" }} onMouseMove={handleMouseMove}>
      {render(position)}
    </div>
  );
}

// Usage
<MouseTracker
  render={({ x, y }) => <p>Mouse is at ({x}, {y})</p>}
/>
```

### Children-as-Function Pattern

```typescript
interface DataFetcherProps<T> {
  url: string;
  children: (state: AsyncState<T>) => React.ReactNode; // Using AsyncState from Section 3
}

function DataFetcher<T>({ url, children }: DataFetcherProps<T>) {
  const [state, setState] = useState<AsyncState<T>>({ status: "idle" });

  useEffect(() => {
    setState({ status: "loading" });
    fetch(url)
      .then((r) => r.json())
      .then((data: T) => setState({ status: "success", data }))
      .catch((error: Error) => setState({ status: "error", error }));
  }, [url]);

  return <>{children(state)}</>;
}

// Usage
<DataFetcher<User[]> url="/api/users">
  {(state) => {
    if (state.status === "loading") return <Spinner />;
    if (state.status === "error") return <ErrorMessage error={state.error} />;
    if (state.status === "success") return <UserList users={state.data} />;
    return null;
  }}
</DataFetcher>
```

### Render Props with Callbacks

```typescript
interface SelectProps<T> {
  items: T[];
  keyExtractor: (item: T) => string;
  renderItem: (item: T, isSelected: boolean, onSelect: () => void) => React.ReactNode;
  renderEmpty?: () => React.ReactNode;
  onSelectionChange?: (selected: T[]) => void;
}

function Select<T>({
  items,
  keyExtractor,
  renderItem,
  renderEmpty,
  onSelectionChange,
}: SelectProps<T>) {
  const [selectedKeys, setSelectedKeys] = useState<Set<string>>(new Set());

  const toggleItem = (item: T) => {
    const key = keyExtractor(item);
    setSelectedKeys((prev) => {
      const next = new Set(prev);
      next.has(key) ? next.delete(key) : next.add(key);
      const selected = items.filter((i) => next.has(keyExtractor(i)));
      onSelectionChange?.(selected);
      return next;
    });
  };

  if (items.length === 0) return <>{renderEmpty?.()}</>;

  return (
    <ul>
      {items.map((item) => {
        const key = keyExtractor(item);
        return (
          <li key={key}>
            {renderItem(item, selectedKeys.has(key), () => toggleItem(item))}
          </li>
        );
      })}
    </ul>
  );
}
```

> **Key Takeaways:**
> - The `children` prop typed as a function is idiomatic React — prefer it over a prop named `render`.
> - Generic render-prop components (`<DataFetcher<T>>`) provide full type inference to callers.
> - Consider custom hooks as an alternative — they often produce cleaner consumer code.

---

## 8. `React.ComponentProps` and `React.HTMLAttributes`

These utility types let you build on existing component APIs without redefining them from scratch.

### `React.ComponentProps` Variants

```typescript
// React.ComponentProps<T>       — includes the `ref` prop
// React.ComponentPropsWithRef<T> — same as above (explicit)
// React.ComponentPropsWithoutRef<T> — excludes `ref` (preferred for most cases)

// Extract props from a DOM element
type ButtonProps = React.ComponentPropsWithoutRef<"button">;
type DivProps   = React.ComponentPropsWithoutRef<"div">;
type AProps     = React.ComponentPropsWithoutRef<"a">;

// Extract props from a custom component
type MyInputProps = React.ComponentPropsWithoutRef<typeof Input>; // from Section 4
```

### Extending Native Element Props

```typescript
// ✅ Correct pattern — extend, don't duplicate
interface CardProps extends React.ComponentPropsWithoutRef<"div"> {
  variant?: "outlined" | "filled" | "elevated";
  padding?: "sm" | "md" | "lg";
}

function Card({ variant = "filled", padding = "md", className, ...rest }: CardProps) {
  return (
    <div
      className={`card card--${variant} card--pad-${padding} ${className ?? ""}`}
      {...rest} // All standard div props (onClick, style, data-*, aria-*, etc.)
    />
  );
}
```

### Merging Custom Props with HTML Attributes

```typescript
// Use Omit when your prop name conflicts with a native one
interface ButtonProps extends Omit<React.ButtonHTMLAttributes<HTMLButtonElement>, "color"> {
  color?: "primary" | "secondary" | "danger"; // Custom union, not CSS string
  isLoading?: boolean;
  leftIcon?: React.ReactNode;
}

const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  ({ color = "primary", isLoading, leftIcon, children, disabled, ...rest }, ref) => (
    <button
      ref={ref}
      disabled={disabled || isLoading}
      data-color={color}
      {...rest}
    >
      {isLoading ? <Spinner /> : leftIcon}
      {children}
    </button>
  )
);
```

### `React.HTMLAttributes` for Generic Wrappers

```typescript
// When you don't know the specific element type
interface BoxProps extends React.HTMLAttributes<HTMLElement> {
  as?: keyof JSX.IntrinsicElements;
}

// Type-safe event handler extraction
type ClickHandler = React.MouseEventHandler<HTMLButtonElement>;
type ChangeHandler = React.ChangeEventHandler<HTMLInputElement>;
type KeyHandler = React.KeyboardEventHandler<HTMLElement>;

// ComponentPropsWithRef for building wrapper components
type TextareaWrapperProps = React.ComponentPropsWithRef<"textarea"> & {
  label: string;
  hint?: string;
};
```

### Extracting Prop Types from Third-Party Components

```typescript
import { Button } from "some-ui-library";

// Reuse the library's prop types without importing them directly
type LibraryButtonProps = React.ComponentPropsWithoutRef<typeof Button>;

// Or pick a subset
type MyButtonSize = LibraryButtonProps["size"]; // e.g., "sm" | "md" | "lg"

// Build on top of a library component
interface StyledButtonProps extends React.ComponentPropsWithoutRef<typeof Button> {
  fullWidth?: boolean;
}
```

> **Key Takeaways:**
> - Prefer `ComponentPropsWithoutRef` for most cases and `ComponentPropsWithRef` only when explicitly forwarding refs.
> - Use `Omit<..., "conflictingProp">` when you need to redefine a prop with a narrower type.
> - `ComponentPropsWithoutRef<typeof YourComponent>` extracts the props of any component, even third-party ones.

---

## 9. Typing React Query (TanStack Query)

TanStack Query (v5) has strong built-in generics. The key is understanding the four type parameters and how to use them effectively.

### Installation

```bash
npm install @tanstack/react-query
```

### The Four Generic Parameters

```typescript
useQuery<
  TData,       // What the query function returns
  TError,      // What is thrown on failure
  TSelectedData, // Output after `select` transform (defaults to TData)
  TQueryKey    // The query key tuple type
>
```

### Basic Typed Query

```typescript
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";

interface User {
  id: number;
  name: string;
  email: string;
}

// API layer — typed fetch functions
async function fetchUser(id: number): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error("Failed to fetch user");
  return res.json();
}

function UserCard({ userId }: { userId: number }) {
  const { data, isLoading, isError, error } = useQuery({
    queryKey: ["user", userId],   // TypeScript infers the key type
    queryFn: () => fetchUser(userId),
  });
  // `data` is `User | undefined`
  // `error` is `Error | null`

  if (isLoading) return <Skeleton />;
  if (isError) return <p>Error: {error.message}</p>;
  if (!data) return null;

  return <div>{data.name} — {data.email}</div>;
}
```

### Custom Query Hook Pattern

```typescript
interface PaginatedResponse<T> {
  data: T[];
  total: number;
  page: number;
  pageSize: number;
}

// Encapsulate query logic in a dedicated hook
function useUsers(page: number, pageSize = 20) {
  return useQuery({
    queryKey: ["users", { page, pageSize }] as const,
    queryFn: (): Promise<PaginatedResponse<User>> =>
      fetch(`/api/users?page=${page}&pageSize=${pageSize}`).then((r) => r.json()),
    placeholderData: (prev) => prev, // Keep previous page while loading
    staleTime: 1000 * 60 * 5,       // 5 minutes
  });
}

// Usage
function UserList() {
  const [page, setPage] = useState(1);
  const { data, isPending, isFetching } = useUsers(page);

  return (
    <>
      {isFetching && <p>Refreshing…</p>}
      {data?.data.map((user) => <UserCard key={user.id} userId={user.id} />)}
      <Pagination
        page={page}
        total={data?.total ?? 0}
        pageSize={20}
        onPageChange={setPage}
      />
    </>
  );
}
```

### Typing Mutations

```typescript
interface CreateUserDto {
  name: string;
  email: string;
  role: "admin" | "user";
}

async function createUser(dto: CreateUserDto): Promise<User> {
  const res = await fetch("/api/users", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(dto),
  });
  if (!res.ok) throw new Error("Failed to create user");
  return res.json();
}

function CreateUserForm() {
  const queryClient = useQueryClient();

  const mutation = useMutation({
    mutationFn: createUser,
    onSuccess: (newUser) => {
      // `newUser` is typed as `User`
      queryClient.invalidateQueries({ queryKey: ["users"] });
      queryClient.setQueryData(["user", newUser.id], newUser);
    },
    onError: (error: Error) => {
      console.error(error.message);
    },
  });

  const handleSubmit = (dto: CreateUserDto) => {
    mutation.mutate(dto);
    // Or: await mutation.mutateAsync(dto);
  };

  return (
    <form onSubmit={/* ... */}>
      {mutation.isPending && <p>Creating…</p>}
      {mutation.isError && <p>Error: {mutation.error.message}</p>}
      {mutation.isSuccess && <p>User created!</p>}
    </form>
  );
}
```

### Using `select` for Data Transformation

```typescript
function useAdminUsers() {
  return useQuery({
    queryKey: ["users"],
    queryFn: (): Promise<User[]> => fetch("/api/users").then((r) => r.json()),
    // `select` transforms TData → TSelectedData
    select: (users) => users.filter((u) => u.role === "admin"),
    // `data` is now typed as `User[]` (admins only)
  });
}
```

### Typing `useInfiniteQuery`

```typescript
function useInfiniteUsers() {
  return useInfiniteQuery({
    queryKey: ["users", "infinite"],
    queryFn: ({ pageParam }: { pageParam: number }) =>
      fetch(`/api/users?page=${pageParam}`).then(
        (r): Promise<PaginatedResponse<User>> => r.json()
      ),
    initialPageParam: 1,
    getNextPageParam: (lastPage) =>
      lastPage.page * lastPage.pageSize < lastPage.total
        ? lastPage.page + 1
        : undefined,
  });
}
```

> **Key Takeaways:**
> - Let TypeScript infer generics from `queryFn`'s return type — explicit annotations are rarely needed.
> - The `select` option changes the output type independently of the fetched type.
> - Use `as const` on `queryKey` arrays to get precise literal types for key invalidation.
> - Create a dedicated API layer (typed async functions) — it keeps component code clean.

---

## 10. Typing Form Libraries: React Hook Form + Zod

Combining React Hook Form with Zod gives you runtime validation and compile-time type safety from a single source of truth — the schema.

### Installation

```bash
npm install react-hook-form zod @hookform/resolvers
```

### Defining the Schema (Single Source of Truth)

```typescript
import { z } from "zod";

const signUpSchema = z
  .object({
    name: z.string().min(2, "Name must be at least 2 characters"),
    email: z.string().email("Please enter a valid email"),
    password: z
      .string()
      .min(8, "Password must be at least 8 characters")
      .regex(/[A-Z]/, "Must contain at least one uppercase letter")
      .regex(/[0-9]/, "Must contain at least one number"),
    confirmPassword: z.string(),
    role: z.enum(["admin", "user", "guest"]),
    age: z.number().int().min(18, "Must be at least 18").optional(),
    agreeToTerms: z.literal(true, {
      errorMap: () => ({ message: "You must agree to the terms" }),
    }),
  })
  .refine((data) => data.password === data.confirmPassword, {
    message: "Passwords do not match",
    path: ["confirmPassword"],
  });

// Derive the TypeScript type from the schema
type SignUpFormData = z.infer<typeof signUpSchema>;
// Equivalent to:
// {
//   name: string;
//   email: string;
//   password: string;
//   confirmPassword: string;
//   role: "admin" | "user" | "guest";
//   age?: number | undefined;
//   agreeToTerms: true;
// }
```

### Full Form Component

```typescript
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";

function SignUpForm() {
  const {
    register,
    handleSubmit,
    watch,
    formState: { errors, isSubmitting, isValid },
    setError,
    reset,
  } = useForm<SignUpFormData>({
    resolver: zodResolver(signUpSchema),
    defaultValues: { role: "user" },
    mode: "onTouched", // Validate on blur, then on change
  });

  const onSubmit = async (data: SignUpFormData) => {
    // `data` is fully typed and validated
    try {
      await apiCreateUser(data);
      reset();
    } catch (error) {
      // Map server errors back to fields
      setError("email", { message: "This email is already registered" });
    }
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)} noValidate>
      <div>
        <label htmlFor="name">Name</label>
        <input id="name" {...register("name")} aria-invalid={!!errors.name} />
        {errors.name && <span role="alert">{errors.name.message}</span>}
      </div>

      <div>
        <label htmlFor="email">Email</label>
        <input id="email" type="email" {...register("email")} />
        {errors.email && <span role="alert">{errors.email.message}</span>}
      </div>

      <div>
        <label htmlFor="password">Password</label>
        <input id="password" type="password" {...register("password")} />
        {errors.password && <span role="alert">{errors.password.message}</span>}
      </div>

      <div>
        <label htmlFor="confirmPassword">Confirm Password</label>
        <input id="confirmPassword" type="password" {...register("confirmPassword")} />
        {errors.confirmPassword && (
          <span role="alert">{errors.confirmPassword.message}</span>
        )}
      </div>

      <div>
        <label htmlFor="role">Role</label>
        <select id="role" {...register("role")}>
          <option value="user">User</option>
          <option value="admin">Admin</option>
          <option value="guest">Guest</option>
        </select>
      </div>

      <div>
        <input id="agreeToTerms" type="checkbox" {...register("agreeToTerms")} />
        <label htmlFor="agreeToTerms">I agree to the Terms of Service</label>
        {errors.agreeToTerms && (
          <span role="alert">{errors.agreeToTerms.message}</span>
        )}
      </div>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? "Creating account…" : "Sign Up"}
      </button>
    </form>
  );
}
```

### Typed Custom Field Components

```typescript
import { useFormContext, Controller } from "react-hook-form";

// Reusable typed field that reads from the nearest FormProvider
interface FormFieldProps {
  name: keyof SignUpFormData; // Constrained to valid field names
  label: string;
  type?: React.InputHTMLAttributes<HTMLInputElement>["type"];
}

function FormField({ name, label, type = "text" }: FormFieldProps) {
  const {
    register,
    formState: { errors },
  } = useFormContext<SignUpFormData>();

  return (
    <div>
      <label htmlFor={name}>{label}</label>
      <input id={name} type={type} {...register(name)} />
      {errors[name] && (
        <span role="alert">{errors[name]?.message as string}</span>
      )}
    </div>
  );
}
```

### Composing Complex Schemas

```typescript
// Re-usable schema pieces
const addressSchema = z.object({
  street: z.string().min(1),
  city: z.string().min(1),
  country: z.string().length(2, "Use a 2-letter country code"),
  postalCode: z.string().regex(/^\d{5}(-\d{4})?$/, "Invalid postal code"),
});

const phoneSchema = z
  .string()
  .regex(/^\+?[1-9]\d{1,14}$/, "Invalid phone number")
  .optional();

const checkoutSchema = z.object({
  customer: z.object({
    firstName: z.string().min(1),
    lastName: z.string().min(1),
    email: z.string().email(),
    phone: phoneSchema,
  }),
  shippingAddress: addressSchema,
  billingAddress: addressSchema,
  sameAsBilling: z.boolean().default(false),
});

type CheckoutFormData = z.infer<typeof checkoutSchema>;
// Deep nested types are fully inferred — no manual interface needed
```

### Watching Fields and Conditional Validation

```typescript
function CheckoutForm() {
  const { watch, register } = useForm<CheckoutFormData>({
    resolver: zodResolver(checkoutSchema),
  });

  const sameAsBilling = watch("sameAsBilling");

  return (
    <form>
      {/* ... */}
      <input type="checkbox" {...register("sameAsBilling")} />
      {!sameAsBilling && (
        // Only rendered (and validated) when needed
        <AddressFields prefix="shippingAddress" />
      )}
    </form>
  );
}
```

### Schema Helpers for Common Patterns

```typescript
// Reusable schema transformations
const stringToNumber = z.string().transform((val) => Number(val));

const nullableString = z.string().nullable().optional();

const isoDateString = z.string().refine(
  (val) => !isNaN(Date.parse(val)),
  { message: "Must be a valid ISO date string" }
);

// Discriminated union in Zod for polymorphic forms
const paymentSchema = z.discriminatedUnion("type", [
  z.object({ type: z.literal("card"), cardNumber: z.string(), cvv: z.string() }),
  z.object({ type: z.literal("paypal"), email: z.string().email() }),
  z.object({ type: z.literal("crypto"), walletAddress: z.string() }),
]);

type PaymentData = z.infer<typeof paymentSchema>;
// type PaymentData =
//   | { type: "card"; cardNumber: string; cvv: string }
//   | { type: "paypal"; email: string }
//   | { type: "crypto"; walletAddress: string }
```

> **Key Takeaways:**
> - `z.infer<typeof schema>` is your single source of truth — no more duplicated interface + schema.
> - Use `useFormContext` and `FormProvider` to share form state across deeply nested components.
> - `zodResolver` from `@hookform/resolvers` bridges Zod and RHF with zero boilerplate.
> - Compose schemas with `.merge()`, `.extend()`, and `.pick()` to avoid repetition.

---

## Summary

| Topic | Core Pattern | Key Utility |
|---|---|---|
| `useReducer` | Discriminated union actions | `never` exhaustiveness check |
| Context API | Factory with `createContext` | `as const` on return |
| Custom hooks | Explicit return type | `Parameters<T>`, `ReturnType<T>` |
| `forwardRef` | `forwardRef<RefType, Props>` | `useImperativeHandle` |
| Polymorphic | `as` prop + generics | `React.ComponentPropsWithoutRef<C>` |
| HOCs | `Omit<P, keyof Injected>` | `displayName` forwarding |
| Render props | Generic children-as-function | Discriminated union states |
| `ComponentProps` | `ComponentPropsWithoutRef<T>` | `Omit` for conflicts |
| React Query | Type from `queryFn` return | `select` for transformation |
| RHF + Zod | `z.infer<typeof schema>` | `zodResolver` |

---

*Week 7 complete. Next: Week 8 — TypeScript for State Management (Redux Toolkit, Zustand, Jotai)*
