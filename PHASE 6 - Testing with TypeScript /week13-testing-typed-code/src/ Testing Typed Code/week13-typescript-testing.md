# Week 13: Testing Typed Code

> A complete guide to writing robust, type-safe tests in TypeScript using Vitest, Jest, React Testing Library, MSW, and more.

---

## Table of Contents

1. [Vitest / Jest with TypeScript](#1-vitest--jest-with-typescript)
2. [Typing Mocks — `vi.fn()` / `jest.fn()`](#2-typing-mocks--vifn--jestfn)
3. [Type-Safe Test Factories](#3-type-safe-test-factories)
4. [Testing Generic Functions](#4-testing-generic-functions)
5. [React Testing Library with TypeScript](#5-react-testing-library-with-typescript)
6. [Typing MSW Handlers](#6-typing-msw-mock-service-worker-handlers)
7. [Testing Custom Hooks with Typed Return Values](#7-testing-custom-hooks-with-typed-return-values)
8. [Type-Level Testing with `expect-type`](#8-type-level-testing-with-expect-type)
9. [Putting It All Together](#9-putting-it-all-together)

---

## 1. Vitest / Jest with TypeScript

### Why TypeScript + Testing Frameworks?

TypeScript adds a compile-time safety net, but you still need runtime tests to verify logic. The goal is **type-safe tests** — tests that catch both logic bugs *and* type contract violations.

---

### Setting Up Vitest with TypeScript

Vitest is the preferred test runner for Vite-based projects. It supports TypeScript out of the box via `esbuild`.

```bash
npm install -D vitest @vitest/ui
```

**`vite.config.ts`**

```typescript
import { defineConfig } from 'vite';

export default defineConfig({
  test: {
    globals: true,         // Makes describe/it/expect globally available
    environment: 'jsdom',  // For DOM/React tests
    setupFiles: './src/test/setup.ts',
  },
});
```

**`tsconfig.json` (test-aware)**

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "types": ["vitest/globals"]
  },
  "include": ["src"]
}
```

---

### Setting Up Jest with TypeScript

```bash
npm install -D jest ts-jest @types/jest
```

**`jest.config.ts`**

```typescript
import type { Config } from 'jest';

const config: Config = {
  preset: 'ts-jest',
  testEnvironment: 'jsdom',
  setupFilesAfterFramework: ['<rootDir>/src/test/setup.ts'],
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1', // Path alias support
  },
};

export default config;
```

---

### Your First Typed Test

```typescript
// src/utils/math.ts
export function add(a: number, b: number): number {
  return a + b;
}

export function divide(a: number, b: number): number {
  if (b === 0) throw new Error('Division by zero');
  return a / b;
}
```

```typescript
// src/utils/math.test.ts
import { describe, it, expect } from 'vitest';
import { add, divide } from './math';

describe('add', () => {
  it('returns the sum of two numbers', () => {
    const result: number = add(2, 3);
    expect(result).toBe(5);
  });
});

describe('divide', () => {
  it('divides correctly', () => {
    expect(divide(10, 2)).toBe(5);
  });

  it('throws on division by zero', () => {
    expect(() => divide(10, 0)).toThrow('Division by zero');
  });
});
```

> **Key insight:** TypeScript checks that `add(2, 3)` is valid *at compile time*. The test checks the runtime value. Both are needed.

---

### Typing `describe` / `it` Callbacks

When using `globals: false`, import explicitly:

```typescript
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
```

When using `globals: true`, they're injected automatically. TypeScript picks them up from `"types": ["vitest/globals"]` in `tsconfig.json`.

---

## 2. Typing Mocks — `vi.fn()` / `jest.fn()`

### The Problem with Untyped Mocks

```typescript
// ❌ No type safety — TypeScript has no idea what this mock accepts or returns
const mockFn = vi.fn();
mockFn(42, 'oops'); // No error, but your real function might only take (number)
```

### Typing Mocks with Generics

`vi.fn()` and `jest.fn()` accept a generic parameter representing the function signature:

```typescript
// ✅ Typed mock
const mockAdd = vi.fn<[number, number], number>();

mockAdd(1, 2);       // OK
mockAdd('a', 'b');   // ❌ TypeScript Error: Argument of type 'string' is not assignable to 'number'
```

The generic is `MockedFunction<(args) => returnType>`:

```typescript
import { vi, type MockedFunction } from 'vitest';

type FetchUser = (id: string) => Promise<{ name: string; age: number }>;

const mockFetchUser: MockedFunction<FetchUser> = vi.fn();

// Now TypeScript enforces both argument and return types
mockFetchUser.mockResolvedValue({ name: 'Alice', age: 30 }); // ✅
mockFetchUser.mockResolvedValue({ name: 'Alice' });           // ❌ Missing 'age'
```

---

### `vi.spyOn()` — Type-Safe Spies

`spyOn` preserves the original function's type automatically:

```typescript
import * as mathUtils from './math';

const spy = vi.spyOn(mathUtils, 'add');

// spy is automatically typed as SpyInstance<[number, number], number>
spy.mockReturnValue(100);
spy.mockReturnValue('oops'); // ❌ TypeScript Error
```

---

### Mocking Modules

```typescript
// src/services/api.ts
export async function getUser(id: string): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
}
```

```typescript
// src/services/api.test.ts
import { vi, describe, it, expect, beforeEach } from 'vitest';
import { getUser } from './api';

// Tell TypeScript this module is mocked
vi.mock('./api');

// Cast to access mock-specific methods
const mockGetUser = getUser as vi.MockedFunction<typeof getUser>;

describe('getUser', () => {
  beforeEach(() => {
    mockGetUser.mockClear();
  });

  it('returns a user', async () => {
    mockGetUser.mockResolvedValue({ id: '1', name: 'Alice' });
    const user = await getUser('1');
    expect(user.name).toBe('Alice');
  });
});
```

---

### Mock Return Value Helpers

| Helper | Description |
|---|---|
| `.mockReturnValue(val)` | Always returns `val` (sync) |
| `.mockResolvedValue(val)` | Always resolves to `val` (async) |
| `.mockRejectedValue(err)` | Always rejects with `err` (async) |
| `.mockReturnValueOnce(val)` | Returns `val` only on the next call |
| `.mockImplementation(fn)` | Full custom implementation |
| `.mockImplementationOnce(fn)` | Custom impl for next call only |

```typescript
const mockFetch = vi.fn<[string], Promise<Response>>();

// Different values per call
mockFetch
  .mockResolvedValueOnce(new Response(JSON.stringify({ id: 1 })))
  .mockResolvedValueOnce(new Response(JSON.stringify({ id: 2 })))
  .mockRejectedValueOnce(new Error('Network error'));
```

---

### Asserting Mock Calls with Types

```typescript
const mockFn = vi.fn<[userId: string, options: { cache: boolean }], void>();

mockFn('user-1', { cache: true });

// Type-safe access to call arguments
const [firstArg, secondArg] = mockFn.mock.calls[0];
//     ^string           ^{ cache: boolean }

expect(firstArg).toBe('user-1');
expect(secondArg.cache).toBe(true);
```

---

## 3. Type-Safe Test Factories

### What Are Test Factories?

Test factories create pre-configured test data with sensible defaults, reducing boilerplate and keeping tests focused on what actually varies.

### The Problem Without Factories

```typescript
// ❌ Verbose — every test must specify all fields even if they don't matter
it('displays user name', () => {
  const user: User = {
    id: 'abc',
    name: 'Alice',
    email: 'alice@example.com',
    role: 'admin',
    createdAt: new Date('2023-01-01'),
    isActive: true,
    // ...many more fields
  };
  render(<UserCard user={user} />);
  expect(screen.getByText('Alice')).toBeInTheDocument();
});
```

### Building a Typed Factory

```typescript
// src/test/factories/user.factory.ts
import type { User } from '../types';

// DeepPartial utility type — makes all nested fields optional
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P];
};

let idCounter = 0;

export function createUser(overrides: DeepPartial<User> = {}): User {
  idCounter++;
  return {
    id: `user-${idCounter}`,
    name: 'Test User',
    email: `test${idCounter}@example.com`,
    role: 'viewer',
    createdAt: new Date('2024-01-01'),
    isActive: true,
    ...overrides,
  };
}
```

```typescript
// Usage in tests
it('shows admin badge for admin users', () => {
  const admin = createUser({ role: 'admin' });
  //    ^User — fully typed
  render(<UserCard user={admin} />);
  expect(screen.getByText('Admin')).toBeInTheDocument();
});
```

### Factory with Nested Objects

```typescript
// src/types.ts
interface Address {
  street: string;
  city: string;
  country: string;
}

interface User {
  id: string;
  name: string;
  address: Address;
}
```

```typescript
// src/test/factories/user.factory.ts
export function createAddress(overrides: Partial<Address> = {}): Address {
  return {
    street: '123 Test Lane',
    city: 'Testville',
    country: 'US',
    ...overrides,
  };
}

export function createUser(overrides: DeepPartial<User> = {}): User {
  return {
    id: crypto.randomUUID(),
    name: 'Test User',
    address: createAddress(overrides.address),
    ...overrides,
  };
}

// Test usage
const user = createUser({ address: { city: 'New York' } });
// user.address.street is still '123 Test Lane' — only city is overridden
```

### Factory Arrays and Sequences

```typescript
export function createUsers(count: number, overrides: DeepPartial<User> = {}): User[] {
  return Array.from({ length: count }, (_, i) =>
    createUser({ name: `User ${i + 1}`, ...overrides })
  );
}

// Create 5 admin users
const adminUsers = createUsers(5, { role: 'admin' });
```

### Using Builder Pattern for Complex Objects

```typescript
class UserBuilder {
  private user: User;

  constructor() {
    this.user = createUser();
  }

  withRole(role: User['role']): this {
    this.user.role = role;
    return this;
  }

  inactive(): this {
    this.user.isActive = false;
    return this;
  }

  build(): User {
    return { ...this.user };
  }
}

// Clean, readable test setup
const inactiveAdmin = new UserBuilder().withRole('admin').inactive().build();
```

---

## 4. Testing Generic Functions

### The Challenge with Generics

Generic functions behave differently depending on the type argument. Tests must cover multiple type instantiations.

### Example: Testing a Generic `first<T>` Function

```typescript
// src/utils/array.ts
export function first<T>(arr: T[]): T | undefined {
  return arr[0];
}

export function chunk<T>(arr: T[], size: number): T[][] {
  const result: T[][] = [];
  for (let i = 0; i < arr.length; i += size) {
    result.push(arr.slice(i, i + size));
  }
  return result;
}
```

```typescript
// src/utils/array.test.ts
import { describe, it, expect } from 'vitest';
import { first, chunk } from './array';

describe('first<T>', () => {
  it('returns first element from number array', () => {
    const result = first([1, 2, 3]);
    // result is inferred as: number | undefined
    expect(result).toBe(1);
  });

  it('returns first element from string array', () => {
    const result = first(['a', 'b', 'c']);
    // result is inferred as: string | undefined
    expect(result).toBe('a');
  });

  it('returns first element from object array', () => {
    const items = [{ id: 1, name: 'Alice' }, { id: 2, name: 'Bob' }];
    const result = first(items);
    // result is inferred as: { id: number; name: string } | undefined
    expect(result?.id).toBe(1);
  });

  it('returns undefined for empty array', () => {
    const result = first<number>([]);
    expect(result).toBeUndefined();
  });
});

describe('chunk<T>', () => {
  it('chunks a number array', () => {
    const result = chunk([1, 2, 3, 4, 5], 2);
    //    ^number[][]
    expect(result).toEqual([[1, 2], [3, 4], [5]]);
  });

  it('chunks an array of objects', () => {
    const users = [
      { id: 1 }, { id: 2 }, { id: 3 }
    ];
    const result = chunk(users, 2);
    //    ^{ id: number }[][]
    expect(result[0]).toHaveLength(2);
    expect(result[1]).toHaveLength(1);
  });
});
```

### Testing Generic Classes

```typescript
// src/utils/stack.ts
export class Stack<T> {
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

  get size(): number {
    return this.items.length;
  }

  isEmpty(): boolean {
    return this.items.length === 0;
  }
}
```

```typescript
// src/utils/stack.test.ts
describe('Stack<T>', () => {
  describe('with numbers', () => {
    let stack: Stack<number>;

    beforeEach(() => {
      stack = new Stack<number>();
    });

    it('pushes and pops numbers', () => {
      stack.push(1);
      stack.push(2);
      expect(stack.pop()).toBe(2);
      expect(stack.pop()).toBe(1);
    });
  });

  describe('with complex objects', () => {
    interface Task { id: string; priority: number }
    let stack: Stack<Task>;

    beforeEach(() => {
      stack = new Stack<Task>();
    });

    it('maintains LIFO order for tasks', () => {
      stack.push({ id: 'task-1', priority: 1 });
      stack.push({ id: 'task-2', priority: 2 });
      
      const top = stack.pop();
      expect(top?.id).toBe('task-2');
      expect(top?.priority).toBe(2);
    });
  });
});
```

### Testing Generic Constraints

```typescript
// src/utils/sort.ts
export function sortBy<T, K extends keyof T>(items: T[], key: K): T[] {
  return [...items].sort((a, b) => {
    if (a[key] < b[key]) return -1;
    if (a[key] > b[key]) return 1;
    return 0;
  });
}
```

```typescript
describe('sortBy<T, K>', () => {
  it('sorts objects by a given key', () => {
    const users = [
      { name: 'Charlie', age: 30 },
      { name: 'Alice', age: 25 },
      { name: 'Bob', age: 28 },
    ];

    const byName = sortBy(users, 'name');
    //             ^— TypeScript ensures 'name' is a key of typeof users[number]
    expect(byName[0].name).toBe('Alice');

    const byAge = sortBy(users, 'age');
    expect(byAge[0].age).toBe(25);

    // ❌ This would be a compile-time error:
    // sortBy(users, 'nonExistentKey');
  });
});
```

---

## 5. React Testing Library with TypeScript

### Setup

```bash
npm install -D @testing-library/react @testing-library/jest-dom @testing-library/user-event
```

**`src/test/setup.ts`**

```typescript
import '@testing-library/jest-dom';
// Adds custom matchers like toBeInTheDocument(), toHaveValue(), etc.
```

**`tsconfig.json`** (add jest-dom types)

```json
{
  "compilerOptions": {
    "types": ["vitest/globals", "@testing-library/jest-dom"]
  }
}
```

---

### Typing `render` Results

```typescript
import { render, screen } from '@testing-library/react';
import type { RenderResult } from '@testing-library/react';

// RenderResult includes: container, baseElement, debug, unmount, rerender, etc.
const result: RenderResult = render(<MyComponent />);

// Most of the time, you can just use screen directly
render(<MyComponent />);
const heading = screen.getByRole('heading', { name: /welcome/i });
//    ^HTMLElement
```

---

### Typing Props in Component Tests

```typescript
// src/components/UserCard.tsx
interface UserCardProps {
  user: {
    id: string;
    name: string;
    email: string;
    role: 'admin' | 'viewer';
  };
  onDelete?: (id: string) => void;
}

export function UserCard({ user, onDelete }: UserCardProps) {
  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
      {user.role === 'admin' && <span>Admin</span>}
      {onDelete && (
        <button onClick={() => onDelete(user.id)}>Delete</button>
      )}
    </div>
  );
}
```

```typescript
// src/components/UserCard.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { vi } from 'vitest';
import { UserCard } from './UserCard';
import { createUser } from '../test/factories/user.factory';

describe('UserCard', () => {
  it('renders user information', () => {
    const user = createUser({ name: 'Alice', email: 'alice@example.com' });
    render(<UserCard user={user} />);

    expect(screen.getByRole('heading', { name: 'Alice' })).toBeInTheDocument();
    expect(screen.getByText('alice@example.com')).toBeInTheDocument();
  });

  it('shows admin badge for admin role', () => {
    const admin = createUser({ role: 'admin' });
    render(<UserCard user={admin} />);
    expect(screen.getByText('Admin')).toBeInTheDocument();
  });

  it('calls onDelete with the user ID when delete is clicked', async () => {
    const user = createUser({ id: 'user-123' });
    const onDelete = vi.fn<[string], void>();

    render(<UserCard user={user} onDelete={onDelete} />);

    await userEvent.click(screen.getByRole('button', { name: /delete/i }));

    expect(onDelete).toHaveBeenCalledWith('user-123');
    expect(onDelete).toHaveBeenCalledTimes(1);
  });

  it('does not render delete button when onDelete is not provided', () => {
    render(<UserCard user={createUser()} />);
    expect(screen.queryByRole('button', { name: /delete/i })).not.toBeInTheDocument();
  });
});
```

---

### Wrapping with Providers

Many components need context providers. Create a typed custom `render` wrapper:

```typescript
// src/test/test-utils.tsx
import React from 'react';
import { render, type RenderOptions } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { BrowserRouter } from 'react-router-dom';

function createTestQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false },
    },
  });
}

interface WrapperProps {
  children: React.ReactNode;
}

function AllProviders({ children }: WrapperProps) {
  const queryClient = createTestQueryClient();
  return (
    <QueryClientProvider client={queryClient}>
      <BrowserRouter>
        {children}
      </BrowserRouter>
    </QueryClientProvider>
  );
}

// Custom render — drop-in replacement for RTL's render
function customRender(
  ui: React.ReactElement,
  options?: Omit<RenderOptions, 'wrapper'>
) {
  return render(ui, { wrapper: AllProviders, ...options });
}

// Re-export everything from RTL
export * from '@testing-library/react';
export { customRender as render };
```

```typescript
// Usage — replaces the RTL import
import { render, screen } from '../test/test-utils';

it('works with all providers', () => {
  render(<MyPageComponent />);
  // QueryClient, Router — all pre-configured
});
```

---

### Typing Async Queries

```typescript
it('loads and displays user data', async () => {
  render(<UserProfilePage userId="1" />);

  // screen.findBy* returns a Promise<HTMLElement> — use await
  const name = await screen.findByRole('heading', { name: /Alice/i });
  expect(name).toBeInTheDocument();

  // Loading state should be gone
  expect(screen.queryByText(/loading/i)).not.toBeInTheDocument();
});
```

---

## 6. Typing MSW (Mock Service Worker) Handlers

### Why MSW?

MSW intercepts network requests at the service worker level, letting you mock APIs without patching `fetch` or `axios`. It works both in the browser and in Node.js (for tests).

### Setup

```bash
npm install -D msw
```

**`src/test/setup.ts`**

```typescript
import { server } from './mocks/server';

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

---

### Defining Typed Handlers

```typescript
// src/test/mocks/handlers.ts
import { http, HttpResponse, type PathParams } from 'msw';

// Type your API response shapes
interface UserResponse {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'viewer';
}

interface ErrorResponse {
  message: string;
  code: string;
}

// Strongly-typed route params
type UserRouteParams = { id: string };

export const handlers = [
  // GET /api/users/:id
  http.get<UserRouteParams, never, UserResponse>(
    '/api/users/:id',
    ({ params }) => {
      const { id } = params;
      //     ^string — typed from UserRouteParams
      return HttpResponse.json({
        id,
        name: 'Alice',
        email: 'alice@example.com',
        role: 'admin',
      });
    }
  ),

  // POST /api/users
  http.post<never, { name: string; email: string }, UserResponse>(
    '/api/users',
    async ({ request }) => {
      const body = await request.json();
      //    ^{ name: string; email: string }
      return HttpResponse.json(
        { id: crypto.randomUUID(), role: 'viewer', ...body },
        { status: 201 }
      );
    }
  ),

  // Simulating errors
  http.get('/api/admin/stats', () => {
    return HttpResponse.json<ErrorResponse>(
      { message: 'Forbidden', code: 'AUTH_REQUIRED' },
      { status: 403 }
    );
  }),
];
```

---

### Setting Up the Server

```typescript
// src/test/mocks/server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);
```

---

### Overriding Handlers Per Test

```typescript
import { server } from '../test/mocks/server';
import { http, HttpResponse } from 'msw';

it('handles server error gracefully', async () => {
  // Override just for this test
  server.use(
    http.get('/api/users/:id', () => {
      return HttpResponse.json(
        { message: 'Internal Server Error', code: 'SERVER_ERROR' },
        { status: 500 }
      );
    })
  );

  render(<UserProfile userId="1" />);
  
  const error = await screen.findByRole('alert');
  expect(error).toHaveTextContent(/something went wrong/i);
});
```

---

### Typing REST Parameters with Generics

```typescript
// For paginated responses
interface PaginatedResponse<T> {
  data: T[];
  total: number;
  page: number;
  pageSize: number;
}

http.get<never, never, PaginatedResponse<UserResponse>>(
  '/api/users',
  ({ request }) => {
    const url = new URL(request.url);
    const page = Number(url.searchParams.get('page') ?? '1');
    
    return HttpResponse.json({
      data: [{ id: '1', name: 'Alice', email: 'alice@test.com', role: 'admin' }],
      total: 1,
      page,
      pageSize: 10,
    });
  }
),
```

---

## 7. Testing Custom Hooks with Typed Return Values

### The `renderHook` API

`@testing-library/react` exports `renderHook` for testing hooks in isolation.

```typescript
import { renderHook, act } from '@testing-library/react';
```

---

### Example: Typed `useCounter` Hook

```typescript
// src/hooks/useCounter.ts
interface UseCounterOptions {
  initial?: number;
  min?: number;
  max?: number;
}

interface UseCounterReturn {
  count: number;
  increment: () => void;
  decrement: () => void;
  reset: () => void;
  isAtMin: boolean;
  isAtMax: boolean;
}

export function useCounter({
  initial = 0,
  min = -Infinity,
  max = Infinity,
}: UseCounterOptions = {}): UseCounterReturn {
  const [count, setCount] = useState(initial);
  
  const increment = () => setCount(c => Math.min(c + 1, max));
  const decrement = () => setCount(c => Math.max(c - 1, min));
  const reset = () => setCount(initial);
  
  return {
    count,
    increment,
    decrement,
    reset,
    isAtMin: count === min,
    isAtMax: count === max,
  };
}
```

```typescript
// src/hooks/useCounter.test.ts
import { renderHook, act } from '@testing-library/react';
import { useCounter } from './useCounter';

describe('useCounter', () => {
  it('starts at the initial value', () => {
    const { result } = renderHook(() => useCounter({ initial: 5 }));
    //         ^{ current: UseCounterReturn }

    expect(result.current.count).toBe(5);
    expect(result.current.isAtMin).toBe(false);
    expect(result.current.isAtMax).toBe(false);
  });

  it('increments the count', () => {
    const { result } = renderHook(() => useCounter());

    act(() => {
      result.current.increment();
    });

    expect(result.current.count).toBe(1);
  });

  it('respects max boundary', () => {
    const { result } = renderHook(() => useCounter({ initial: 9, max: 10 }));

    act(() => {
      result.current.increment();
    });
    expect(result.current.count).toBe(10);
    expect(result.current.isAtMax).toBe(true);

    act(() => {
      result.current.increment(); // Should not exceed max
    });
    expect(result.current.count).toBe(10);
  });

  it('resets to initial value', () => {
    const { result } = renderHook(() => useCounter({ initial: 5 }));

    act(() => {
      result.current.increment();
      result.current.increment();
    });
    expect(result.current.count).toBe(7);

    act(() => {
      result.current.reset();
    });
    expect(result.current.count).toBe(5);
  });
});
```

---

### Testing Async Hooks

```typescript
// src/hooks/useUsers.ts
interface UseUsersReturn {
  users: User[];
  isLoading: boolean;
  error: Error | null;
  refetch: () => Promise<void>;
}

export function useUsers(): UseUsersReturn {
  const [users, setUsers] = useState<User[]>([]);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  const fetchUsers = async () => {
    setIsLoading(true);
    setError(null);
    try {
      const response = await fetch('/api/users');
      if (!response.ok) throw new Error('Failed to fetch users');
      const data = await response.json();
      setUsers(data);
    } catch (err) {
      setError(err instanceof Error ? err : new Error('Unknown error'));
    } finally {
      setIsLoading(false);
    }
  };

  useEffect(() => { fetchUsers(); }, []);

  return { users, isLoading, error, refetch: fetchUsers };
}
```

```typescript
// src/hooks/useUsers.test.ts
import { renderHook, waitFor } from '@testing-library/react';
import { useUsers } from './useUsers';

// MSW handles the fetch — no need to mock fetch directly
describe('useUsers', () => {
  it('fetches users on mount', async () => {
    const { result } = renderHook(() => useUsers());

    // Immediately after render, should be loading
    expect(result.current.isLoading).toBe(true);

    // Wait for state to settle
    await waitFor(() => {
      expect(result.current.isLoading).toBe(false);
    });

    expect(result.current.users).toHaveLength(1);
    expect(result.current.error).toBeNull();
  });

  it('handles fetch errors', async () => {
    server.use(
      http.get('/api/users', () => HttpResponse.error())
    );

    const { result } = renderHook(() => useUsers());

    await waitFor(() => {
      expect(result.current.isLoading).toBe(false);
    });

    expect(result.current.error).toBeInstanceOf(Error);
    expect(result.current.users).toHaveLength(0);
  });
});
```

---

### Testing Hooks with Context Providers

```typescript
// When the hook needs providers
const { result } = renderHook(() => useAuth(), {
  wrapper: ({ children }) => (
    <AuthProvider>{children}</AuthProvider>
  ),
});
```

Or use the custom render wrapper from Section 5:

```typescript
import { renderHook } from '../test/test-utils';
// Uses AllProviders automatically
```

---

## 8. Type-Level Testing with `expect-type`

### Why Type-Level Tests?

Runtime tests verify *behavior*. Type-level tests verify *type contracts* — that your generics, overloads, and conditional types produce the types you intend.

```bash
npm install -D expect-type
```

---

### Core API

```typescript
import { expectTypeOf } from 'expect-type';

// Assert a value's type
expectTypeOf(someValue).toEqualTypeOf<ExpectedType>();

// Assert assignability (less strict)
expectTypeOf(someValue).toMatchTypeOf<ExpectedType>();

// Assert it's a function
expectTypeOf(myFn).toBeFunction();

// Assert it's nullable
expectTypeOf(maybeNull).toBeNullable();

// Assert a function's parameters and return type
expectTypeOf(myFn).parameter(0).toEqualTypeOf<string>();
expectTypeOf(myFn).returns.toEqualTypeOf<Promise<User>>();
```

---

### Testing Inference on Generic Functions

```typescript
// src/utils/identity.ts
export function identity<T>(value: T): T {
  return value;
}
```

```typescript
// src/utils/identity.test-d.ts  (or .test.ts)
import { expectTypeOf } from 'expect-type';
import { identity } from './identity';

it('infers the type from the argument', () => {
  expectTypeOf(identity(42)).toEqualTypeOf<number>();
  expectTypeOf(identity('hello')).toEqualTypeOf<string>();
  expectTypeOf(identity({ a: 1 })).toEqualTypeOf<{ a: number }>();
  expectTypeOf(identity([1, 2, 3])).toEqualTypeOf<number[]>();
});
```

---

### Testing Conditional Types

```typescript
// src/types/utilities.ts
type NonNullable<T> = T extends null | undefined ? never : T;

type IsString<T> = T extends string ? true : false;

type Flatten<T> = T extends Array<infer U> ? U : T;
```

```typescript
// src/types/utilities.test-d.ts
import { expectTypeOf } from 'expect-type';
import type { Flatten } from './utilities';

it('Flatten unwraps arrays', () => {
  expectTypeOf<Flatten<number[]>>().toEqualTypeOf<number>();
  expectTypeOf<Flatten<string[]>>().toEqualTypeOf<string>();
  expectTypeOf<Flatten<string>>().toEqualTypeOf<string>(); // Non-array passthrough
});
```

---

### Testing Overloaded Functions

```typescript
// src/utils/parse.ts
export function parse(value: string): number;
export function parse(value: number): string;
export function parse(value: string | number): number | string {
  if (typeof value === 'string') return parseFloat(value);
  return String(value);
}
```

```typescript
it('has correct overload signatures', () => {
  // When called with string → returns number
  expectTypeOf(parse('3.14')).toEqualTypeOf<number>();

  // When called with number → returns string
  expectTypeOf(parse(42)).toEqualTypeOf<string>();
});
```

---

### Testing Type Guards

```typescript
// src/utils/guards.ts
export function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'name' in value
  );
}
```

```typescript
it('isUser is a type predicate', () => {
  // Verify it's typed as a type guard function
  expectTypeOf(isUser).guards.toEqualTypeOf<User>();
});

it('narrows type correctly at runtime', () => {
  const value: unknown = { id: '1', name: 'Alice' };
  
  if (isUser(value)) {
    // TypeScript knows value is User here
    expectTypeOf(value).toEqualTypeOf<User>();
  }
});
```

---

### Testing React Component Prop Types

```typescript
import { expectTypeOf } from 'expect-type';
import type { ComponentProps } from 'react';
import { UserCard } from './UserCard';

it('UserCard has the expected props', () => {
  type Props = ComponentProps<typeof UserCard>;

  expectTypeOf<Props['user']>().toMatchTypeOf<{
    id: string;
    name: string;
  }>();

  // onDelete is optional
  expectTypeOf<Props['onDelete']>().toEqualTypeOf<
    ((id: string) => void) | undefined
  >();
});
```

---

### Organizing Type-Level Tests

Convention: use `.test-d.ts` suffix for pure type tests, or co-locate with `.test.ts` files:

```
src/
  utils/
    array.ts
    array.test.ts        ← runtime tests
    array.test-d.ts      ← type-level tests
  types/
    utilities.ts
    utilities.test-d.ts  ← type-level tests only
```

In `vitest.config.ts`, include both:

```typescript
export default defineConfig({
  test: {
    include: ['**/*.test.ts', '**/*.test-d.ts'],
  },
});
```

---

## 9. Putting It All Together

### A Complete Feature Test Example

Here's a realistic end-to-end example combining everything from this module: a typed user management feature with tests at every layer.

**Types & API**

```typescript
// src/types/user.ts
export interface User {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'viewer';
}

export interface CreateUserPayload {
  name: string;
  email: string;
  role: User['role'];
}
```

```typescript
// src/api/users.ts
export async function fetchUser(id: string): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error(`Failed to fetch user: ${res.status}`);
  return res.json();
}

export async function createUser(payload: CreateUserPayload): Promise<User> {
  const res = await fetch('/api/users', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(payload),
  });
  if (!res.ok) throw new Error('Failed to create user');
  return res.json();
}
```

**Hook**

```typescript
// src/hooks/useUserManager.ts
import { useState } from 'react';
import { fetchUser, createUser } from '../api/users';
import type { User, CreateUserPayload } from '../types/user';

interface UseUserManagerReturn {
  user: User | null;
  isLoading: boolean;
  error: string | null;
  loadUser: (id: string) => Promise<void>;
  addUser: (payload: CreateUserPayload) => Promise<User>;
}

export function useUserManager(): UseUserManagerReturn {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const loadUser = async (id: string) => {
    setIsLoading(true);
    setError(null);
    try {
      const data = await fetchUser(id);
      setUser(data);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Unknown error');
    } finally {
      setIsLoading(false);
    }
  };

  const addUser = async (payload: CreateUserPayload): Promise<User> => {
    const newUser = await createUser(payload);
    setUser(newUser);
    return newUser;
  };

  return { user, isLoading, error, loadUser, addUser };
}
```

**Complete Test Suite**

```typescript
// src/hooks/useUserManager.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { renderHook, act, waitFor } from '@testing-library/react';
import { expectTypeOf } from 'expect-type';
import { http, HttpResponse } from 'msw';
import { server } from '../test/mocks/server';
import { useUserManager } from './useUserManager';
import { createUser as createUserFactory } from '../test/factories/user.factory';
import type { UseUserManagerReturn } from './useUserManager';

// ── Type-level test ──────────────────────────────────────────────────────────
it('has the correct return type signature', () => {
  expectTypeOf<UseUserManagerReturn['user']>().toEqualTypeOf<User | null>();
  expectTypeOf<UseUserManagerReturn['error']>().toEqualTypeOf<string | null>();
  expectTypeOf<UseUserManagerReturn['loadUser']>()
    .parameter(0).toEqualTypeOf<string>();
  expectTypeOf<UseUserManagerReturn['loadUser']>()
    .returns.toEqualTypeOf<Promise<void>>();
});

// ── Runtime tests ────────────────────────────────────────────────────────────
describe('useUserManager', () => {
  const mockUser = createUserFactory({ id: 'user-1', name: 'Alice' });

  beforeEach(() => {
    // Set up default MSW handler
    server.use(
      http.get('/api/users/:id', ({ params }) => {
        if (params.id === 'user-1') {
          return HttpResponse.json(mockUser);
        }
        return HttpResponse.json({ message: 'Not found' }, { status: 404 });
      })
    );
  });

  it('starts with null state', () => {
    const { result } = renderHook(() => useUserManager());
    expect(result.current.user).toBeNull();
    expect(result.current.isLoading).toBe(false);
    expect(result.current.error).toBeNull();
  });

  it('loads a user by ID', async () => {
    const { result } = renderHook(() => useUserManager());

    act(() => {
      result.current.loadUser('user-1');
    });

    expect(result.current.isLoading).toBe(true);

    await waitFor(() => {
      expect(result.current.isLoading).toBe(false);
    });

    expect(result.current.user?.name).toBe('Alice');
    expect(result.current.error).toBeNull();
  });

  it('sets error state on failed load', async () => {
    const { result } = renderHook(() => useUserManager());

    await act(async () => {
      await result.current.loadUser('nonexistent-id');
    });

    expect(result.current.user).toBeNull();
    expect(result.current.error).toMatch(/Failed to fetch user/);
  });

  it('adds a user and updates state', async () => {
    server.use(
      http.post('/api/users', async ({ request }) => {
        const body = await request.json() as { name: string; email: string };
        return HttpResponse.json(
          createUserFactory({ ...body, id: 'new-user' }),
          { status: 201 }
        );
      })
    );

    const { result } = renderHook(() => useUserManager());
    let createdUser: User;

    await act(async () => {
      createdUser = await result.current.addUser({
        name: 'Bob',
        email: 'bob@example.com',
        role: 'viewer',
      });
    });

    expect(createdUser!.id).toBe('new-user');
    expect(result.current.user?.name).toBe('Bob');
  });
});
```

---

## Quick Reference Summary

| Concept | Syntax |
|---|---|
| Typed mock | `vi.fn<[Args], Return>()` |
| Typed spy | `vi.spyOn(obj, 'method')` — auto-typed |
| Mock module | `vi.mock('./module')` + cast with `as vi.MockedFunction<typeof fn>` |
| Test factory | `function createX(overrides: DeepPartial<X> = {}): X` |
| Generic test | Test each relevant type instantiation separately |
| RTL typed render | `render<ComponentType>(<Component />)` — usually inferred |
| Custom render | Wrap RTL's `render` with providers |
| MSW typed handler | `http.get<Params, Body, Response>(path, handler)` |
| Hook testing | `renderHook(() => useMyHook())` → `result.current` |
| Async hook | `await waitFor(() => expect(result.current.x).toBe(y))` |
| Type assertion | `expectTypeOf(value).toEqualTypeOf<Expected>()` |
| Guard test | `expectTypeOf(fn).guards.toEqualTypeOf<GuardedType>()` |

---

## Further Reading

- [Vitest Documentation](https://vitest.dev/)
- [Jest Documentation](https://jestjs.io/)
- [Testing Library Docs](https://testing-library.com/)
- [MSW Documentation](https://mswjs.io/)
- [expect-type on GitHub](https://github.com/mmkal/expect-type)
- [TypeScript Handbook — Generics](https://www.typescriptlang.org/docs/handbook/2/generics.html)

---

*End of Week 13 — Testing Typed Code*
