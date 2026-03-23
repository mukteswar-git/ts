# Week 10: Database & API Patterns in TypeScript

> **Prerequisites:** Basic TypeScript knowledge (types, interfaces, generics), familiarity with REST APIs, Node.js fundamentals.

---

## Table of Contents

1. [Typed Prisma ORM](#1-typed-prisma-orm)
2. [Mongoose with TypeScript](#2-mongoose-with-typescript)
3. [Typing REST API Responses](#3-typing-rest-api-responses)
4. [Zod for Runtime Validation + Type Inference](#4-zod-for-runtime-validation--type-inference)
5. [Type-Safe Environment Config](#5-type-safe-environment-config)
6. [Typing JWT Payloads](#6-typing-jwt-payloads)
7. [OpenAPI + TypeScript Code Generation](#7-openapi--typescript-code-generation)
8. [tRPC Basics — End-to-End Type Safety](#8-trpc-basics--end-to-end-type-safety)
9. [Putting It All Together](#9-putting-it-all-together)

---

## 1. Typed Prisma ORM

Prisma is a next-generation ORM that auto-generates a fully-typed client from your schema. Unlike traditional ORMs, you never write types by hand — the Prisma CLI generates them directly from your `schema.prisma` file.

### Installation & Setup

```bash
npm install prisma @prisma/client
npx prisma init
```

### Defining a Schema

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String?
  role      Role     @default(USER)
  posts     Post[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model Post {
  id        Int      @id @default(autoincrement())
  title     String
  content   String?
  published Boolean  @default(false)
  author    User     @relation(fields: [authorId], references: [id])
  authorId  Int
}

enum Role {
  USER
  ADMIN
}
```

### Generating the Client

```bash
npx prisma generate
```

This command generates TypeScript types under `node_modules/@prisma/client`. You get types like `User`, `Post`, `Role`, `Prisma.UserCreateInput`, `Prisma.UserWhereUniqueInput`, and many more — all automatically.

### Using Auto-Generated Types

```typescript
import { PrismaClient, User, Post, Prisma } from "@prisma/client";

const prisma = new PrismaClient();

// ✅ Return type is inferred as User
async function getUserById(id: number): Promise<User | null> {
  return prisma.user.findUnique({ where: { id } });
}

// ✅ Prisma.UserCreateInput is generated for you
async function createUser(data: Prisma.UserCreateInput): Promise<User> {
  return prisma.user.create({ data });
}

// ✅ Select only specific fields — return type narrows automatically
async function getUserEmail(id: number) {
  // Return type: { id: number; email: string } | null
  return prisma.user.findUnique({
    where: { id },
    select: { id: true, email: true },
  });
}

// ✅ Include related models — type includes nested Post[]
async function getUserWithPosts(id: number) {
  return prisma.user.findUnique({
    where: { id },
    include: { posts: true },
  });
}
```

### Prisma Utility Types

Prisma provides utility types for partial selects, create/update inputs, and more:

```typescript
import { Prisma } from "@prisma/client";

// Type for creating a user (all required fields required, optional fields optional)
type CreateUserInput = Prisma.UserCreateInput;

// Type for updating a user (all fields optional)
type UpdateUserInput = Prisma.UserUpdateInput;

// Type for a User with its posts included
type UserWithPosts = Prisma.UserGetPayload<{
  include: { posts: true };
}>;

// Type for a post with only title and published fields
type PostSummary = Prisma.PostGetPayload<{
  select: { title: true; published: true };
}>;
```

### Transactions

```typescript
async function transferOwnership(
  postId: number,
  fromUserId: number,
  toUserId: number
) {
  // All operations are typed
  return prisma.$transaction(async (tx) => {
    const post = await tx.post.findFirst({
      where: { id: postId, authorId: fromUserId },
    });

    if (!post) throw new Error("Post not found or unauthorized");

    return tx.post.update({
      where: { id: postId },
      data: { authorId: toUserId },
    });
  });
}
```

> **Key Takeaway:** With Prisma, your database schema IS your type source of truth. Run `prisma generate` after every schema change and your TypeScript types update automatically.

---

## 2. Mongoose with TypeScript

Mongoose is a MongoDB ODM. Unlike Prisma, types are not auto-generated — you define them yourself and connect them to Mongoose schemas. TypeScript support has improved greatly in Mongoose 6+.

### Installation

```bash
npm install mongoose
npm install -D @types/mongoose  # Not needed for Mongoose 6+ (built-in types)
```

### Defining a Schema with Types

The modern approach uses a TypeScript interface alongside the Mongoose schema:

```typescript
import mongoose, { Schema, Document, Model } from "mongoose";

// Step 1: Define the plain TypeScript interface (the shape of your data)
interface IUser {
  name: string;
  email: string;
  age?: number;
  role: "user" | "admin";
  createdAt: Date;
}

// Step 2: Define the Document interface (includes Mongoose Document methods)
interface IUserDocument extends IUser, Document {
  getDisplayName(): string; // Custom instance method type
}

// Step 3: Define Model interface (for static methods)
interface IUserModel extends Model<IUserDocument> {
  findByEmail(email: string): Promise<IUserDocument | null>;
}

// Step 4: Create the Schema
const UserSchema = new Schema<IUserDocument, IUserModel>(
  {
    name: { type: String, required: true },
    email: { type: String, required: true, unique: true, lowercase: true },
    age: { type: Number, min: 0 },
    role: { type: String, enum: ["user", "admin"], default: "user" },
  },
  { timestamps: true }
);

// Step 5: Add instance methods
UserSchema.methods.getDisplayName = function (): string {
  return `${this.name} (${this.role})`;
};

// Step 6: Add static methods
UserSchema.statics.findByEmail = function (
  email: string
): Promise<IUserDocument | null> {
  return this.findOne({ email });
};

// Step 7: Create and export the Model
const User: IUserModel = mongoose.model<IUserDocument, IUserModel>(
  "User",
  UserSchema
);

export { User, IUser, IUserDocument };
```

### Using the Typed Model

```typescript
import { User, IUser } from "./models/User";

async function createUser(data: IUser) {
  const user = new User(data);
  await user.save();

  // ✅ Instance method is typed
  console.log(user.getDisplayName());
  return user;
}

async function findUser(email: string) {
  // ✅ Static method is typed, returns IUserDocument | null
  const user = await User.findByEmail(email);
  return user;
}

async function updateUserAge(id: string, age: number) {
  // ✅ Return type is IUserDocument | null
  return User.findByIdAndUpdate(
    id,
    { age },
    { new: true } // return updated doc
  );
}
```

### Using Mongoose's Built-in HydratedDocument (Mongoose 6+)

```typescript
import { Schema, model, HydratedDocument, InferSchemaType } from "mongoose";

const postSchema = new Schema({
  title: { type: String, required: true },
  content: String,
  published: { type: Boolean, default: false },
  tags: [String],
  authorId: { type: Schema.Types.ObjectId, ref: "User", required: true },
});

// Infer the type directly from the schema — no separate interface needed
type IPost = InferSchemaType<typeof postSchema>;

// HydratedDocument includes all Mongoose Document properties
type PostDocument = HydratedDocument<IPost>;

const Post = model("Post", postSchema);

async function getPublishedPosts(): Promise<PostDocument[]> {
  return Post.find({ published: true }).lean();
}
```

> **Key Takeaway:** Mongoose requires you to define types manually and wire them to your schema. Use `InferSchemaType` in newer Mongoose versions to reduce duplication, and always type your instance/static methods explicitly.

---

## 3. Typing REST API Responses

Consistent, typed API responses across your application prevent bugs, improve DX with autocompletion, and make error handling predictable.

### The Generic ApiResponse Pattern

```typescript
// types/api.ts

// Base success response
interface ApiSuccess<T> {
  success: true;
  data: T;
  message?: string;
}

// Base error response
interface ApiError {
  success: false;
  error: {
    code: string;
    message: string;
    details?: unknown;
  };
}

// Union type — a response is either success or error
type ApiResponse<T> = ApiSuccess<T> | ApiError;

// Paginated response
interface PaginatedData<T> {
  items: T[];
  total: number;
  page: number;
  pageSize: number;
  totalPages: number;
}

type PaginatedResponse<T> = ApiResponse<PaginatedData<T>>;
```

### Helper Functions for Consistent Responses

```typescript
// utils/response.ts
import { Response } from "express";
import { ApiResponse, ApiError } from "../types/api";

export function sendSuccess<T>(
  res: Response,
  data: T,
  statusCode = 200,
  message?: string
): void {
  res.status(statusCode).json({
    success: true,
    data,
    message,
  } satisfies ApiResponse<T>);
}

export function sendError(
  res: Response,
  code: string,
  message: string,
  statusCode = 400,
  details?: unknown
): void {
  res.status(statusCode).json({
    success: false,
    error: { code, message, details },
  } satisfies ApiResponse<never>);
}
```

### Typed Express Route Handlers

```typescript
import { Request, Response, RequestHandler } from "express";
import { ApiResponse, PaginatedResponse } from "../types/api";
import { User } from "../models/User";

// Define request/response types for each route
interface GetUsersQuery {
  page?: string;
  limit?: string;
  role?: "user" | "admin";
}

interface UserResponse {
  id: string;
  name: string;
  email: string;
  role: string;
}

// Fully typed route handler
export const getUsers: RequestHandler<
  {},                  // Route params
  ApiResponse<UserResponse[]>,  // Response body
  {},                  // Request body
  GetUsersQuery        // Query params
> = async (req, res) => {
  const page = parseInt(req.query.page ?? "1");
  const limit = parseInt(req.query.limit ?? "10");

  const users = await User.find(
    req.query.role ? { role: req.query.role } : {}
  )
    .skip((page - 1) * limit)
    .limit(limit)
    .select("name email role");

  sendSuccess(res, users.map((u) => ({
    id: u.id,
    name: u.name,
    email: u.email,
    role: u.role,
  })));
};
```

### Typed Fetch Utility for Clients

```typescript
// utils/fetchApi.ts
import { ApiResponse } from "../types/api";

async function apiFetch<T>(
  url: string,
  options?: RequestInit
): Promise<ApiResponse<T>> {
  const response = await fetch(url, {
    headers: { "Content-Type": "application/json" },
    ...options,
  });

  const json = await response.json();
  return json as ApiResponse<T>;
}

// Type guard to narrow the union
function isApiSuccess<T>(
  response: ApiResponse<T>
): response is { success: true; data: T; message?: string } {
  return response.success === true;
}

// Usage
async function loadUser(id: string) {
  const result = await apiFetch<UserResponse>(`/api/users/${id}`);

  if (isApiSuccess(result)) {
    // ✅ result.data is UserResponse here
    console.log(result.data.name);
  } else {
    // ✅ result.error is ApiError here
    console.error(result.error.message);
  }
}
```

---

## 4. Zod for Runtime Validation + Type Inference

TypeScript types only exist at compile time. When data arrives from HTTP requests, JSON files, or environment variables at runtime, you need Zod to validate the shape and infer TypeScript types from the same schema.

### Installation

```bash
npm install zod
```

### Basic Schema Definition and Type Inference

```typescript
import { z } from "zod";

// Define validation schema
const UserSchema = z.object({
  name: z.string().min(2, "Name must be at least 2 characters"),
  email: z.string().email("Invalid email address"),
  age: z.number().int().min(0).max(120).optional(),
  role: z.enum(["user", "admin"]).default("user"),
  website: z.string().url().nullable(),
});

// ✅ Infer TypeScript type from the schema — no duplication!
type User = z.infer<typeof UserSchema>;
// Equivalent to:
// type User = {
//   name: string;
//   email: string;
//   age?: number | undefined;
//   role: "user" | "admin";
//   website: string | null;
// }
```

### Parsing and Validation

```typescript
// .parse() throws if validation fails
try {
  const user = UserSchema.parse({
    name: "Alice",
    email: "alice@example.com",
    role: "admin",
    website: null,
  });
  // user is fully typed as User
} catch (error) {
  if (error instanceof z.ZodError) {
    console.log(error.errors);
    // [{ path: [...], message: "...", code: "..." }, ...]
  }
}

// .safeParse() returns a result object — never throws
const result = UserSchema.safeParse(requestBody);

if (result.success) {
  const user = result.data; // ✅ typed as User
} else {
  const errors = result.error.format(); // Nested error object
}
```

### Advanced Schemas

```typescript
// Nested objects
const AddressSchema = z.object({
  street: z.string(),
  city: z.string(),
  zip: z.string().regex(/^\d{5}$/, "ZIP must be 5 digits"),
  country: z.string().length(2, "Use ISO 2-letter country code"),
});

const ProfileSchema = z.object({
  user: UserSchema,
  address: AddressSchema.optional(),
  tags: z.array(z.string()).max(10, "Maximum 10 tags"),
  metadata: z.record(z.string(), z.unknown()), // { [key: string]: unknown }
});

type Profile = z.infer<typeof ProfileSchema>;

// Transformations — parse + transform in one step
const DateStringSchema = z
  .string()
  .datetime()
  .transform((str) => new Date(str)); // string in, Date out

// Refinements — custom validation logic
const PasswordSchema = z
  .string()
  .min(8)
  .refine((val) => /[A-Z]/.test(val), {
    message: "Must contain at least one uppercase letter",
  })
  .refine((val) => /[0-9]/.test(val), {
    message: "Must contain at least one number",
  });

// Discriminated unions
const EventSchema = z.discriminatedUnion("type", [
  z.object({ type: z.literal("click"), x: z.number(), y: z.number() }),
  z.object({ type: z.literal("keypress"), key: z.string() }),
  z.object({ type: z.literal("scroll"), delta: z.number() }),
]);

type Event = z.infer<typeof EventSchema>;
```

### Zod Middleware for Express

```typescript
import { Request, Response, NextFunction } from "express";
import { AnyZodObject, ZodError } from "zod";

// Reusable validation middleware factory
export function validate(schema: AnyZodObject) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const result = await schema.safeParseAsync({
      body: req.body,
      query: req.query,
      params: req.params,
    });

    if (!result.success) {
      return res.status(400).json({
        success: false,
        error: {
          code: "VALIDATION_ERROR",
          message: "Invalid request data",
          details: result.error.format(),
        },
      });
    }

    // Attach parsed (and transformed) data to request
    req.body = result.data.body;
    next();
  };
}

// Define route schema
const CreatePostSchema = z.object({
  body: z.object({
    title: z.string().min(1).max(200),
    content: z.string().optional(),
    published: z.boolean().default(false),
    tags: z.array(z.string()).optional(),
  }),
});

// Use in router
import express from "express";
const router = express.Router();

router.post("/posts", validate(CreatePostSchema), async (req, res) => {
  // req.body is now typed and validated
  const { title, content, published, tags } = req.body;
  // ...
});
```

> **Key Takeaway:** Zod is the bridge between the untyped runtime world and TypeScript's type system. Define once, validate at runtime, and infer types statically — all from the same schema.

---

## 5. Type-Safe Environment Config

Environment variables are strings by default — `process.env.PORT` is `string | undefined`. Without proper typing, you risk runtime crashes from missing or incorrectly typed config values.

### The Problem

```typescript
// ❌ Dangerous — PORT might be undefined, and it's always a string
const port = process.env.PORT; // string | undefined
app.listen(port); // Wrong — listen expects a number

// ❌ Also dangerous — no validation, fails silently
const dbUrl = process.env.DATABASE_URL!; // You asserted non-null, but is it really?
```

### Solution: Zod-Validated Environment Config

```typescript
// config/env.ts
import { z } from "zod";

const EnvSchema = z.object({
  // Server
  NODE_ENV: z.enum(["development", "test", "production"]).default("development"),
  PORT: z.string().transform(Number).pipe(z.number().int().positive()),

  // Database
  DATABASE_URL: z.string().url("DATABASE_URL must be a valid URL"),

  // Auth
  JWT_SECRET: z.string().min(32, "JWT_SECRET must be at least 32 characters"),
  JWT_EXPIRES_IN: z.string().default("7d"),

  // Optional external services
  SMTP_HOST: z.string().optional(),
  SMTP_PORT: z.string().transform(Number).pipe(z.number().int()).optional(),
  REDIS_URL: z.string().url().optional(),

  // Feature flags
  FEATURE_BETA: z
    .string()
    .transform((v) => v === "true")
    .default("false"),
});

// Type is inferred automatically
export type Env = z.infer<typeof EnvSchema>;

// Parse and validate at application startup
function loadEnv(): Env {
  const result = EnvSchema.safeParse(process.env);

  if (!result.success) {
    console.error("❌ Invalid environment configuration:");
    console.error(result.error.format());
    process.exit(1); // Fail fast — don't start the app with bad config
  }

  return result.data;
}

// Export a single validated env object
export const env = loadEnv();
```

### Using the Config

```typescript
import { env } from "./config/env";

// ✅ All values are typed correctly
app.listen(env.PORT, () => {
  console.log(`Server running on port ${env.PORT} in ${env.NODE_ENV} mode`);
});

const client = new PrismaClient({
  datasources: { db: { url: env.DATABASE_URL } },
});

if (env.FEATURE_BETA) {
  // Enable beta features
}
```

### Organizing Config by Domain

```typescript
// config/index.ts
import { env } from "./env";

export const config = {
  server: {
    port: env.PORT,
    nodeEnv: env.NODE_ENV,
    isDev: env.NODE_ENV === "development",
    isProd: env.NODE_ENV === "production",
  },
  db: {
    url: env.DATABASE_URL,
  },
  auth: {
    jwtSecret: env.JWT_SECRET,
    jwtExpiresIn: env.JWT_EXPIRES_IN,
  },
  mail: env.SMTP_HOST
    ? {
        host: env.SMTP_HOST,
        port: env.SMTP_PORT ?? 587,
      }
    : null,
} as const;
```

---

## 6. Typing JWT Payloads

JWTs are widely used for authentication, but `jwt.verify()` returns `string | JwtPayload` — a loosely typed object. Properly typing your JWT payload prevents accessing non-existent fields.

### Installation

```bash
npm install jsonwebtoken
npm install -D @types/jsonwebtoken
```

### Defining the Payload Type

```typescript
import jwt, { SignOptions, JwtPayload } from "jsonwebtoken";
import { env } from "../config/env";

// Define the shape of YOUR JWT payload
interface TokenPayload extends JwtPayload {
  userId: number;
  email: string;
  role: "user" | "admin";
  // Note: iat, exp, iss, sub come from JwtPayload
}

// Token types for access/refresh distinction
type AccessTokenPayload = TokenPayload & { type: "access" };
type RefreshTokenPayload = Pick<TokenPayload, "userId"> & {
  type: "refresh";
  tokenVersion: number;
};
```

### Signing Tokens

```typescript
function signAccessToken(payload: Omit<AccessTokenPayload, "type">): string {
  const fullPayload: AccessTokenPayload = { ...payload, type: "access" };

  return jwt.sign(fullPayload, env.JWT_SECRET, {
    expiresIn: env.JWT_EXPIRES_IN,
    issuer: "myapp",
    audience: "myapp-users",
  } satisfies SignOptions);
}

function signRefreshToken(
  payload: Omit<RefreshTokenPayload, "type">
): string {
  const fullPayload: RefreshTokenPayload = { ...payload, type: "refresh" };

  return jwt.sign(fullPayload, env.JWT_SECRET, {
    expiresIn: "30d",
  });
}
```

### Verifying Tokens with Type Safety

The key challenge: `jwt.verify()` returns `string | JwtPayload`. We need a typed wrapper:

```typescript
class TokenError extends Error {
  constructor(
    message: string,
    public readonly code: "EXPIRED" | "INVALID" | "MALFORMED"
  ) {
    super(message);
    this.name = "TokenError";
  }
}

function verifyToken<T extends JwtPayload>(token: string): T {
  try {
    const decoded = jwt.verify(token, env.JWT_SECRET, {
      issuer: "myapp",
      audience: "myapp-users",
    });

    if (typeof decoded === "string") {
      throw new TokenError("Token decoded as string", "MALFORMED");
    }

    return decoded as T;
  } catch (error) {
    if (error instanceof jwt.TokenExpiredError) {
      throw new TokenError("Token has expired", "EXPIRED");
    }
    if (error instanceof jwt.JsonWebTokenError) {
      throw new TokenError("Invalid token", "INVALID");
    }
    throw error;
  }
}

// Type-safe verify functions
function verifyAccessToken(token: string): AccessTokenPayload {
  const payload = verifyToken<AccessTokenPayload>(token);

  if (payload.type !== "access") {
    throw new TokenError("Not an access token", "INVALID");
  }

  return payload;
}

function verifyRefreshToken(token: string): RefreshTokenPayload {
  const payload = verifyToken<RefreshTokenPayload>(token);

  if (payload.type !== "refresh") {
    throw new TokenError("Not a refresh token", "INVALID");
  }

  return payload;
}
```

### Typed Auth Middleware for Express

```typescript
import { Request, Response, NextFunction } from "express";
import { AccessTokenPayload } from "../types/auth";

// Extend Express's Request to include the authenticated user
declare global {
  namespace Express {
    interface Request {
      user?: AccessTokenPayload;
    }
  }
}

export function authenticate(
  req: Request,
  res: Response,
  next: NextFunction
): void {
  const authHeader = req.headers.authorization;

  if (!authHeader?.startsWith("Bearer ")) {
    res.status(401).json({ success: false, error: { message: "No token provided" } });
    return;
  }

  const token = authHeader.slice(7); // Remove "Bearer "

  try {
    req.user = verifyAccessToken(token);
    next();
  } catch (error) {
    if (error instanceof TokenError) {
      res.status(401).json({
        success: false,
        error: { code: error.code, message: error.message },
      });
    } else {
      next(error);
    }
  }
}

export function requireRole(role: "user" | "admin") {
  return (req: Request, res: Response, next: NextFunction): void => {
    if (!req.user) {
      res.status(401).json({ success: false, error: { message: "Unauthenticated" } });
      return;
    }

    if (req.user.role !== role && req.user.role !== "admin") {
      res.status(403).json({ success: false, error: { message: "Insufficient permissions" } });
      return;
    }

    next();
  };
}

// Usage in routes
router.get("/admin/users", authenticate, requireRole("admin"), async (req, res) => {
  // ✅ req.user is AccessTokenPayload here
  console.log(`Admin ${req.user!.email} accessed user list`);
});
```

---

## 7. OpenAPI + TypeScript Code Generation

Instead of manually writing TypeScript types for third-party APIs or your own API docs, you can generate them automatically from an OpenAPI (Swagger) spec. This ensures your types always match the spec.

### Popular Tools

| Tool | Best For |
|---|---|
| `openapi-typescript` | Generating types from an OpenAPI spec |
| `orval` | Generating types + React Query / SWR hooks |
| `openapi-generator-cli` | Full client SDK generation (many languages) |

### Using `openapi-typescript`

```bash
npm install -D openapi-typescript
```

Generate types from a local or remote spec:

```bash
# From a local file
npx openapi-typescript ./api-spec.yaml -o ./src/types/api-generated.ts

# From a remote URL
npx openapi-typescript https://petstore3.swagger.io/api/v3/openapi.json -o ./src/types/petstore.ts
```

### Example Spec to Generated Types

Given this OpenAPI spec snippet:

```yaml
# api-spec.yaml
openapi: "3.0.0"
info:
  title: My API
  version: "1.0.0"
paths:
  /users/{id}:
    get:
      operationId: getUserById
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      responses:
        "200":
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/User"
        "404":
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"

components:
  schemas:
    User:
      type: object
      required: [id, email, name]
      properties:
        id:
          type: integer
        email:
          type: string
          format: email
        name:
          type: string
        role:
          type: string
          enum: [user, admin]
    Error:
      type: object
      required: [message]
      properties:
        message:
          type: string
        code:
          type: string
```

The generator produces:

```typescript
// src/types/api-generated.ts (auto-generated — do not edit manually)

export interface paths {
  "/users/{id}": {
    get: operations["getUserById"];
  };
}

export interface operations {
  getUserById: {
    parameters: {
      path: { id: number };
    };
    responses: {
      200: {
        content: {
          "application/json": components["schemas"]["User"];
        };
      };
      404: {
        content: {
          "application/json": components["schemas"]["Error"];
        };
      };
    };
  };
}

export interface components {
  schemas: {
    User: {
      id: number;
      email: string;
      name: string;
      role?: "user" | "admin";
    };
    Error: {
      message: string;
      code?: string;
    };
  };
}
```

### Using Generated Types

```typescript
import { components, operations } from "./types/api-generated";

// Use schema types directly
type User = components["schemas"]["User"];
type ApiError = components["schemas"]["Error"];

// Use operation response types
type GetUserResponse =
  operations["getUserById"]["responses"][200]["content"]["application/json"];

// Type-safe API client
async function getUser(id: number): Promise<User> {
  const response = await fetch(`/api/users/${id}`);

  if (!response.ok) {
    const error: ApiError = await response.json();
    throw new Error(error.message);
  }

  return response.json() as Promise<User>;
}
```

### Automating Generation in package.json

```json
{
  "scripts": {
    "generate:api": "openapi-typescript ./api-spec.yaml -o ./src/types/api-generated.ts",
    "prebuild": "npm run generate:api"
  }
}
```

> **Key Takeaway:** Treat OpenAPI specs as a contract. Generate types from the spec (or generate the spec from your code) and let automation keep your types in sync.

---

## 8. tRPC Basics — End-to-End Type Safety

tRPC eliminates the API layer entirely in full-stack TypeScript apps. You write server functions, and the client calls them with full type inference — no code generation, no manual type sharing.

### Installation

```bash
npm install @trpc/server @trpc/client @trpc/react-query zod
npm install @tanstack/react-query  # For React frontend
```

### Building the Server

```typescript
// server/trpc.ts — Initialize tRPC
import { initTRPC, TRPCError } from "@trpc/server";
import { z } from "zod";

// Context type (e.g., authenticated user, database connection)
interface Context {
  user?: { id: number; role: "user" | "admin" };
  prisma: PrismaClient;
}

// Initialize tRPC instance with context type
const t = initTRPC.context<Context>().create();

// Export reusable builders
export const router = t.router;
export const publicProcedure = t.procedure;

// Protected procedure — throws if not authenticated
export const protectedProcedure = t.procedure.use(({ ctx, next }) => {
  if (!ctx.user) {
    throw new TRPCError({ code: "UNAUTHORIZED" });
  }
  return next({ ctx: { ...ctx, user: ctx.user } }); // user is non-nullable after this
});

// Admin procedure
export const adminProcedure = protectedProcedure.use(({ ctx, next }) => {
  if (ctx.user.role !== "admin") {
    throw new TRPCError({ code: "FORBIDDEN" });
  }
  return next({ ctx });
});
```

```typescript
// server/routers/users.ts — User router
import { z } from "zod";
import { router, publicProcedure, protectedProcedure } from "../trpc";
import { TRPCError } from "@trpc/server";

export const usersRouter = router({
  // Query: getById
  getById: publicProcedure
    .input(z.object({ id: z.number().int().positive() }))
    .query(async ({ input, ctx }) => {
      // ✅ input.id is typed as number
      const user = await ctx.prisma.user.findUnique({
        where: { id: input.id },
      });

      if (!user) {
        throw new TRPCError({ code: "NOT_FOUND", message: "User not found" });
      }

      return user; // Return type is inferred from Prisma
    }),

  // Query: list with filters
  list: publicProcedure
    .input(
      z.object({
        page: z.number().int().positive().default(1),
        limit: z.number().int().min(1).max(100).default(10),
        role: z.enum(["user", "admin"]).optional(),
      })
    )
    .query(async ({ input, ctx }) => {
      const { page, limit, role } = input;
      const users = await ctx.prisma.user.findMany({
        where: role ? { role } : {},
        skip: (page - 1) * limit,
        take: limit,
      });
      const total = await ctx.prisma.user.count({ where: role ? { role } : {} });

      return { users, total, page, limit };
    }),

  // Mutation: create
  create: protectedProcedure
    .input(
      z.object({
        name: z.string().min(2),
        email: z.string().email(),
      })
    )
    .mutation(async ({ input, ctx }) => {
      // ✅ ctx.user is non-nullable here (guaranteed by protectedProcedure)
      return ctx.prisma.user.create({ data: input });
    }),

  // Mutation: delete (admin only)
  delete: adminProcedure
    .input(z.object({ id: z.number() }))
    .mutation(async ({ input, ctx }) => {
      return ctx.prisma.user.delete({ where: { id: input.id } });
    }),
});
```

```typescript
// server/routers/index.ts — Root router
import { router } from "../trpc";
import { usersRouter } from "./users";
import { postsRouter } from "./posts";

export const appRouter = router({
  users: usersRouter,
  posts: postsRouter,
});

// ✅ Export the router TYPE — this is what the client imports
export type AppRouter = typeof appRouter;
```

### Setting Up Express Adapter

```typescript
// server/index.ts
import express from "express";
import { createExpressMiddleware } from "@trpc/server/adapters/express";
import { appRouter } from "./routers";
import { createContext } from "./context";

const app = express();

app.use(
  "/trpc",
  createExpressMiddleware({
    router: appRouter,
    createContext, // Function that returns Context object
  })
);
```

### Using the Client (React)

```typescript
// client/trpc.ts — Client setup
import { createTRPCReact } from "@trpc/react-query";
import { httpBatchLink } from "@trpc/client";
import type { AppRouter } from "../server/routers"; // ✅ Import TYPE only

// Create the typed tRPC client hooks
export const trpc = createTRPCReact<AppRouter>();

// In your React root (App.tsx)
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

const queryClient = new QueryClient();
const trpcClient = trpc.createClient({
  links: [
    httpBatchLink({
      url: "http://localhost:3000/trpc",
      headers() {
        return { Authorization: `Bearer ${getToken()}` };
      },
    }),
  ],
});

export function App() {
  return (
    <trpc.Provider client={trpcClient} queryClient={queryClient}>
      <QueryClientProvider client={queryClient}>
        <UserList />
      </QueryClientProvider>
    </trpc.Provider>
  );
}
```

```typescript
// client/components/UserList.tsx
import { trpc } from "../trpc";

export function UserList() {
  // ✅ Full type inference — data is typed as the router's return type
  const { data, isLoading, error } = trpc.users.list.useQuery({
    page: 1,
    limit: 10,
  });

  const createMutation = trpc.users.create.useMutation({
    onSuccess: () => {
      // Invalidate and refetch
      trpc.useUtils().users.list.invalidate();
    },
  });

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return (
    <div>
      {data?.users.map((user) => (
        // ✅ user.name, user.email, user.role are all typed
        <div key={user.id}>{user.name} — {user.email}</div>
      ))}

      <button
        onClick={() =>
          createMutation.mutate({ name: "Alice", email: "alice@example.com" })
        }
      >
        Add User
      </button>
    </div>
  );
}
```

> **Key Takeaway:** tRPC gives you end-to-end type safety without code generation. Change your server function's return type and your client code immediately shows a TypeScript error if it's incompatible.

---

## 9. Putting It All Together

Here's how all the patterns from this week combine in a real feature — a user registration endpoint:

```typescript
// The complete flow for a "register user" feature

// 1. Zod schema defines the shape (used for both validation AND types)
const RegisterSchema = z.object({
  name: z.string().min(2).max(100),
  email: z.string().email(),
  password: z.string().min(8),
});
type RegisterInput = z.infer<typeof RegisterSchema>;

// 2. The response shape
const AuthResponseSchema = z.object({
  accessToken: z.string(),
  user: z.object({ id: z.number(), name: z.string(), email: z.string() }),
});
type AuthResponse = z.infer<typeof AuthResponseSchema>;

// 3. Environment config used safely
import { config } from "./config";

// 4. Prisma handles DB with full types
async function registerUser(input: RegisterInput): Promise<AuthResponse> {
  const existing = await prisma.user.findUnique({
    where: { email: input.email },
  });

  if (existing) throw new Error("Email already in use");

  const user = await prisma.user.create({
    data: {
      name: input.name,
      email: input.email,
      passwordHash: await hashPassword(input.password),
    },
  });

  // 5. JWT payload is typed
  const tokenPayload: Omit<AccessTokenPayload, "type"> = {
    userId: user.id,
    email: user.email,
    role: "user",
  };

  const accessToken = signAccessToken(tokenPayload);

  return { accessToken, user: { id: user.id, name: user.name, email: user.email } };
}

// 6. In an Express route — validated with Zod middleware
router.post("/auth/register", validate(z.object({ body: RegisterSchema })), async (req, res) => {
  try {
    const result = await registerUser(req.body as RegisterInput);
    sendSuccess(res, result, 201);
  } catch (error) {
    sendError(res, "REGISTRATION_FAILED", (error as Error).message, 409);
  }
});

// 7. Or in tRPC — fully typed end-to-end
export const authRouter = router({
  register: publicProcedure
    .input(RegisterSchema)
    .mutation(async ({ input }) => {
      return registerUser(input);
    }),
});
```

---

## Quick Reference: When to Use What

| Pattern | Use When |
|---|---|
| **Prisma types** | PostgreSQL/MySQL/SQLite with relational data; you want auto-generated types |
| **Mongoose + TS** | MongoDB with document-oriented data; existing Mongoose codebase |
| **Generic ApiResponse** | Building a REST API consumed by multiple clients |
| **Zod** | Validating any external data at runtime (HTTP requests, env vars, JSON files) |
| **Env config with Zod** | Always — never use `process.env.X` directly in your code |
| **Typed JWT** | Any authentication system using JWTs |
| **OpenAPI codegen** | Integrating with third-party APIs or maintaining an API spec |
| **tRPC** | Full-stack TypeScript (Next.js, Remix, etc.) where client and server share a codebase |

---

## Exercises

**Exercise 1 — Prisma:**
Extend the Prisma schema to add a `Comment` model related to both `User` and `Post`. Write a typed function that returns a post with its author and all comments (including each comment's author).

**Exercise 2 — Zod:**
Write a Zod schema for a checkout form with: customer name, email, shipping address (nested object), items array (each with productId and quantity), and an optional coupon code. Infer the TypeScript type and write a validation function.

**Exercise 3 — JWT:**
Create a token rotation system with `signRefreshToken` and `verifyRefreshToken`. Add a `tokenVersion` field to user records in Prisma and invalidate all tokens for a user by incrementing their `tokenVersion`.

**Exercise 4 — tRPC:**
Add a `posts` router to the tRPC app with procedures: `list` (public, with pagination), `getById` (public), `create` (protected), `publish` (protected, only the author can publish), and `delete` (admin only).

**Exercise 5 — Full Integration:**
Build a typed Express middleware that validates API keys (stored in a Prisma table), checks a rate limit (stored in Redis), and attaches the API key's associated user to the request — all with full TypeScript types throughout.

---

*End of Week 10 — Database & API Patterns in TypeScript*
