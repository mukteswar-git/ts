# Week 12: Patterns & Architecture in TypeScript

> A comprehensive guide to production-grade TypeScript design patterns, from the Repository pattern to full DDD and codebase migration strategies.

---

## Table of Contents

1. [Repository Pattern with Generics](#1-repository-pattern-with-generics)
2. [Service Layer Typing](#2-service-layer-typing)
3. [Type-Safe Error Handling — The Result Pattern](#3-type-safe-error-handling--the-result-pattern)
4. [Builder Pattern in TypeScript](#4-builder-pattern-in-typescript)
5. [Factory Functions with Overloads](#5-factory-functions-with-overloads)
6. [Dependency Injection Patterns](#6-dependency-injection-patterns)
7. [Domain-Driven Design Types](#7-domain-driven-design-types)
8. [Opaque Types (Branded Types)](#8-opaque-types-branded-types)
9. [Migrating a JS Codebase to TypeScript](#9-migrating-a-js-codebase-to-typescript)
10. [Gradual Adoption Strategy with `allowJs`](#10-gradual-adoption-strategy-with-allowjs)

---

## 1. Repository Pattern with Generics

The **Repository pattern** abstracts the data-access layer so your business logic never talks directly to a database, ORM, or API. Generics make it reusable across every entity in your domain.

### 1.1 The Base Interface

```typescript
// types/repository.ts

export interface Repository<T, ID = string> {
  findById(id: ID): Promise<T | null>;
  findAll(filter?: Partial<T>): Promise<T[]>;
  save(entity: T): Promise<T>;
  update(id: ID, partial: Partial<T>): Promise<T | null>;
  delete(id: ID): Promise<boolean>;
  count(filter?: Partial<T>): Promise<number>;
}
```

### 1.2 An Abstract Base Repository

Provide shared logic (pagination, sorting, audit timestamps) once, then let concrete classes extend it.

```typescript
// repositories/base.repository.ts

import { Repository } from '../types/repository';

export interface PaginationOptions {
  page: number;
  limit: number;
  sortBy?: string;
  sortOrder?: 'asc' | 'desc';
}

export interface PaginatedResult<T> {
  data: T[];
  total: number;
  page: number;
  limit: number;
  totalPages: number;
  hasNext: boolean;
  hasPrev: boolean;
}

export abstract class BaseRepository<T extends { id: string }, ID = string>
  implements Repository<T, ID>
{
  protected abstract entities: Map<string, T>;

  abstract findById(id: ID): Promise<T | null>;
  abstract findAll(filter?: Partial<T>): Promise<T[]>;
  abstract save(entity: T): Promise<T>;
  abstract update(id: ID, partial: Partial<T>): Promise<T | null>;
  abstract delete(id: ID): Promise<boolean>;

  async count(filter?: Partial<T>): Promise<number> {
    const all = await this.findAll(filter);
    return all.length;
  }

  async paginate(
    options: PaginationOptions,
    filter?: Partial<T>
  ): Promise<PaginatedResult<T>> {
    const { page, limit, sortBy, sortOrder = 'asc' } = options;
    let all = await this.findAll(filter);

    if (sortBy) {
      all = all.sort((a, b) => {
        const aVal = (a as any)[sortBy];
        const bVal = (b as any)[sortBy];
        const cmp = aVal < bVal ? -1 : aVal > bVal ? 1 : 0;
        return sortOrder === 'asc' ? cmp : -cmp;
      });
    }

    const total = all.length;
    const totalPages = Math.ceil(total / limit);
    const start = (page - 1) * limit;
    const data = all.slice(start, start + limit);

    return {
      data,
      total,
      page,
      limit,
      totalPages,
      hasNext: page < totalPages,
      hasPrev: page > 1,
    };
  }
}
```

### 1.3 A Concrete In-Memory Repository

```typescript
// repositories/user.repository.ts

import { BaseRepository } from './base.repository';

export interface User {
  id: string;
  email: string;
  name: string;
  role: 'admin' | 'user' | 'moderator';
  createdAt: Date;
  updatedAt: Date;
}

export class UserRepository extends BaseRepository<User> {
  protected entities = new Map<string, User>();

  async findById(id: string): Promise<User | null> {
    return this.entities.get(id) ?? null;
  }

  async findByEmail(email: string): Promise<User | null> {
    for (const user of this.entities.values()) {
      if (user.email === email) return user;
    }
    return null;
  }

  async findAll(filter?: Partial<User>): Promise<User[]> {
    let users = Array.from(this.entities.values());
    if (filter) {
      users = users.filter((u) =>
        (Object.keys(filter) as (keyof User)[]).every(
          (key) => u[key] === filter[key]
        )
      );
    }
    return users;
  }

  async save(user: User): Promise<User> {
    this.entities.set(user.id, { ...user, updatedAt: new Date() });
    return this.entities.get(user.id)!;
  }

  async update(id: string, partial: Partial<User>): Promise<User | null> {
    const existing = await this.findById(id);
    if (!existing) return null;
    const updated: User = { ...existing, ...partial, id, updatedAt: new Date() };
    this.entities.set(id, updated);
    return updated;
  }

  async delete(id: string): Promise<boolean> {
    return this.entities.delete(id);
  }
}
```

### 1.4 A Database-Backed Repository (Prisma example)

```typescript
// repositories/user.prisma.repository.ts

import { PrismaClient, User as PrismaUser } from '@prisma/client';
import { Repository } from '../types/repository';
import { User } from './user.repository';

export class PrismaUserRepository implements Repository<User> {
  constructor(private readonly prisma: PrismaClient) {}

  private toDomain(p: PrismaUser): User {
    return {
      id: p.id,
      email: p.email,
      name: p.name,
      role: p.role as User['role'],
      createdAt: p.createdAt,
      updatedAt: p.updatedAt,
    };
  }

  async findById(id: string): Promise<User | null> {
    const user = await this.prisma.user.findUnique({ where: { id } });
    return user ? this.toDomain(user) : null;
  }

  async findAll(filter?: Partial<User>): Promise<User[]> {
    const users = await this.prisma.user.findMany({ where: filter as any });
    return users.map(this.toDomain);
  }

  async save(entity: User): Promise<User> {
    const user = await this.prisma.user.upsert({
      where: { id: entity.id },
      create: entity as any,
      update: entity as any,
    });
    return this.toDomain(user);
  }

  async update(id: string, partial: Partial<User>): Promise<User | null> {
    try {
      const user = await this.prisma.user.update({ where: { id }, data: partial as any });
      return this.toDomain(user);
    } catch {
      return null;
    }
  }

  async delete(id: string): Promise<boolean> {
    try {
      await this.prisma.user.delete({ where: { id } });
      return true;
    } catch {
      return false;
    }
  }

  async count(filter?: Partial<User>): Promise<number> {
    return this.prisma.user.count({ where: filter as any });
  }
}
```

---

## 2. Service Layer Typing

The service layer contains **business logic**. It sits between HTTP controllers and repositories and should be fully typed so callers know exactly what they receive.

### 2.1 DTOs (Data Transfer Objects)

```typescript
// dto/user.dto.ts

export interface CreateUserDTO {
  email: string;
  name: string;
  password: string;
  role?: 'admin' | 'user' | 'moderator';
}

export interface UpdateUserDTO {
  name?: string;
  email?: string;
  role?: 'admin' | 'user' | 'moderator';
}

export interface UserResponseDTO {
  id: string;
  email: string;
  name: string;
  role: string;
  createdAt: string; // ISO 8601 — never expose Date objects to clients
}
```

### 2.2 A Typed Service Interface

```typescript
// services/user.service.interface.ts

import { CreateUserDTO, UpdateUserDTO, UserResponseDTO } from '../dto/user.dto';
import { PaginatedResult, PaginationOptions } from '../repositories/base.repository';
import { Result } from '../types/result'; // covered in Section 3

export interface IUserService {
  createUser(dto: CreateUserDTO): Promise<Result<UserResponseDTO>>;
  getUserById(id: string): Promise<Result<UserResponseDTO>>;
  updateUser(id: string, dto: UpdateUserDTO): Promise<Result<UserResponseDTO>>;
  deleteUser(id: string): Promise<Result<void>>;
  listUsers(pagination: PaginationOptions): Promise<Result<PaginatedResult<UserResponseDTO>>>;
}
```

### 2.3 Implementing the Service

```typescript
// services/user.service.ts

import { v4 as uuidv4 } from 'uuid';
import bcrypt from 'bcrypt';
import { IUserService } from './user.service.interface';
import { UserRepository } from '../repositories/user.repository';
import { CreateUserDTO, UpdateUserDTO, UserResponseDTO } from '../dto/user.dto';
import { PaginatedResult, PaginationOptions } from '../repositories/base.repository';
import { Result, ok, err } from '../types/result';

export class UserService implements IUserService {
  constructor(private readonly userRepo: UserRepository) {}

  private toDTO(user: { id: string; email: string; name: string; role: string; createdAt: Date }): UserResponseDTO {
    return {
      id: user.id,
      email: user.email,
      name: user.name,
      role: user.role,
      createdAt: user.createdAt.toISOString(),
    };
  }

  async createUser(dto: CreateUserDTO): Promise<Result<UserResponseDTO>> {
    const existing = await this.userRepo.findByEmail(dto.email);
    if (existing) {
      return err({ code: 'EMAIL_TAKEN', message: `Email ${dto.email} is already in use` });
    }

    const passwordHash = await bcrypt.hash(dto.password, 12);
    const user = await this.userRepo.save({
      id: uuidv4(),
      email: dto.email,
      name: dto.name,
      role: dto.role ?? 'user',
      createdAt: new Date(),
      updatedAt: new Date(),
      // passwordHash stored separately in real app
    } as any);

    return ok(this.toDTO(user));
  }

  async getUserById(id: string): Promise<Result<UserResponseDTO>> {
    const user = await this.userRepo.findById(id);
    if (!user) {
      return err({ code: 'NOT_FOUND', message: `User ${id} not found` });
    }
    return ok(this.toDTO(user));
  }

  async updateUser(id: string, dto: UpdateUserDTO): Promise<Result<UserResponseDTO>> {
    const updated = await this.userRepo.update(id, dto);
    if (!updated) {
      return err({ code: 'NOT_FOUND', message: `User ${id} not found` });
    }
    return ok(this.toDTO(updated));
  }

  async deleteUser(id: string): Promise<Result<void>> {
    const deleted = await this.userRepo.delete(id);
    if (!deleted) {
      return err({ code: 'NOT_FOUND', message: `User ${id} not found` });
    }
    return ok(undefined);
  }

  async listUsers(pagination: PaginationOptions): Promise<Result<PaginatedResult<UserResponseDTO>>> {
    const result = await this.userRepo.paginate(pagination);
    return ok({ ...result, data: result.data.map(this.toDTO.bind(this)) });
  }
}
```

---

## 3. Type-Safe Error Handling — The Result Pattern

Throwing exceptions mixes control flow and forces callers to read documentation to know what might fail. The **Result type** makes errors first-class citizens of your type system.

### 3.1 Defining `Result<T, E>`

```typescript
// types/result.ts

export type AppError = {
  code: string;
  message: string;
  details?: Record<string, unknown>;
};

export type Result<T, E = AppError> =
  | { success: true; data: T }
  | { success: false; error: E };

// Constructor helpers
export const ok = <T>(data: T): Result<T, never> => ({ success: true, data });
export const err = <E = AppError>(error: E): Result<never, E> => ({ success: false, error });

// Type guards
export const isOk = <T, E>(r: Result<T, E>): r is { success: true; data: T } => r.success;
export const isErr = <T, E>(r: Result<T, E>): r is { success: false; error: E } => !r.success;
```

### 3.2 Utility Functions

```typescript
// types/result.utils.ts

import { Result, ok, err, isOk } from './result';

/** Transform a successful result */
export const map = <T, U, E>(
  result: Result<T, E>,
  fn: (data: T) => U
): Result<U, E> => (isOk(result) ? ok(fn(result.data)) : result);

/** Chain results (flatMap / andThen) */
export const andThen = <T, U, E>(
  result: Result<T, E>,
  fn: (data: T) => Result<U, E>
): Result<U, E> => (isOk(result) ? fn(result.data) : result);

/** Recover from an error */
export const orElse = <T, E, F>(
  result: Result<T, E>,
  fn: (error: E) => Result<T, F>
): Result<T, F> => (isOk(result) ? (result as unknown as Result<T, F>) : fn(result.error));

/** Unwrap or throw */
export const unwrap = <T, E>(result: Result<T, E>): T => {
  if (isOk(result)) return result.data;
  throw new Error(`Unwrap failed: ${JSON.stringify(result.error)}`);
};

/** Wrap a promise that may throw into a Result */
export const tryCatch = async <T>(
  fn: () => Promise<T>,
  mapError?: (e: unknown) => { code: string; message: string }
): Promise<Result<T>> => {
  try {
    return ok(await fn());
  } catch (e) {
    return err(
      mapError?.(e) ?? {
        code: 'UNEXPECTED_ERROR',
        message: e instanceof Error ? e.message : 'Unknown error',
      }
    );
  }
};
```

### 3.3 Specific Error Types with Discriminated Unions

```typescript
// types/domain-errors.ts

export type NotFoundError = {
  code: 'NOT_FOUND';
  message: string;
  resourceType: string;
  resourceId: string;
};

export type ValidationError = {
  code: 'VALIDATION_ERROR';
  message: string;
  fields: Record<string, string[]>;
};

export type AuthorizationError = {
  code: 'UNAUTHORIZED' | 'FORBIDDEN';
  message: string;
};

export type DatabaseError = {
  code: 'DATABASE_ERROR';
  message: string;
  originalError?: unknown;
};

export type DomainError =
  | NotFoundError
  | ValidationError
  | AuthorizationError
  | DatabaseError;

// Usage — exhaustive error handling
import { Result } from './result';

function handleResult(result: Result<string, DomainError>): string {
  if (!result.success) {
    switch (result.error.code) {
      case 'NOT_FOUND':
        return `404: ${result.error.resourceType} ${result.error.resourceId} missing`;
      case 'VALIDATION_ERROR':
        return `400: ${JSON.stringify(result.error.fields)}`;
      case 'UNAUTHORIZED':
      case 'FORBIDDEN':
        return `401/403: ${result.error.message}`;
      case 'DATABASE_ERROR':
        return `500: Database problem`;
      // TypeScript will error if a new case is added but not handled here
    }
  }
  return result.data;
}
```

---

## 4. Builder Pattern in TypeScript

The Builder pattern constructs complex objects step-by-step and prevents invalid intermediate states from ever being used.

### 4.1 A Basic Builder

```typescript
// builders/query.builder.ts

interface QueryConfig {
  table: string;
  conditions: string[];
  columns: string[];
  orderBy?: { column: string; direction: 'ASC' | 'DESC' };
  limit?: number;
  offset?: number;
  joins: string[];
}

export class QueryBuilder {
  private config: QueryConfig = {
    table: '',
    conditions: [],
    columns: ['*'],
    joins: [],
  };

  from(table: string): this {
    this.config.table = table;
    return this;
  }

  select(...columns: string[]): this {
    this.config.columns = columns;
    return this;
  }

  where(condition: string): this {
    this.config.conditions.push(condition);
    return this;
  }

  join(table: string, on: string, type: 'INNER' | 'LEFT' | 'RIGHT' = 'INNER'): this {
    this.config.joins.push(`${type} JOIN ${table} ON ${on}`);
    return this;
  }

  orderBy(column: string, direction: 'ASC' | 'DESC' = 'ASC'): this {
    this.config.orderBy = { column, direction };
    return this;
  }

  limit(n: number): this {
    this.config.limit = n;
    return this;
  }

  offset(n: number): this {
    this.config.offset = n;
    return this;
  }

  build(): string {
    if (!this.config.table) throw new Error('Table is required');

    const parts: string[] = [
      `SELECT ${this.config.columns.join(', ')}`,
      `FROM ${this.config.table}`,
      ...this.config.joins,
    ];

    if (this.config.conditions.length > 0) {
      parts.push(`WHERE ${this.config.conditions.join(' AND ')}`);
    }
    if (this.config.orderBy) {
      parts.push(`ORDER BY ${this.config.orderBy.column} ${this.config.orderBy.direction}`);
    }
    if (this.config.limit !== undefined) parts.push(`LIMIT ${this.config.limit}`);
    if (this.config.offset !== undefined) parts.push(`OFFSET ${this.config.offset}`);

    return parts.join(' ');
  }
}

// Usage
const query = new QueryBuilder()
  .from('users')
  .select('users.id', 'users.name', 'orders.total')
  .join('orders', 'orders.user_id = users.id', 'LEFT')
  .where('users.role = "admin"')
  .where('users.createdAt > "2024-01-01"')
  .orderBy('users.name')
  .limit(20)
  .offset(40)
  .build();
```

### 4.2 Type-State Builder (Compile-Time Validation)

The type-state pattern uses phantom types to ensure the builder is used in the correct order at compile time.

```typescript
// builders/http-request.builder.ts

type HasMethod = { method: true };
type HasUrl    = { url: true };
type Empty     = Record<string, never>;

class HttpRequestBuilder<State extends Record<string, boolean> = Empty> {
  private _method?: string;
  private _url?: string;
  private _headers: Record<string, string> = {};
  private _body?: unknown;

  method<M extends 'GET' | 'POST' | 'PUT' | 'DELETE' | 'PATCH'>(
    m: M
  ): HttpRequestBuilder<State & HasMethod> {
    this._method = m;
    return this as any;
  }

  url(u: string): HttpRequestBuilder<State & HasUrl> {
    this._url = u;
    return this as any;
  }

  header(key: string, value: string): this {
    this._headers[key] = value;
    return this;
  }

  body(data: unknown): this {
    this._body = data;
    return this;
  }

  // build() is only callable when both method AND url have been set
  build(
    this: HttpRequestBuilder<HasMethod & HasUrl>
  ): { method: string; url: string; headers: Record<string, string>; body?: unknown } {
    return {
      method: this._method!,
      url: this._url!,
      headers: this._headers,
      body: this._body,
    };
  }
}

// ✅ Compiles
const request = new HttpRequestBuilder()
  .method('POST')
  .url('https://api.example.com/users')
  .header('Content-Type', 'application/json')
  .body({ name: 'Alice' })
  .build();

// ❌ TypeScript error: .build() is missing
// new HttpRequestBuilder().method('GET').build();
```

---

## 5. Factory Functions with Overloads

Overloads let one function return **different types** based on different inputs — all type-safe, no casts needed by callers.

### 5.1 Basic Overloads

```typescript
// factories/logger.factory.ts

type LogLevel = 'debug' | 'info' | 'warn' | 'error';

interface BaseLogger {
  log(level: LogLevel, message: string): void;
}

interface ConsoleLogger extends BaseLogger {
  type: 'console';
  setPrefix(prefix: string): void;
}

interface FileLogger extends BaseLogger {
  type: 'file';
  flush(): Promise<void>;
  getPath(): string;
}

interface RemoteLogger extends BaseLogger {
  type: 'remote';
  connect(): Promise<void>;
  disconnect(): Promise<void>;
}

// Overload signatures
function createLogger(type: 'console'): ConsoleLogger;
function createLogger(type: 'file', path: string): FileLogger;
function createLogger(type: 'remote', endpoint: string, apiKey: string): RemoteLogger;

// Single implementation signature
function createLogger(
  type: 'console' | 'file' | 'remote',
  ...args: string[]
): ConsoleLogger | FileLogger | RemoteLogger {
  switch (type) {
    case 'console': {
      let prefix = '';
      return {
        type: 'console',
        setPrefix: (p) => { prefix = p; },
        log: (level, message) => console.log(`[${level.toUpperCase()}]${prefix ? ` ${prefix}` : ''}: ${message}`),
      };
    }
    case 'file': {
      const [path] = args;
      const buffer: string[] = [];
      return {
        type: 'file',
        getPath: () => path,
        log: (level, message) => buffer.push(`[${level}] ${message}`),
        flush: async () => { /* write buffer to disk */ buffer.length = 0; },
      };
    }
    case 'remote': {
      const [endpoint, apiKey] = args;
      return {
        type: 'remote',
        log: (level, message) => { /* send to remote */ },
        connect: async () => { /* open WS */ },
        disconnect: async () => { /* close WS */ },
      };
    }
  }
}

// Callers get the exact right type — no casting needed
const consoleLogger = createLogger('console');        // ConsoleLogger
consoleLogger.setPrefix('[App]');

const fileLogger = createLogger('file', '/var/log/app.log'); // FileLogger
await fileLogger.flush();

const remoteLogger = createLogger('remote', 'wss://logs.example.com', 'sk-123'); // RemoteLogger
await remoteLogger.connect();
```

### 5.2 Generic Factory with Type Mapping

```typescript
// factories/validator.factory.ts

interface StringValidator  { type: 'string';  minLength?: number; maxLength?: number; pattern?: RegExp }
interface NumberValidator  { type: 'number';  min?: number; max?: number; integer?: boolean }
interface DateValidator    { type: 'date';    minDate?: Date; maxDate?: Date }
interface BooleanValidator { type: 'boolean' }

type ValidatorConfig =
  | StringValidator
  | NumberValidator
  | DateValidator
  | BooleanValidator;

type InferType<C extends ValidatorConfig> =
  C extends StringValidator  ? string  :
  C extends NumberValidator  ? number  :
  C extends DateValidator    ? Date    :
  C extends BooleanValidator ? boolean :
  never;

interface Validator<T> {
  validate(value: unknown): value is T;
  parse(value: unknown): T;
}

function createValidator<C extends ValidatorConfig>(config: C): Validator<InferType<C>> {
  return {
    validate(value: unknown): value is InferType<C> {
      try { this.parse(value); return true; }
      catch { return false; }
    },
    parse(value: unknown): InferType<C> {
      switch (config.type) {
        case 'string': {
          if (typeof value !== 'string') throw new Error('Expected string');
          if (config.minLength && value.length < config.minLength)
            throw new Error(`Min length is ${config.minLength}`);
          return value as InferType<C>;
        }
        case 'number': {
          if (typeof value !== 'number') throw new Error('Expected number');
          if (config.min !== undefined && value < config.min) throw new Error(`Min is ${config.min}`);
          if (config.integer && !Number.isInteger(value)) throw new Error('Expected integer');
          return value as InferType<C>;
        }
        case 'date': {
          const d = value instanceof Date ? value : new Date(value as string);
          if (isNaN(d.getTime())) throw new Error('Invalid date');
          return d as InferType<C>;
        }
        case 'boolean': {
          if (typeof value !== 'boolean') throw new Error('Expected boolean');
          return value as InferType<C>;
        }
      }
    },
  };
}

// Usage — fully typed
const nameValidator    = createValidator({ type: 'string', minLength: 2, maxLength: 50 });
const ageValidator     = createValidator({ type: 'number', min: 0, max: 150, integer: true });
const birthValidator   = createValidator({ type: 'date' });

const name: string  = nameValidator.parse('Alice');
const age:  number  = ageValidator.parse(30);
const dob:  Date    = birthValidator.parse('1990-01-01');
```

---

## 6. Dependency Injection Patterns

DI decouples components from their dependencies, making code testable and composable.

### 6.1 Constructor Injection (Manual DI)

```typescript
// di/manual.ts

// Define interfaces (contracts), not implementations
interface IEmailService {
  sendEmail(to: string, subject: string, body: string): Promise<void>;
}

interface IUserRepository {
  findById(id: string): Promise<{ id: string; email: string; name: string } | null>;
}

interface ILogger {
  info(message: string): void;
  error(message: string, error?: unknown): void;
}

// Consumers depend on interfaces
class NotificationService {
  constructor(
    private readonly emailService: IEmailService,
    private readonly userRepo: IUserRepository,
    private readonly logger: ILogger
  ) {}

  async notifyUser(userId: string, message: string): Promise<void> {
    const user = await this.userRepo.findById(userId);
    if (!user) {
      this.logger.error(`User ${userId} not found for notification`);
      return;
    }

    await this.emailService.sendEmail(user.email, 'Notification', message);
    this.logger.info(`Notified user ${userId}`);
  }
}

// Compose in the application root ("composition root")
class ProductionEmailService implements IEmailService {
  async sendEmail(to: string, subject: string, body: string): Promise<void> {
    // Real SMTP / SendGrid call
  }
}

class ConsoleLogger implements ILogger {
  info(msg: string)  { console.log(`[INFO]  ${msg}`); }
  error(msg: string, err?: unknown) { console.error(`[ERROR] ${msg}`, err); }
}

const notificationService = new NotificationService(
  new ProductionEmailService(),
  new UserRepository(),   // from Section 1
  new ConsoleLogger()
);
```

### 6.2 A Lightweight DI Container

```typescript
// di/container.ts

type Constructor<T = unknown> = new (...args: any[]) => T;
type Token<T> = string | symbol | Constructor<T>;

class Container {
  private bindings = new Map<Token<any>, () => any>();
  private singletons = new Map<Token<any>, any>();

  bind<T>(token: Token<T>, factory: () => T): this {
    this.bindings.set(token, factory);
    return this;
  }

  singleton<T>(token: Token<T>, factory: () => T): this {
    return this.bind(token, () => {
      if (!this.singletons.has(token)) {
        this.singletons.set(token, factory());
      }
      return this.singletons.get(token);
    });
  }

  resolve<T>(token: Token<T>): T {
    const factory = this.bindings.get(token);
    if (!factory) throw new Error(`No binding found for token: ${String(token)}`);
    return factory();
  }
}

// Token symbols (avoid string collisions)
const TOKENS = {
  Logger:              Symbol('ILogger'),
  EmailService:        Symbol('IEmailService'),
  UserRepository:      Symbol('IUserRepository'),
  NotificationService: Symbol('NotificationService'),
} as const;

// Wire up
const container = new Container();

container.singleton(TOKENS.Logger, () => new ConsoleLogger());
container.singleton(TOKENS.EmailService, () => new ProductionEmailService());
container.singleton(TOKENS.UserRepository, () => new UserRepository());
container.bind(TOKENS.NotificationService, () =>
  new NotificationService(
    container.resolve<IEmailService>(TOKENS.EmailService),
    container.resolve<IUserRepository>(TOKENS.UserRepository),
    container.resolve<ILogger>(TOKENS.Logger)
  )
);

const svc = container.resolve<NotificationService>(TOKENS.NotificationService);
```

---

## 7. Domain-Driven Design Types

DDD provides a rich vocabulary for structuring your type system around the business domain.

### 7.1 Value Objects

Value objects are **immutable** and compared by their **value**, not their identity.

```typescript
// domain/value-objects.ts

class Email {
  private readonly _value: string;

  constructor(email: string) {
    const regex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!regex.test(email)) throw new Error(`Invalid email: ${email}`);
    this._value = email.toLowerCase();
  }

  get value(): string { return this._value; }
  equals(other: Email): boolean { return this._value === other._value; }
  toString(): string { return this._value; }
}

class Money {
  constructor(
    private readonly amount: number,
    private readonly currency: string
  ) {
    if (amount < 0) throw new Error('Amount cannot be negative');
    if (currency.length !== 3) throw new Error('Currency must be a 3-letter ISO code');
  }

  add(other: Money): Money {
    this.assertSameCurrency(other);
    return new Money(this.amount + other.amount, this.currency);
  }

  subtract(other: Money): Money {
    this.assertSameCurrency(other);
    return new Money(this.amount - other.amount, this.currency);
  }

  multiply(factor: number): Money {
    return new Money(Math.round(this.amount * factor * 100) / 100, this.currency);
  }

  equals(other: Money): boolean {
    return this.amount === other.amount && this.currency === other.currency;
  }

  toString(): string { return `${this.currency} ${this.amount.toFixed(2)}`; }

  private assertSameCurrency(other: Money): void {
    if (this.currency !== other.currency) {
      throw new Error(`Currency mismatch: ${this.currency} vs ${other.currency}`);
    }
  }
}
```

### 7.2 Entities and Aggregates

```typescript
// domain/order.aggregate.ts

import { Money } from './value-objects';

type OrderStatus = 'pending' | 'confirmed' | 'shipped' | 'delivered' | 'cancelled';

interface OrderItem {
  productId: string;
  productName: string;
  quantity: number;
  unitPrice: Money;
}

interface DomainEvent {
  occurredAt: Date;
  aggregateId: string;
}

interface OrderPlaced extends DomainEvent { type: 'ORDER_PLACED'; customerId: string }
interface OrderConfirmed extends DomainEvent { type: 'ORDER_CONFIRMED' }
interface OrderCancelled extends DomainEvent { type: 'ORDER_CANCELLED'; reason: string }

type OrderEvent = OrderPlaced | OrderConfirmed | OrderCancelled;

// Aggregate root — enforces all invariants
class Order {
  private _status: OrderStatus = 'pending';
  private _items: OrderItem[] = [];
  private _events: OrderEvent[] = [];

  constructor(
    public readonly id: string,
    public readonly customerId: string,
    public readonly createdAt: Date = new Date()
  ) {
    this._events.push({ type: 'ORDER_PLACED', occurredAt: new Date(), aggregateId: id, customerId });
  }

  addItem(item: OrderItem): void {
    if (this._status !== 'pending') {
      throw new Error('Cannot modify a non-pending order');
    }
    const existing = this._items.find((i) => i.productId === item.productId);
    if (existing) {
      existing.quantity += item.quantity;
    } else {
      this._items.push({ ...item });
    }
  }

  confirm(): void {
    if (this._status !== 'pending') throw new Error('Only pending orders can be confirmed');
    if (this._items.length === 0) throw new Error('Cannot confirm an empty order');
    this._status = 'confirmed';
    this._events.push({ type: 'ORDER_CONFIRMED', occurredAt: new Date(), aggregateId: this.id });
  }

  cancel(reason: string): void {
    if (['delivered', 'cancelled'].includes(this._status)) {
      throw new Error(`Cannot cancel a ${this._status} order`);
    }
    this._status = 'cancelled';
    this._events.push({ type: 'ORDER_CANCELLED', occurredAt: new Date(), aggregateId: this.id, reason });
  }

  get total(): Money {
    return this._items.reduce(
      (sum, item) => sum.add(item.unitPrice.multiply(item.quantity)),
      new Money(0, 'USD')
    );
  }

  get status(): OrderStatus { return this._status; }
  get items(): ReadonlyArray<OrderItem> { return Object.freeze([...this._items]); }

  pullEvents(): OrderEvent[] {
    const events = [...this._events];
    this._events = [];
    return events;
  }
}
```

### 7.3 Domain Services and Specifications

```typescript
// domain/discount.specification.ts

interface Specification<T> {
  isSatisfiedBy(candidate: T): boolean;
  and(other: Specification<T>): Specification<T>;
  or(other: Specification<T>): Specification<T>;
  not(): Specification<T>;
}

abstract class BaseSpecification<T> implements Specification<T> {
  abstract isSatisfiedBy(candidate: T): boolean;

  and(other: Specification<T>): Specification<T> {
    return new AndSpecification(this, other);
  }
  or(other: Specification<T>): Specification<T> {
    return new OrSpecification(this, other);
  }
  not(): Specification<T> {
    return new NotSpecification(this);
  }
}

class AndSpecification<T> extends BaseSpecification<T> {
  constructor(private a: Specification<T>, private b: Specification<T>) { super(); }
  isSatisfiedBy(c: T) { return this.a.isSatisfiedBy(c) && this.b.isSatisfiedBy(c); }
}

class OrSpecification<T> extends BaseSpecification<T> {
  constructor(private a: Specification<T>, private b: Specification<T>) { super(); }
  isSatisfiedBy(c: T) { return this.a.isSatisfiedBy(c) || this.b.isSatisfiedBy(c); }
}

class NotSpecification<T> extends BaseSpecification<T> {
  constructor(private spec: Specification<T>) { super(); }
  isSatisfiedBy(c: T) { return !this.spec.isSatisfiedBy(c); }
}

// Domain-specific specifications
class PremiumCustomerSpec extends BaseSpecification<{ totalSpend: number }> {
  isSatisfiedBy(c: { totalSpend: number }) { return c.totalSpend >= 1000; }
}

class NewCustomerSpec extends BaseSpecification<{ accountAgeDays: number }> {
  isSatisfiedBy(c: { accountAgeDays: number }) { return c.accountAgeDays <= 30; }
}
```

---

## 8. Opaque Types (Branded Types)

Branded types prevent accidentally using a `UserId` where a `ProductId` is expected — even though both are strings underneath.

### 8.1 The Brand Pattern

```typescript
// types/branded.ts

// The brand is a phantom type — it exists only at compile time
declare const __brand: unique symbol;
type Brand<T, B> = T & { readonly [__brand]: B };

// Create branded types
type UserId    = Brand<string, 'UserId'>;
type ProductId = Brand<string, 'ProductId'>;
type OrderId   = Brand<string, 'OrderId'>;
type Email     = Brand<string, 'Email'>;
type PositiveInt = Brand<number, 'PositiveInt'>;

// Smart constructors — the ONLY way to create branded values
const UserId = {
  create: (id: string): UserId => {
    if (!id.trim()) throw new Error('UserId cannot be empty');
    return id as UserId;
  },
  parse: (id: unknown): UserId => {
    if (typeof id !== 'string') throw new Error('UserId must be a string');
    return UserId.create(id);
  },
};

const Email = {
  create: (email: string): Email => {
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) throw new Error(`Invalid email: ${email}`);
    return email.toLowerCase() as Email;
  },
};

const PositiveInt = {
  create: (n: number): PositiveInt => {
    if (!Number.isInteger(n) || n <= 0) throw new Error(`Expected positive integer, got ${n}`);
    return n as PositiveInt;
  },
};
```

### 8.2 Why This Matters

```typescript
// Without branded types — this compiles but is a bug:
function getOrder(userId: string, orderId: string): void {}
getOrder(orderId, userId); // ✅ no error — but wrong!

// With branded types — caught at compile time:
function getOrder(userId: UserId, orderId: OrderId): void {}

const uid = UserId.create('user-123');
const oid = 'order-456' as OrderId;

getOrder(uid, oid);  // ✅ correct
// getOrder(oid, uid);  // ❌ TypeScript error — types are incompatible
```

### 8.3 Combining Brands with Generics

```typescript
// types/id.ts

type Id<Entity extends string> = Brand<string, Entity>;

type UserId    = Id<'User'>;
type ProductId = Id<'Product'>;
type OrderId   = Id<'Order'>;

function createId<E extends string>(entity: E, value: string): Id<E> {
  if (!value.trim()) throw new Error(`${entity} ID cannot be empty`);
  return value as Id<E>;
}

const userId:    UserId    = createId('User', 'u-123');
const productId: ProductId = createId('Product', 'p-456');

// ❌ Compile error — UserId is not assignable to ProductId
// const wrong: ProductId = userId;
```

---

## 9. Migrating a JS Codebase to TypeScript

Migrating doesn't mean rewriting everything at once. A phased approach keeps your team productive and ships value continuously.

### Phase 1 — Configure TypeScript Permissively

```json
// tsconfig.json — initial migration config
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",

    // Migration flags
    "allowJs": true,           // Compile JS files alongside TS
    "checkJs": false,          // Don't type-check JS yet
    "noImplicitAny": false,    // Allow implicit any (for now)
    "strict": false,           // Relax strictness initially

    // Always on — even during migration
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### Phase 2 — Rename Files and Add Types Gradually

```bash
# Rename one module at a time
mv src/services/user.js src/services/user.ts

# Install type definitions
npm install --save-dev @types/node @types/express @types/lodash
```

```typescript
// Before (JavaScript)
function processUser(user) {
  return {
    ...user,
    fullName: `${user.firstName} ${user.lastName}`,
    age: calculateAge(user.birthDate),
  };
}

// After (TypeScript — Phase 2: explicit but not yet strict)
interface UserInput {
  id: string;
  firstName: string;
  lastName: string;
  birthDate: Date;
  [key: string]: any; // allow extra props during migration
}

interface ProcessedUser extends UserInput {
  fullName: string;
  age: number;
}

function processUser(user: UserInput): ProcessedUser {
  return {
    ...user,
    fullName: `${user.firstName} ${user.lastName}`,
    age: calculateAge(user.birthDate),
  };
}
```

### Phase 3 — Enable Strict Checks Incrementally

Enable flags one-by-one, fixing errors as you go:

```json
{
  "compilerOptions": {
    "checkJs": true,           // ← enable after all .ts conversions
    "noImplicitAny": true,     // ← fix implicit any usage
    "strictNullChecks": true,  // ← handle null/undefined
    "strictFunctionTypes": true,
    "noImplicitReturns": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
    // Eventually: "strict": true
  }
}
```

### Phase 4 — Handling Third-Party Code Without Types

```typescript
// For packages without @types — use declaration merging or ambient modules

// types/vendor.d.ts
declare module 'untyped-package' {
  export function doSomething(input: string): string;
  export const version: string;
}

// Quick escape hatch during migration (remove as you go)
declare module '*.json' {
  const value: any;
  export default value;
}
```

---

## 10. Gradual Adoption Strategy with `allowJs`

`allowJs` is the key that lets TypeScript and JavaScript coexist in the same project, the same build, and the same imports.

### 10.1 How `allowJs` Works

```
src/
├── auth/
│   ├── jwt.ts          ✅ Fully typed
│   └── middleware.js   ⏳ Still JavaScript
├── users/
│   ├── service.ts      ✅ Fully typed
│   └── helpers.js      ⏳ Still JavaScript
└── legacy/
    └── old-utils.js    ⏳ Migration queued
```

TypeScript resolves imports between files regardless of extension — a `.ts` file can import a `.js` file and vice versa.

### 10.2 JSDoc Annotations — Type Without Converting

Add types to `.js` files using JSDoc comments. TypeScript respects them when `checkJs: true`.

```javascript
// src/legacy/calculator.js

/**
 * @param {number} a
 * @param {number} b
 * @returns {number}
 */
function add(a, b) {
  return a + b;
}

/**
 * @typedef {{ id: string; amount: number; currency: string }} Invoice
 */

/**
 * @param {Invoice} invoice
 * @returns {string}
 */
function formatInvoice(invoice) {
  return `Invoice ${invoice.id}: ${invoice.currency}${invoice.amount}`;
}

module.exports = { add, formatInvoice };
```

### 10.3 Migration Prioritization Framework

Focus effort where the ROI is highest:

```
HIGH PRIORITY (migrate first)
├── Shared utilities used everywhere
├── Data models / domain objects
├── API boundaries (controllers, routes)
└── Code that changes frequently

MEDIUM PRIORITY
├── Service classes
├── Helper functions
└── Configuration modules

LOW PRIORITY (migrate last)
├── Simple glue code
├── One-off scripts
└── Stable code that never changes
```

### 10.4 Tracking Migration Progress

```typescript
// scripts/migration-status.ts

import * as fs from 'fs';
import * as path from 'path';
import * as glob from 'glob';

interface MigrationStats {
  total: number;
  typescript: number;
  javascript: number;
  percentage: number;
}

function getMigrationStats(srcDir: string): MigrationStats {
  const tsFiles = glob.sync(`${srcDir}/**/*.ts`, { ignore: ['**/*.d.ts'] });
  const jsFiles = glob.sync(`${srcDir}/**/*.js`);

  const total = tsFiles.length + jsFiles.length;
  const typescript = tsFiles.length;
  const javascript = jsFiles.length;

  return {
    total,
    typescript,
    javascript,
    percentage: total > 0 ? Math.round((typescript / total) * 100) : 0,
  };
}

const stats = getMigrationStats('./src');
console.log(`Migration Progress: ${stats.percentage}%`);
console.log(`✅ TypeScript: ${stats.typescript} files`);
console.log(`⏳ JavaScript: ${stats.javascript} files`);
console.log(`📁 Total: ${stats.total} files`);
```

### 10.5 The Complete Migration Roadmap

```
Week 1–2:   Set up tsconfig with allowJs, install @types
Week 3–4:   Convert shared utilities and domain types
Week 5–6:   Convert service and repository layers
Week 7–8:   Convert API controllers and routes
Week 9–10:  Enable strict: true, fix remaining errors
Week 11–12: Remove allowJs (or keep for legacy code)
Ongoing:    Code review enforces TypeScript for all new files
```

---

## Summary

| Topic | Key Takeaway |
|---|---|
| **Repository Pattern** | Abstract data access behind a typed generic interface |
| **Service Layer** | DTOs + typed service interfaces decouple HTTP from business logic |
| **Result Pattern** | Make errors part of your return type, not a surprise exception |
| **Builder Pattern** | Enforce construction order at compile time with type-state |
| **Factory Overloads** | One function, multiple specific return types |
| **Dependency Injection** | Inject interfaces, not implementations; wire at the composition root |
| **DDD Types** | Value objects, entities, aggregates model your domain in the type system |
| **Branded Types** | Primitive obsession killer — prevent mixing `UserId` with `ProductId` |
| **JS Migration** | Phased, file-by-file; convert highest-impact modules first |
| **allowJs Strategy** | TypeScript and JavaScript coexist; JSDoc bridges the gap |

---

> **Next Steps:** Week 13 covers advanced TypeScript compiler APIs, custom transformers, and building your own type-checking tooling.
