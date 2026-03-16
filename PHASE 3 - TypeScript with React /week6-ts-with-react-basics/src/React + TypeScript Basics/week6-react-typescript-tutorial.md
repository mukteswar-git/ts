# Phase 3: TypeScript with React
## Week 6: React + TypeScript Basics

> **Prerequisites:** Familiarity with React fundamentals (components, hooks, props) and basic TypeScript (types, interfaces, generics).

---

## Table of Contents

1. [Setting Up Vite with React + TypeScript](#1-setting-up-vite-with-react--typescript)
2. [Typing Functional Components](#2-typing-functional-components)
3. [Props Interfaces and Optional Props](#3-props-interfaces-and-optional-props)
4. [Children Prop Typing](#4-children-prop-typing)
5. [Typing useState](#5-typing-usestate)
6. [Typing useRef](#6-typing-useref)
7. [Typing Event Handlers](#7-typing-event-handlers)
8. [Generic Components](#8-generic-components)
9. [Weekly Challenge](#9-weekly-challenge)

---

## 1. Setting Up Vite with React + TypeScript

Vite is the recommended build tool for modern React + TypeScript projects. It offers near-instant cold starts, lightning-fast HMR (Hot Module Replacement), and first-class TypeScript support out of the box.

### Scaffold the Project

```bash
# Using npm
npm create vite@latest my-app -- --template react-ts

# Using yarn
yarn create vite my-app --template react-ts

# Using pnpm
pnpm create vite my-app --template react-ts
```

Then install dependencies and start the dev server:

```bash
cd my-app
npm install
npm run dev
```

### Project Structure

After scaffolding, you'll see:

```
my-app/
├── public/
├── src/
│   ├── assets/
│   ├── App.tsx          ← Root component (TSX)
│   ├── App.css
│   ├── main.tsx         ← Entry point
│   └── vite-env.d.ts    ← Vite type declarations
├── index.html
├── package.json
├── tsconfig.json        ← TypeScript config
├── tsconfig.node.json
└── vite.config.ts
```

### Key Configuration Files

**`tsconfig.json`** — The TypeScript compiler configuration:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,

    /* Bundler mode */
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",          // ← Enables JSX without importing React

    /* Linting */
    "strict": true,              // ← Enables all strict type checks
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

> **Tip:** Always keep `"strict": true`. It catches real bugs early — undefined access, implicit `any`, and more.

**`vite-env.d.ts`** — Adds Vite-specific types (e.g., `import.meta.env`):

```typescript
/// <reference types="vite/client" />
```

### File Extensions: `.ts` vs `.tsx`

| Extension | Use for |
|-----------|---------|
| `.ts`     | Pure TypeScript (utilities, hooks without JSX, types) |
| `.tsx`    | TypeScript files that contain JSX |

---

## 2. Typing Functional Components

There are two main approaches to typing React functional components. Understanding the difference helps you write cleaner, more intentional code.

### Approach 1: `React.FC` (Functional Component)

`React.FC` is a generic type provided by `@types/react`. It explicitly declares that a variable is a React functional component.

```tsx
import React from 'react';

// React.FC<Props> — explicit component typing
const Greeting: React.FC<{ name: string }> = ({ name }) => {
  return <h1>Hello, {name}!</h1>;
};
```

**What `React.FC` gives you:**
- Automatically types the return value as `React.ReactElement | null`
- Provides type checking on the component's props

**What `React.FC` does NOT give you (post-React 18):**
- It no longer implicitly includes `children` in props (this was removed in React 18)

### Approach 2: Explicit Return Type (Recommended)

The modern, preferred approach is to type your props explicitly and annotate the return type directly:

```tsx
// Explicit return type — clear and precise
function Greeting({ name }: { name: string }): React.ReactElement {
  return <h1>Hello, {name}!</h1>;
};

// Or with JSX.Element (equivalent in most cases)
function Greeting({ name }: { name: string }): JSX.Element {
  return <h1>Hello, {name}!</h1>;
};
```

### Return Type Options Compared

| Type | Meaning |
|------|---------|
| `JSX.Element` | A single JSX element (cannot return `null`) |
| `React.ReactElement` | Essentially the same as `JSX.Element` |
| `React.ReactNode` | JSX, string, number, `null`, `undefined`, array — the widest type |

```tsx
// JSX.Element — when you always return markup
function Button(): JSX.Element {
  return <button>Click me</button>;
}

// React.ReactNode — when the component might return null or conditionals
function ConditionalBanner({ show }: { show: boolean }): React.ReactNode {
  if (!show) return null;  // ✅ allowed with ReactNode
  return <div className="banner">Important notice!</div>;
}
```

### `React.FC` vs Explicit Return Type — Side by Side

```tsx
// ─── React.FC approach ───────────────────────────────────────
const UserCard: React.FC<{ username: string; age: number }> = ({ username, age }) => {
  return (
    <div>
      <p>{username}</p>
      <p>{age} years old</p>
    </div>
  );
};

// ─── Explicit return type approach ───────────────────────────
interface UserCardProps {
  username: string;
  age: number;
}

function UserCard({ username, age }: UserCardProps): JSX.Element {
  return (
    <div>
      <p>{username}</p>
      <p>{age} years old</p>
    </div>
  );
}
```

> **Recommendation:** Prefer the explicit return type approach. It's more readable, avoids the historical pitfalls of `React.FC`, and aligns with the official React TypeScript Cheatsheet recommendations.

---

## 3. Props Interfaces and Optional Props

Defining prop types with interfaces makes your components self-documenting and prevents entire classes of runtime errors.

### Defining a Props Interface

```tsx
// Define the interface above the component
interface ButtonProps {
  label: string;           // required
  onClick: () => void;     // required function
  variant: 'primary' | 'secondary' | 'danger';  // required union type
}

function Button({ label, onClick, variant }: ButtonProps): JSX.Element {
  return (
    <button className={`btn btn-${variant}`} onClick={onClick}>
      {label}
    </button>
  );
}
```

### Optional Props with `?`

Mark a prop optional using `?`. TypeScript will infer its type as `T | undefined`.

```tsx
interface CardProps {
  title: string;           // required
  subtitle?: string;       // optional — string | undefined
  imageUrl?: string;       // optional
  isHighlighted?: boolean; // optional
}

function Card({ title, subtitle, imageUrl, isHighlighted = false }: CardProps): JSX.Element {
  //                                                           ↑
  //                                  Default value for optional boolean
  return (
    <div className={isHighlighted ? 'card card--highlighted' : 'card'}>
      {imageUrl && <img src={imageUrl} alt={title} />}
      <h2>{title}</h2>
      {subtitle && <p>{subtitle}</p>}
    </div>
  );
}

// Usage
<Card title="TypeScript Tips" />
<Card title="TypeScript Tips" subtitle="Week 6" isHighlighted />
```

### Default Values

Two ways to handle defaults for optional props:

```tsx
// ── Method 1: Destructuring defaults (preferred) ──────────────
interface ToastProps {
  message: string;
  duration?: number;
  position?: 'top' | 'bottom';
}

function Toast({
  message,
  duration = 3000,
  position = 'top',
}: ToastProps): JSX.Element {
  return (
    <div className={`toast toast--${position}`}>
      {message} (disappears in {duration}ms)
    </div>
  );
}

// ── Method 2: defaultProps (older pattern, avoid in new code) ──
Toast.defaultProps = {
  duration: 3000,
  position: 'top',
};
```

### Extending HTML Element Props

A powerful pattern is extending native HTML element attributes so your component accepts all standard HTML props:

```tsx
// Extend native button attributes
interface CustomButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  isLoading?: boolean;
  variant?: 'primary' | 'outline';
}

function CustomButton({
  isLoading = false,
  variant = 'primary',
  children,
  disabled,
  ...rest          // all other native button props (onClick, type, aria-*, etc.)
}: CustomButtonProps): JSX.Element {
  return (
    <button
      className={`btn btn-${variant}`}
      disabled={disabled || isLoading}
      {...rest}
    >
      {isLoading ? 'Loading...' : children}
    </button>
  );
}

// Usage — accepts all native button attributes automatically
<CustomButton
  onClick={() => console.log('clicked')}
  type="submit"
  aria-label="Submit form"
  variant="primary"
>
  Submit
</CustomButton>
```

---

## 4. Children Prop Typing

The `children` prop is special in React. TypeScript requires you to be explicit about what kind of children a component accepts.

### `React.ReactNode` — The Most Permissive Type

`React.ReactNode` accepts virtually anything: JSX, strings, numbers, arrays, `null`, `undefined`, booleans.

```tsx
interface ContainerProps {
  children: React.ReactNode;   // widest — accepts almost anything
  className?: string;
}

function Container({ children, className }: ContainerProps): JSX.Element {
  return <div className={className}>{children}</div>;
}

// All of these are valid
<Container>Hello</Container>
<Container><p>Paragraph</p></Container>
<Container>{null}</Container>
<Container>{isVisible && <Alert />}</Container>
```

### `React.ReactElement` — Single JSX Element Only

`React.ReactElement` is stricter — it only accepts a JSX element (not strings, numbers, or `null`).

```tsx
interface TooltipProps {
  children: React.ReactElement;  // must be a single JSX element
  tooltipText: string;
}

function Tooltip({ children, tooltipText }: TooltipProps): JSX.Element {
  return (
    <div className="tooltip-wrapper">
      {children}
      <span className="tooltip-text">{tooltipText}</span>
    </div>
  );
}

// ✅ Valid
<Tooltip tooltipText="More info"><button>Hover me</button></Tooltip>

// ❌ TypeScript error
<Tooltip tooltipText="More info">Just a string</Tooltip>
```

### Comparison Table

| Type | Accepts |
|------|---------|
| `React.ReactNode` | JSX, string, number, null, undefined, boolean, arrays |
| `React.ReactElement` | JSX element only (e.g., `<div>`, `<MyComponent>`) |
| `JSX.Element` | Same as `React.ReactElement` |
| `string` | Text content only |
| `never` | No children allowed |

### Requiring Exactly One Child

```tsx
import React from 'react';

interface CloneWrapperProps {
  children: React.ReactElement;   // enforces a single element
}

function CloneWrapper({ children }: CloneWrapperProps): JSX.Element {
  // Can safely clone the single element and add extra props
  return React.cloneElement(children, { className: 'wrapped' });
}
```

### PropsWithChildren Utility Type

`React.PropsWithChildren<T>` is a utility that adds `children?: React.ReactNode` to any props type — useful for quick one-liners:

```tsx
type SectionProps = React.PropsWithChildren<{
  title: string;
}>;

function Section({ title, children }: SectionProps): JSX.Element {
  return (
    <section>
      <h2>{title}</h2>
      {children}
    </section>
  );
}
```

---

## 5. Typing useState

`useState` is a generic hook. TypeScript can often infer the type from the initial value, but explicit typing is recommended when the initial value is `null`, `undefined`, or a complex object.

### Basic Inference

When the initial value is clear, TypeScript infers the type automatically:

```tsx
function Counter(): JSX.Element {
  const [count, setCount] = useState(0);           // inferred: number
  const [name, setName] = useState('');            // inferred: string
  const [isOpen, setIsOpen] = useState(false);     // inferred: boolean

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}
```

### Explicit Generic Typing

Use explicit generics when the initial value is `null`, `undefined`, or when inference would be too narrow:

```tsx
// State that starts as null but will become a string
const [selectedId, setSelectedId] = useState<string | null>(null);

// State that starts as undefined
const [user, setUser] = useState<User | undefined>(undefined);

// Array state — without explicit typing, TS infers never[]
const [items, setItems] = useState<string[]>([]);
const [users, setUsers] = useState<User[]>([]);
```

### Object State

For complex object state, define an interface first:

```tsx
interface FormData {
  email: string;
  password: string;
  rememberMe: boolean;
}

function LoginForm(): JSX.Element {
  const [formData, setFormData] = useState<FormData>({
    email: '',
    password: '',
    rememberMe: false,
  });

  // Partial update using spread
  const handleChange = (field: keyof FormData, value: string | boolean) => {
    setFormData(prev => ({ ...prev, [field]: value }));
  };

  return (
    <form>
      <input
        value={formData.email}
        onChange={e => handleChange('email', e.target.value)}
        placeholder="Email"
      />
      <input
        type="password"
        value={formData.password}
        onChange={e => handleChange('password', e.target.value)}
        placeholder="Password"
      />
      <label>
        <input
          type="checkbox"
          checked={formData.rememberMe}
          onChange={e => handleChange('rememberMe', e.target.checked)}
        />
        Remember me
      </label>
    </form>
  );
}
```

### Union Type State (Discriminated Unions)

A powerful pattern for state machines and loading states:

```tsx
type FetchState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; message: string };

function UserProfile({ userId }: { userId: string }): JSX.Element {
  const [state, setState] = useState<FetchState<User>>({ status: 'idle' });

  // TypeScript narrows the type based on status
  if (state.status === 'loading') return <Spinner />;
  if (state.status === 'error') return <ErrorMessage message={state.message} />;
  if (state.status === 'success') return <UserCard user={state.data} />;
  return <button onClick={fetchUser}>Load Profile</button>;
}
```

---

## 6. Typing useRef

`useRef` serves two distinct purposes in React, and TypeScript types them differently.

### Purpose 1: DOM References

Use `useRef` to hold a reference to a DOM element. The generic type is the DOM element type.

```tsx
function TextInput(): JSX.Element {
  // Type is HTMLInputElement — matches the element we attach it to
  const inputRef = useRef<HTMLInputElement>(null);

  const focusInput = () => {
    // TypeScript knows inputRef.current could be null (before mount)
    // Use optional chaining to be safe
    inputRef.current?.focus();
  };

  return (
    <div>
      <input ref={inputRef} type="text" placeholder="Click button to focus" />
      <button onClick={focusInput}>Focus Input</button>
    </div>
  );
}
```

> **Why `null` as initial value?** When the component first renders, the DOM element doesn't exist yet, so the ref starts as `null`. React populates it after the element mounts.

### Common DOM Element Types

```tsx
const divRef    = useRef<HTMLDivElement>(null);
const inputRef  = useRef<HTMLInputElement>(null);
const btnRef    = useRef<HTMLButtonElement>(null);
const formRef   = useRef<HTMLFormElement>(null);
const canvasRef = useRef<HTMLCanvasElement>(null);
const videoRef  = useRef<HTMLVideoElement>(null);
const imgRef    = useRef<HTMLImageElement>(null);
const ulRef     = useRef<HTMLUListElement>(null);
```

### Purpose 2: Mutable Value Container (No Re-render)

`useRef` can also hold any mutable value that persists across renders without triggering a re-render. Use a non-null initial value in this case:

```tsx
function Timer(): JSX.Element {
  const [seconds, setSeconds] = useState(0);
  // intervalId doesn't need to be null — initialize with a sentinel value
  const intervalRef = useRef<ReturnType<typeof setInterval> | null>(null);

  const startTimer = () => {
    intervalRef.current = setInterval(() => {
      setSeconds(prev => prev + 1);
    }, 1000);
  };

  const stopTimer = () => {
    if (intervalRef.current) {
      clearInterval(intervalRef.current);
      intervalRef.current = null;
    }
  };

  return (
    <div>
      <p>Elapsed: {seconds}s</p>
      <button onClick={startTimer}>Start</button>
      <button onClick={stopTimer}>Stop</button>
    </div>
  );
}
```

### Tracking Previous Values

A classic use case — tracking the previous render's value:

```tsx
function useTrackPrevious<T>(value: T): T | undefined {
  const prevRef = useRef<T | undefined>(undefined);

  // After render, store current value for next render
  useEffect(() => {
    prevRef.current = value;
  });

  return prevRef.current;
}

// Usage
function PriceDisplay({ price }: { price: number }): JSX.Element {
  const prevPrice = useTrackPrevious(price);

  return (
    <div>
      <p>Current: ${price}</p>
      {prevPrice !== undefined && (
        <p>Previous: ${prevPrice}</p>
      )}
    </div>
  );
}
```

---

## 7. Typing Event Handlers

React wraps native DOM events in its own synthetic event system. TypeScript provides specific generic types for each event type.

### Core Event Types

```tsx
// The pattern: React.EventType<HTMLElementType>
React.ChangeEvent<HTMLInputElement>      // input, textarea, select changes
React.MouseEvent<HTMLButtonElement>      // mouse click, hover, etc.
React.FormEvent<HTMLFormElement>         // form submission
React.KeyboardEvent<HTMLInputElement>    // keyboard events
React.FocusEvent<HTMLInputElement>       // focus/blur events
React.DragEvent<HTMLDivElement>          // drag and drop
React.WheelEvent<HTMLDivElement>         // scroll wheel
```

### Input & Change Events

```tsx
function SearchBox(): JSX.Element {
  const [query, setQuery] = useState('');

  // React.ChangeEvent<HTMLInputElement> gives you access to e.target.value
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>): void => {
    setQuery(e.target.value);
  };

  return <input value={query} onChange={handleChange} placeholder="Search..." />;
}
```

For **textarea** and **select**, change only the element type:

```tsx
// Textarea
const handleTextArea = (e: React.ChangeEvent<HTMLTextAreaElement>): void => {
  console.log(e.target.value);
};

// Select dropdown
const handleSelect = (e: React.ChangeEvent<HTMLSelectElement>): void => {
  console.log(e.target.value);   // selected option's value
};
```

### Mouse Events

```tsx
function InteractiveCard(): JSX.Element {
  const handleClick = (e: React.MouseEvent<HTMLDivElement>): void => {
    e.stopPropagation();  // stop event bubbling
    console.log(`Clicked at ${e.clientX}, ${e.clientY}`);
  };

  const handleRightClick = (e: React.MouseEvent<HTMLDivElement>): void => {
    e.preventDefault();  // prevent context menu
    console.log('Right-clicked!');
  };

  return (
    <div onClick={handleClick} onContextMenu={handleRightClick}>
      Click or right-click me
    </div>
  );
}
```

### Form Submission

```tsx
interface LoginFields {
  email: string;
  password: string;
}

function LoginForm(): JSX.Element {
  const [fields, setFields] = useState<LoginFields>({ email: '', password: '' });

  // React.FormEvent<HTMLFormElement>
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>): void => {
    e.preventDefault();  // prevent page reload
    console.log('Submitting:', fields);
  };

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>): void => {
    const { name, value } = e.target;
    setFields(prev => ({ ...prev, [name]: value }));
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="email" value={fields.email} onChange={handleChange} />
      <input name="password" type="password" value={fields.password} onChange={handleChange} />
      <button type="submit">Login</button>
    </form>
  );
}
```

### Keyboard Events

```tsx
function CommandInput(): JSX.Element {
  const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>): void => {
    if (e.key === 'Enter') {
      console.log('Enter pressed');
    }
    if (e.key === 'Escape') {
      e.currentTarget.blur();
    }
    if (e.ctrlKey && e.key === 's') {
      e.preventDefault();
      console.log('Ctrl+S — save triggered');
    }
  };

  return <input onKeyDown={handleKeyDown} placeholder="Type a command..." />;
}
```

### Inline Handlers vs Named Handlers

Both approaches are valid — inline is concise for simple cases, named functions are cleaner for complex logic:

```tsx
function Example(): JSX.Element {
  const [value, setValue] = useState('');

  // Named handler — preferred for complex logic
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>): void => {
    setValue(e.target.value.trim());
  };

  return (
    <div>
      {/* Named handler */}
      <input onChange={handleChange} value={value} />

      {/* Inline handler — TypeScript infers the event type automatically */}
      <input onChange={e => setValue(e.target.value)} />

      {/* Button with mouse event */}
      <button onClick={(e: React.MouseEvent<HTMLButtonElement>) => console.log(e.type)}>
        Click
      </button>
    </div>
  );
}
```

---

## 8. Generic Components

Generic components are one of TypeScript's most powerful features for React. They let you write reusable components that work with any data type while maintaining full type safety.

### The Problem Generics Solve

Without generics, you'd either use `any` (unsafe) or write duplicate components for each data type:

```tsx
// ❌ Without generics — forced to use any, loses type safety
function List({ items }: { items: any[] }) {
  return <ul>{items.map(item => <li key={item.id}>{item.name}</li>)}</ul>;
}

// ❌ Or duplicate code for every type
function UserList({ users }: { users: User[] }) { ... }
function ProductList({ products }: { products: Product[] }) { ... }
```

### Basic Generic Component

```tsx
// ✅ Generic component — works with any type T
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
  keyExtractor: (item: T) => string | number;
}

function List<T>({ items, renderItem, keyExtractor }: ListProps<T>): JSX.Element {
  return (
    <ul>
      {items.map(item => (
        <li key={keyExtractor(item)}>
          {renderItem(item)}
        </li>
      ))}
    </ul>
  );
}

// Usage — TypeScript infers T from the items array
interface User { id: number; name: string; email: string; }
interface Product { sku: string; title: string; price: number; }

const users: User[] = [{ id: 1, name: 'Alice', email: 'alice@example.com' }];
const products: Product[] = [{ sku: 'A1', title: 'Widget', price: 9.99 }];

// T is inferred as User
<List
  items={users}
  keyExtractor={user => user.id}
  renderItem={user => <span>{user.name} — {user.email}</span>}
/>

// T is inferred as Product
<List
  items={products}
  keyExtractor={product => product.sku}
  renderItem={product => <span>{product.title} — ${product.price}</span>}
/>
```

### Generic Select/Dropdown Component

```tsx
interface SelectProps<T> {
  options: T[];
  value: T | null;
  onChange: (value: T) => void;
  getLabel: (option: T) => string;
  getValue: (option: T) => string | number;
  placeholder?: string;
}

function Select<T>({
  options,
  value,
  onChange,
  getLabel,
  getValue,
  placeholder = 'Select an option',
}: SelectProps<T>): JSX.Element {
  const handleChange = (e: React.ChangeEvent<HTMLSelectElement>): void => {
    const selectedValue = e.target.value;
    const found = options.find(opt => String(getValue(opt)) === selectedValue);
    if (found !== undefined) onChange(found);
  };

  return (
    <select
      value={value !== null ? String(getValue(value)) : ''}
      onChange={handleChange}
    >
      <option value="" disabled>{placeholder}</option>
      {options.map(option => (
        <option key={getValue(option)} value={String(getValue(option))}>
          {getLabel(option)}
        </option>
      ))}
    </select>
  );
}

// Usage with different data shapes
<Select
  options={users}
  value={selectedUser}
  onChange={setSelectedUser}
  getLabel={u => u.name}
  getValue={u => u.id}
/>
```

### Generic with Constraints

Use `extends` to constrain what types T can be:

```tsx
// T must have at least an id and name property
interface WithId {
  id: number | string;
}

interface TableProps<T extends WithId> {
  data: T[];
  columns: Array<{
    key: keyof T;
    header: string;
  }>;
}

function Table<T extends WithId>({ data, columns }: TableProps<T>): JSX.Element {
  return (
    <table>
      <thead>
        <tr>
          {columns.map(col => (
            <th key={String(col.key)}>{col.header}</th>
          ))}
        </tr>
      </thead>
      <tbody>
        {data.map(row => (
          <tr key={row.id}>
            {columns.map(col => (
              <td key={String(col.key)}>{String(row[col.key])}</td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  );
}

// Usage
<Table
  data={users}
  columns={[
    { key: 'name', header: 'Name' },
    { key: 'email', header: 'Email' },
  ]}
/>
```

### Generic Custom Hook

Generics apply to custom hooks too — a critical pattern for reusable logic:

```tsx
interface UseFetchResult<T> {
  data: T | null;
  loading: boolean;
  error: string | null;
}

function useFetch<T>(url: string): UseFetchResult<T> {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const controller = new AbortController();

    fetch(url, { signal: controller.signal })
      .then(res => {
        if (!res.ok) throw new Error(`HTTP error: ${res.status}`);
        return res.json() as Promise<T>;
      })
      .then(setData)
      .catch(err => {
        if (err.name !== 'AbortError') setError(err.message);
      })
      .finally(() => setLoading(false));

    return () => controller.abort();
  }, [url]);

  return { data, loading, error };
}

// Usage — TypeScript knows the shape of data
interface Post { id: number; title: string; body: string; }

function PostList(): JSX.Element {
  const { data: posts, loading, error } = useFetch<Post[]>(
    'https://jsonplaceholder.typicode.com/posts'
  );

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error}</p>;
  if (!posts) return <p>No posts found</p>;

  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

---

## 9. Weekly Challenge

Build a **TypeScript-typed Task Manager** that exercises every concept from this week. Requirements:

### Requirements

1. **Setup** — Create a new Vite + React + TypeScript project
2. **Types** — Define a `Task` interface with: `id`, `title`, `description?`, `priority` (`'low' | 'medium' | 'high'`), `completed`, `createdAt`
3. **Components** — Build these typed components:
   - `TaskList<T>` — generic list component
   - `TaskCard` — displays a task, accepts full `Task` as props with optional `onDelete` callback
   - `TaskForm` — controlled form with typed `onChange` and `onSubmit` handlers
   - `PriorityBadge` — renders color-coded badge, uses union type prop
4. **Hooks** — Use `useState<Task[]>`, `useRef<HTMLInputElement>` for auto-focus
5. **Events** — Type all event handlers explicitly (no implicit `any`)

### Bonus Challenges

- Build a generic `useLocalStorage<T>` hook that persists state to localStorage
- Create a `FilterBar` component using `React.FC` and compare the developer experience with the explicit return type approach
- Add a generic `Pagination<T>` component that slices any array

### Self-Check Questions

Before moving to Week 7, make sure you can answer:

- When should you choose `React.ReactNode` over `React.ReactElement` for `children`?
- What's the difference between `useRef<HTMLInputElement>(null)` and `useRef<number>(0)`?
- Why is `useState<string[]>([])` preferable to just `useState([])`?
- How do generic constraints (`T extends SomeType`) improve type safety?
- When is `React.FC` appropriate vs an explicit return type annotation?

---

## Quick Reference Card

```typescript
// ── Component Typing ──────────────────────────────────────────
function MyComp({ prop }: Props): JSX.Element { ... }          // preferred
const MyComp: React.FC<Props> = ({ prop }) => { ... }          // alternative

// ── Props ─────────────────────────────────────────────────────
interface Props {
  required: string
  optional?: number
  union: 'a' | 'b' | 'c'
  callback: (value: string) => void
  children: React.ReactNode                                     // any children
  children: React.ReactElement                                  // single JSX only
}

// ── useState ─────────────────────────────────────────────────
const [val, setVal] = useState(0)                              // inferred
const [val, setVal] = useState<string | null>(null)            // explicit
const [items, setItems] = useState<Item[]>([])                 // array

// ── useRef ───────────────────────────────────────────────────
const domRef = useRef<HTMLInputElement>(null)                   // DOM element
const valRef = useRef<number>(0)                               // mutable value

// ── Event Handlers ────────────────────────────────────────────
(e: React.ChangeEvent<HTMLInputElement>) => void               // input change
(e: React.MouseEvent<HTMLButtonElement>) => void               // button click
(e: React.FormEvent<HTMLFormElement>) => void                  // form submit
(e: React.KeyboardEvent<HTMLInputElement>) => void             // key event

// ── Generic Components ────────────────────────────────────────
function Comp<T>({ items }: { items: T[] }): JSX.Element       // basic
function Comp<T extends { id: number }>(...): JSX.Element      // constrained
```

---

*Next up → **Week 7: Advanced Hooks & Context with TypeScript** — typing useReducer, useContext, createContext, and custom hook patterns.*
