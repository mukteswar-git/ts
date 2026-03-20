# Week 9: Node.js & Express Typing with TypeScript

> A complete tutorial on integrating TypeScript into Node.js and Express applications — covering setup, typing patterns, middleware, environment variables, error handling, and project configuration.

---

## Table of Contents

1. [TypeScript in Node.js (ts-node, tsx)](#1-typescript-in-nodejs-ts-node-tsx)
2. [Typing Express Request and Response](#2-typing-express-request-and-response)
3. [Extending Express Types (req.user, etc.)](#3-extending-express-types-requser-etc)
4. [Typed Middleware](#4-typed-middleware)
5. [Typing Environment Variables (dotenv + zod)](#5-typing-environment-variables-dotenv--zod)
6. [Module Resolution for Node (@types/node)](#6-module-resolution-for-node-typesnode)
7. [Typed Error Handlers](#7-typed-error-handlers)
8. [Path Aliases in tsconfig](#8-path-aliases-in-tsconfig)
9. [Putting It All Together](#9-putting-it-all-together)

---

## 1. TypeScript in Node.js (ts-node, tsx)

### Why TypeScript in Node.js?

Node.js runs JavaScript natively, but TypeScript gives you static typing, better IDE support, and fewer runtime errors. To run TypeScript in Node.js, you need a tool that compiles or transpiles `.ts` files on the fly.

### Installing the Tools

```bash
# Initialize a new project
mkdir my-express-app && cd my-express-app
npm init -y

# Install TypeScript and Node.js types
npm install -D typescript @types/node

# Initialize tsconfig
npx tsc --init
```

### Option 1: ts-node

`ts-node` is a TypeScript execution engine for Node.js. It compiles TypeScript on the fly without emitting files.

```bash
npm install -D ts-node
```

**Running a file:**

```bash
npx ts-node src/index.ts
```

**Using with nodemon for development:**

```bash
npm install -D nodemon
```

Add a `nodemon.json`:

```json
{
  "watch": ["src"],
  "ext": "ts",
  "exec": "ts-node src/index.ts"
}
```

Add to `package.json`:

```json
{
  "scripts": {
    "dev": "nodemon",
    "build": "tsc",
    "start": "node dist/index.js"
  }
}
```

### Option 2: tsx (Recommended for Modern Projects)

`tsx` is a faster alternative to `ts-node`. It uses esbuild under the hood for near-instant transpilation.

```bash
npm install -D tsx
```

**Running a file:**

```bash
npx tsx src/index.ts
```

**Watch mode (built-in):**

```bash
npx tsx watch src/index.ts
```

**package.json scripts:**

```json
{
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js"
  }
}
```

### ts-node vs tsx — Quick Comparison

| Feature | ts-node | tsx |
|---|---|---|
| Speed | Slower (uses TypeScript compiler) | Fast (uses esbuild) |
| Type checking | Full type checking | No type checking (transpile-only) |
| ESM support | Requires extra config | Built-in |
| Best for | CI/CD where you want full type checks | Development speed |

> **Best practice:** Use `tsx` for development speed, but run `tsc --noEmit` separately for type-checking (e.g., in CI or a pre-commit hook).

### A Basic tsconfig.json for Node.js

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

---

## 2. Typing Express Request and Response

### Installing Express and Types

```bash
npm install express
npm install -D @types/express
```

### Basic Typed Express Server

```typescript
// src/index.ts
import express, { Application, Request, Response } from 'express';

const app: Application = express();

app.use(express.json());

app.get('/', (req: Request, res: Response) => {
  res.json({ message: 'Hello, TypeScript!' });
});

app.listen(3000, () => {
  console.log('Server running on http://localhost:3000');
});
```

### Typing Route Parameters

Use generics on `Request` to type `params`, `query`, `body`, and `locals`:

```typescript
// Request<Params, ResBody, ReqBody, QueryString>
import { Request, Response } from 'express';

interface UserParams {
  id: string;
}

interface UserBody {
  name: string;
  email: string;
}

interface UserQuery {
  include?: string;
}

// GET /users/:id?include=profile
app.get('/users/:id', (
  req: Request<UserParams, {}, {}, UserQuery>,
  res: Response
) => {
  const userId = req.params.id;       // string ✅
  const include = req.query.include;  // string | undefined ✅
  res.json({ userId, include });
});

// POST /users
app.post('/users', (
  req: Request<{}, {}, UserBody>,
  res: Response
) => {
  const { name, email } = req.body;  // typed ✅
  res.status(201).json({ name, email });
});
```

### Typing the Response Body

```typescript
interface ApiResponse<T> {
  data: T;
  success: boolean;
  message?: string;
}

interface User {
  id: number;
  name: string;
  email: string;
}

// Type the response body with the second generic
app.get('/users/:id', (
  req: Request<{ id: string }>,
  res: Response<ApiResponse<User>>
) => {
  const user: User = { id: 1, name: 'Alice', email: 'alice@example.com' };

  res.json({
    data: user,
    success: true,
    message: 'User found'
  });
});
```

### Using RequestHandler Type

For reusable handler functions, use the `RequestHandler` type:

```typescript
import { RequestHandler } from 'express';

const getUser: RequestHandler<{ id: string }> = (req, res) => {
  const { id } = req.params; // typed as string ✅
  res.json({ id });
};

app.get('/users/:id', getUser);
```

---

## 3. Extending Express Types (req.user, etc.)

By default, `req.user` doesn't exist on the Express `Request` type. To add custom properties, you extend Express's type definitions using **declaration merging**.

### Setting Up a Type Declaration File

Create a file `src/types/express/index.d.ts`:

```typescript
// src/types/express/index.d.ts

// Import the base express types
import { User } from '../models/User'; // your User model

declare global {
  namespace Express {
    interface Request {
      user?: User;           // added by auth middleware
      requestId?: string;    // added by request ID middleware
      startTime?: number;    // added by logging middleware
    }
  }
}

// This empty export makes it a module (required)
export {};
```

### Your User Model

```typescript
// src/types/models/User.ts
export interface User {
  id: number;
  email: string;
  role: 'admin' | 'user' | 'guest';
  name: string;
}
```

### Telling TypeScript Where to Find Your Types

In `tsconfig.json`, make sure your types folder is included:

```json
{
  "compilerOptions": {
    "typeRoots": ["./src/types", "./node_modules/@types"]
  },
  "include": ["src/**/*"]
}
```

### Using the Extended Types

After declaration merging, `req.user` is fully typed everywhere:

```typescript
import { Request, Response } from 'express';

app.get('/profile', (req: Request, res: Response) => {
  if (!req.user) {
    return res.status(401).json({ error: 'Not authenticated' });
  }

  // req.user is typed as User ✅
  const { id, email, role } = req.user;

  res.json({ id, email, role });
});
```

### Extending with Multiple Properties

```typescript
// src/types/express/index.d.ts
import { User } from '../models/User';
import { Session } from 'express-session';

declare global {
  namespace Express {
    interface Request {
      user?: User;
      requestId: string;
      startTime: number;
      logger: {
        info: (msg: string) => void;
        error: (msg: string, err?: Error) => void;
      };
    }

    // Extend the Response locals
    interface Locals {
      cacheControl?: string;
    }
  }
}

export {};
```

---

## 4. Typed Middleware

Middleware in Express has the signature `(req, res, next) => void`. TypeScript makes it easy to type these correctly.

### The RequestHandler Type

```typescript
import { RequestHandler, Request, Response, NextFunction } from 'express';

// Basic middleware — no type parameters needed
const logger: RequestHandler = (req, res, next) => {
  console.log(`${req.method} ${req.path}`);
  next();
};

app.use(logger);
```

### Authentication Middleware

```typescript
// src/middleware/auth.ts
import { RequestHandler } from 'express';
import jwt from 'jsonwebtoken';
import { User } from '../types/models/User';

export const authenticate: RequestHandler = (req, res, next) => {
  const authHeader = req.headers.authorization;

  if (!authHeader?.startsWith('Bearer ')) {
    res.status(401).json({ error: 'Missing or invalid token' });
    return; // Important: return to stop execution, don't call next()
  }

  const token = authHeader.split(' ')[1];

  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET!) as User;
    req.user = payload; // Works because we extended Request ✅
    next();
  } catch (err) {
    res.status(401).json({ error: 'Invalid token' });
  }
};
```

### Role-Based Authorization Middleware (Middleware Factory)

A middleware factory is a function that returns a `RequestHandler`. This pattern is common for parameterized middleware:

```typescript
// src/middleware/authorize.ts
import { RequestHandler } from 'express';
import { User } from '../types/models/User';

type Role = User['role'];

export const authorize = (...roles: Role[]): RequestHandler => {
  return (req, res, next) => {
    if (!req.user) {
      res.status(401).json({ error: 'Not authenticated' });
      return;
    }

    if (!roles.includes(req.user.role)) {
      res.status(403).json({ error: 'Insufficient permissions' });
      return;
    }

    next();
  };
};
```

**Using the factory:**

```typescript
import { authenticate } from './middleware/auth';
import { authorize } from './middleware/authorize';

// Route protected by authentication AND authorization
app.delete('/users/:id',
  authenticate,
  authorize('admin'),
  (req, res) => {
    res.json({ deleted: req.params.id });
  }
);
```

### Request ID Middleware

```typescript
// src/middleware/requestId.ts
import { RequestHandler } from 'express';
import { randomUUID } from 'crypto';

export const attachRequestId: RequestHandler = (req, res, next) => {
  req.requestId = randomUUID(); // Works because we extended Request ✅
  res.setHeader('X-Request-ID', req.requestId);
  next();
};
```

### Validation Middleware (with Zod)

```typescript
// src/middleware/validate.ts
import { RequestHandler } from 'express';
import { ZodSchema, ZodError } from 'zod';

export const validateBody = <T>(schema: ZodSchema<T>): RequestHandler => {
  return (req, res, next) => {
    const result = schema.safeParse(req.body);

    if (!result.success) {
      res.status(400).json({
        error: 'Validation failed',
        details: result.error.flatten().fieldErrors
      });
      return;
    }

    req.body = result.data; // Override with parsed/typed data
    next();
  };
};
```

**Using the validation middleware:**

```typescript
import { z } from 'zod';
import { validateBody } from './middleware/validate';

const createUserSchema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
  age: z.number().min(18)
});

type CreateUserDto = z.infer<typeof createUserSchema>;

app.post('/users',
  validateBody(createUserSchema),
  (req: Request<{}, {}, CreateUserDto>, res) => {
    const { name, email, age } = req.body; // Fully typed ✅
    res.status(201).json({ name, email, age });
  }
);
```

---

## 5. Typing Environment Variables (dotenv + zod)

Environment variables are always strings at runtime, which makes them a common source of bugs. Using `dotenv` + `zod` gives you validated, typed env vars.

### Installation

```bash
npm install dotenv zod
```

### The Pattern: Validate at Startup

Create a dedicated env module that validates and exports your config:

```typescript
// src/config/env.ts
import { z } from 'zod';
import dotenv from 'dotenv';

// Load the .env file
dotenv.config();

// Define the schema
const envSchema = z.object({
  // Server
  NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
  PORT: z.coerce.number().min(1).max(65535).default(3000),
  HOST: z.string().default('0.0.0.0'),

  // Database
  DATABASE_URL: z.string().url(),
  DB_POOL_SIZE: z.coerce.number().default(10),

  // Auth
  JWT_SECRET: z.string().min(32, 'JWT_SECRET must be at least 32 characters'),
  JWT_EXPIRES_IN: z.string().default('7d'),

  // Optional
  SENTRY_DSN: z.string().url().optional(),
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
});

// Validate — throws a ZodError with helpful messages if invalid
const parsed = envSchema.safeParse(process.env);

if (!parsed.success) {
  console.error('❌ Invalid environment variables:');
  console.error(parsed.error.flatten().fieldErrors);
  process.exit(1); // Exit immediately — don't run with bad config
}

// Export the typed config
export const env = parsed.data;

// Export the inferred type for use elsewhere
export type Env = z.infer<typeof envSchema>;
```

### Your .env File

```bash
# .env
NODE_ENV=development
PORT=3000
DATABASE_URL=postgres://user:password@localhost:5432/mydb
JWT_SECRET=super-secret-key-at-least-32-characters-long
JWT_EXPIRES_IN=7d
LOG_LEVEL=info
```

### Using the Typed Config

```typescript
// src/index.ts
import { env } from './config/env';

// All properties are typed ✅
// env.PORT is number (not string!)
// env.NODE_ENV is 'development' | 'production' | 'test'
// env.SENTRY_DSN is string | undefined

app.listen(env.PORT, env.HOST, () => {
  console.log(`Server running in ${env.NODE_ENV} mode`);
  console.log(`Listening at http://${env.HOST}:${env.PORT}`);
});
```

### Handling Multiple Environments

```typescript
// src/config/env.ts (extended)
import { z } from 'zod';
import dotenv from 'dotenv';
import path from 'path';

// Load environment-specific .env file
const envFile = process.env.NODE_ENV === 'test' ? '.env.test' : '.env';
dotenv.config({ path: path.resolve(process.cwd(), envFile) });

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
}).refine(
  (data) => data.NODE_ENV !== 'production' || data.JWT_SECRET.length >= 64,
  { message: 'In production, JWT_SECRET must be at least 64 characters', path: ['JWT_SECRET'] }
);

const result = envSchema.safeParse(process.env);

if (!result.success) {
  console.error('❌ Environment validation failed:');
  result.error.issues.forEach(issue => {
    console.error(`  ${issue.path.join('.')}: ${issue.message}`);
  });
  process.exit(1);
}

export const env = result.data;
```

---

## 6. Module Resolution for Node (@types/node)

Node.js has built-in modules like `fs`, `path`, `crypto`, and `http`. To use them with TypeScript, you need `@types/node`.

### Installation

```bash
npm install -D @types/node
```

### Using Built-in Node Modules

```typescript
import * as fs from 'fs';
import * as path from 'path';
import * as crypto from 'crypto';
import { EventEmitter } from 'events';
import { promisify } from 'util';

// All fully typed ✅
const readFile = promisify(fs.readFile);

async function readConfig(filename: string): Promise<string> {
  const filePath = path.join(__dirname, '..', 'config', filename);
  const content = await readFile(filePath, 'utf-8');
  return content;
}

function generateToken(): string {
  return crypto.randomBytes(32).toString('hex');
}
```

### Configuring Module Resolution in tsconfig

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "moduleResolution": "node",
    "target": "ES2020",
    "lib": ["ES2020"],
    "types": ["node"],       // Only include @types/node globally
    "typeRoots": [
      "./src/types",         // Your custom types first
      "./node_modules/@types" // Then DefinitelyTyped
    ]
  }
}
```

### ESM vs CommonJS

If you use `"module": "ESNext"` or `"module": "Node16"` (for ESM), your imports change:

```typescript
// CommonJS (module: "commonjs")
import path from 'path';
import { readFile } from 'fs/promises';
const __dirname = path.dirname(new URL(import.meta.url).pathname); // not needed in CJS

// ESM (module: "ESNext" or "Node16")
import path from 'path';
import { readFile } from 'fs/promises';
import { fileURLToPath } from 'url';

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename); // ESM doesn't have __dirname by default
```

### tsconfig for ESM Node.js

```json
{
  "compilerOptions": {
    "module": "Node16",
    "moduleResolution": "Node16",
    "target": "ES2022",
    "outDir": "./dist"
  }
}
```

And in `package.json`:

```json
{
  "type": "module"
}
```

---

## 7. Typed Error Handlers

Express error-handling middleware takes four parameters: `(err, req, res, next)`. TypeScript requires the four-parameter signature to recognize it as an error handler.

### Defining a Custom Error Class

```typescript
// src/errors/AppError.ts
export class AppError extends Error {
  public readonly statusCode: number;
  public readonly isOperational: boolean;
  public readonly code?: string;

  constructor(
    message: string,
    statusCode: number = 500,
    isOperational: boolean = true,
    code?: string
  ) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = isOperational;
    this.code = code;

    // Restore prototype chain (needed when extending built-ins in TypeScript)
    Object.setPrototypeOf(this, new.target.prototype);
    Error.captureStackTrace(this, this.constructor);
  }
}

// Specific error subclasses
export class NotFoundError extends AppError {
  constructor(resource: string) {
    super(`${resource} not found`, 404, true, 'NOT_FOUND');
  }
}

export class ValidationError extends AppError {
  public readonly fields?: Record<string, string[]>;

  constructor(message: string, fields?: Record<string, string[]>) {
    super(message, 400, true, 'VALIDATION_ERROR');
    this.fields = fields;
  }
}

export class UnauthorizedError extends AppError {
  constructor(message = 'Unauthorized') {
    super(message, 401, true, 'UNAUTHORIZED');
  }
}
```

### The Typed Error Handler Middleware

```typescript
// src/middleware/errorHandler.ts
import { ErrorRequestHandler } from 'express';
import { ZodError } from 'zod';
import { AppError, ValidationError } from '../errors/AppError';
import { env } from '../config/env';

// ErrorRequestHandler is: (err, req, res, next) => void
export const errorHandler: ErrorRequestHandler = (err, req, res, next) => {
  // Zod validation errors
  if (err instanceof ZodError) {
    res.status(400).json({
      success: false,
      error: 'Validation failed',
      code: 'VALIDATION_ERROR',
      details: err.flatten().fieldErrors
    });
    return;
  }

  // Custom operational errors
  if (err instanceof ValidationError) {
    res.status(err.statusCode).json({
      success: false,
      error: err.message,
      code: err.code,
      fields: err.fields
    });
    return;
  }

  if (err instanceof AppError) {
    res.status(err.statusCode).json({
      success: false,
      error: err.message,
      code: err.code
    });
    return;
  }

  // Unknown / programming errors
  console.error('Unhandled error:', err);

  res.status(500).json({
    success: false,
    error: env.NODE_ENV === 'production'
      ? 'An unexpected error occurred'
      : err.message,
    // Only include stack trace in development
    ...(env.NODE_ENV === 'development' && { stack: err.stack })
  });
};
```

### A "Not Found" Route Handler

```typescript
// src/middleware/notFound.ts
import { RequestHandler } from 'express';
import { NotFoundError } from '../errors/AppError';

export const notFound: RequestHandler = (req, res, next) => {
  next(new NotFoundError(`Route ${req.method} ${req.path}`));
};
```

### Wiring It Up

The error handler and 404 handler must be added **after** all routes:

```typescript
// src/index.ts
import express from 'express';
import { errorHandler } from './middleware/errorHandler';
import { notFound } from './middleware/notFound';
import userRoutes from './routes/users';

const app = express();
app.use(express.json());

// Routes
app.use('/api/users', userRoutes);

// 404 — must come after all routes
app.use(notFound);

// Error handler — must be last, must have 4 parameters
app.use(errorHandler);
```

### Throwing Typed Errors in Route Handlers

```typescript
import { Request, Response, NextFunction } from 'express';
import { NotFoundError, UnauthorizedError } from '../errors/AppError';

app.get('/users/:id', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const user = await db.users.findById(req.params.id);

    if (!user) {
      throw new NotFoundError('User'); // 404 ✅
    }

    if (user.id !== req.user?.id) {
      throw new UnauthorizedError('Cannot access another user\'s profile'); // 401 ✅
    }

    res.json(user);
  } catch (err) {
    next(err); // Pass to error handler ✅
  }
});
```

### Async Error Wrapper Utility

Wrapping every handler in `try/catch` is repetitive. Use a helper:

```typescript
// src/utils/asyncHandler.ts
import { Request, Response, NextFunction, RequestHandler } from 'express';

type AsyncRequestHandler<
  P = {},
  ResBody = any,
  ReqBody = any,
  Query = {}
> = (
  req: Request<P, ResBody, ReqBody, Query>,
  res: Response<ResBody>,
  next: NextFunction
) => Promise<void>;

export const asyncHandler = <P = {}, ResBody = any, ReqBody = any, Query = {}>(
  fn: AsyncRequestHandler<P, ResBody, ReqBody, Query>
): RequestHandler<P, ResBody, ReqBody, Query> => {
  return (req, res, next) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
};
```

**Using the wrapper:**

```typescript
import { asyncHandler } from '../utils/asyncHandler';

app.get('/users/:id', asyncHandler<{ id: string }>(async (req, res) => {
  const user = await db.users.findById(req.params.id);

  if (!user) throw new NotFoundError('User');

  res.json(user); // No try/catch needed! ✅
}));
```

---

## 8. Path Aliases in tsconfig

Long relative imports like `../../../middleware/auth` are hard to read and brittle. Path aliases let you use short, absolute-style imports instead.

### Configuring tsconfig.json

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@config/*": ["src/config/*"],
      "@middleware/*": ["src/middleware/*"],
      "@routes/*": ["src/routes/*"],
      "@models/*": ["src/models/*"],
      "@utils/*": ["src/utils/*"],
      "@errors/*": ["src/errors/*"],
      "@types/*": ["src/types/*"]
    }
  }
}
```

### Before and After

```typescript
// ❌ Before — brittle relative imports
import { authenticate } from '../../../middleware/auth';
import { AppError } from '../../errors/AppError';
import { env } from '../config/env';

// ✅ After — clean path aliases
import { authenticate } from '@middleware/auth';
import { AppError } from '@errors/AppError';
import { env } from '@config/env';
```

### Making Aliases Work at Runtime

TypeScript compiles aliases away in type-checking, but Node.js won't understand them at runtime. You need a module alias resolver.

**Option A: Using `tsconfig-paths` (with ts-node)**

```bash
npm install -D tsconfig-paths
```

Update `nodemon.json` or your ts-node command:

```json
{
  "exec": "ts-node -r tsconfig-paths/register src/index.ts"
}
```

Or in `package.json`:

```json
{
  "scripts": {
    "dev": "tsx --tsconfig tsconfig.json src/index.ts"
  }
}
```

**Option B: Using `tsc-alias` (for compiled output)**

```bash
npm install -D tsc-alias
```

Update `package.json`:

```json
{
  "scripts": {
    "build": "tsc && tsc-alias",
    "start": "node dist/index.js"
  }
}
```

**Option C: Using `tsx` (zero-config)**

`tsx` respects `tsconfig.json` paths automatically during development:

```json
{
  "scripts": {
    "dev": "tsx watch src/index.ts"
  }
}
```

You still need `tsc-alias` for the compiled production build.

### Recommended Project Structure with Aliases

```
src/
├── config/
│   └── env.ts          (@config/env)
├── errors/
│   └── AppError.ts     (@errors/AppError)
├── middleware/
│   ├── auth.ts         (@middleware/auth)
│   ├── errorHandler.ts (@middleware/errorHandler)
│   └── validate.ts     (@middleware/validate)
├── models/
│   └── User.ts         (@models/User)
├── routes/
│   └── users.ts        (@routes/users)
├── types/
│   └── express/
│       └── index.d.ts  (declaration merging)
├── utils/
│   └── asyncHandler.ts (@utils/asyncHandler)
└── index.ts
```

---

## 9. Putting It All Together

Here is a complete, production-ready Express app that uses every concept from this tutorial.

### Final Project Structure

```
my-express-app/
├── src/
│   ├── config/
│   │   └── env.ts
│   ├── errors/
│   │   └── AppError.ts
│   ├── middleware/
│   │   ├── auth.ts
│   │   ├── errorHandler.ts
│   │   ├── notFound.ts
│   │   ├── requestId.ts
│   │   └── validate.ts
│   ├── routes/
│   │   └── users.ts
│   ├── types/
│   │   └── express/
│   │       └── index.d.ts
│   ├── utils/
│   │   └── asyncHandler.ts
│   └── index.ts
├── .env
├── .env.example
├── nodemon.json
├── package.json
└── tsconfig.json
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@config/*": ["src/config/*"],
      "@middleware/*": ["src/middleware/*"],
      "@routes/*": ["src/routes/*"],
      "@errors/*": ["src/errors/*"],
      "@utils/*": ["src/utils/*"],
      "@types/*": ["src/types/*"]
    },
    "typeRoots": ["./src/types", "./node_modules/@types"]
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### package.json

```json
{
  "name": "my-express-app",
  "version": "1.0.0",
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "build": "tsc && tsc-alias",
    "start": "node dist/index.js",
    "type-check": "tsc --noEmit"
  },
  "dependencies": {
    "express": "^4.18.2",
    "zod": "^3.22.0",
    "dotenv": "^16.3.1",
    "jsonwebtoken": "^9.0.0"
  },
  "devDependencies": {
    "@types/express": "^4.17.21",
    "@types/node": "^20.10.0",
    "@types/jsonwebtoken": "^9.0.5",
    "typescript": "^5.3.0",
    "tsx": "^4.6.0",
    "tsc-alias": "^1.8.8"
  }
}
```

### src/index.ts

```typescript
import express from 'express';
import { env } from '@config/env';
import { authenticate } from '@middleware/auth';
import { errorHandler } from '@middleware/errorHandler';
import { notFound } from '@middleware/notFound';
import { attachRequestId } from '@middleware/requestId';
import userRoutes from '@routes/users';

const app = express();

// Core middleware
app.use(express.json());
app.use(attachRequestId);

// Routes
app.use('/api/users', userRoutes);

// Protected route example
app.get('/api/profile', authenticate, (req, res) => {
  res.json({ user: req.user }); // req.user is typed ✅
});

// 404 and Error handlers — must be last
app.use(notFound);
app.use(errorHandler);

app.listen(env.PORT, () => {
  console.log(`🚀 Server running at http://localhost:${env.PORT}`);
  console.log(`🌍 Environment: ${env.NODE_ENV}`);
});
```

### src/routes/users.ts

```typescript
import { Router } from 'express';
import { z } from 'zod';
import { validateBody } from '@middleware/validate';
import { authenticate } from '@middleware/auth';
import { asyncHandler } from '@utils/asyncHandler';
import { NotFoundError } from '@errors/AppError';

const router = Router();

const createUserSchema = z.object({
  name: z.string().min(1, 'Name is required'),
  email: z.string().email('Invalid email'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
});

type CreateUserDto = z.infer<typeof createUserSchema>;

// GET /api/users/:id — public
router.get('/:id', asyncHandler<{ id: string }>(async (req, res) => {
  // Simulate DB lookup
  const user = await fakeDb.findById(req.params.id);

  if (!user) throw new NotFoundError('User');

  res.json({ success: true, data: user });
}));

// POST /api/users — public
router.post('/',
  validateBody(createUserSchema),
  asyncHandler<{}, {}, CreateUserDto>(async (req, res) => {
    const { name, email, password } = req.body; // fully typed ✅

    const user = await fakeDb.create({ name, email, password });

    res.status(201).json({ success: true, data: user });
  })
);

// DELETE /api/users/:id — protected
router.delete('/:id',
  authenticate,
  asyncHandler<{ id: string }>(async (req, res) => {
    // req.user is typed as User | undefined ✅
    if (req.user?.role !== 'admin') {
      res.status(403).json({ error: 'Admin access required' });
      return;
    }

    await fakeDb.delete(req.params.id);

    res.json({ success: true, message: 'User deleted' });
  })
);

export default router;
```

---

## Key Takeaways

- Use **tsx** for fast development, `tsc --noEmit` for type checking.
- Use **generics** on `Request<Params, ResBody, ReqBody, Query>` to type route handlers precisely.
- Use **declaration merging** (`declare global { namespace Express { interface Request {} } }`) to extend `req.user` and other custom properties.
- Use **RequestHandler** and **ErrorRequestHandler** types for clean middleware signatures.
- Validate and type environment variables at startup with **dotenv + zod** — and crash early if config is invalid.
- Include **@types/node** and configure `typeRoots` so built-in Node.js modules are fully typed.
- Use a **custom AppError class** with statusCode and code for structured error responses.
- Use the **asyncHandler** wrapper to avoid repetitive try/catch in every async route.
- Use **path aliases** in `tsconfig.json` with `tsc-alias` to replace long relative import paths.

---

*Week 9 — TypeScript Bootcamp | Node.js & Express Typing*
