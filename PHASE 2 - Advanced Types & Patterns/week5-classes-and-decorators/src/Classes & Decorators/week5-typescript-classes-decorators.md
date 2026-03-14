# Week 5: TypeScript Classes & Decorators

> **Prerequisites:** Weeks 1–4 (Types, Interfaces, Generics, Utility Types)  
> **Estimated Time:** 4–6 hours  
> **Difficulty:** Intermediate → Advanced

---

## Table of Contents

1. [Classes with Full Type Annotations](#1-classes-with-full-type-annotations)
2. [Access Modifiers: public, private, protected](#2-access-modifiers)
3. [Readonly Class Properties](#3-readonly-class-properties)
4. [Abstract Classes and Methods](#4-abstract-classes-and-methods)
5. [Implementing Interfaces](#5-implementing-interfaces)
6. [Class Generics](#6-class-generics)
7. [Static Members](#7-static-members)
8. [Decorators Introduction (@)](#8-decorators-introduction)
9. [Parameter Decorators](#9-parameter-decorators)
10. [Declaration Merging](#10-declaration-merging)
11. [Putting It All Together](#11-putting-it-all-together)
12. [Exercises](#12-exercises)

---

## 1. Classes with Full Type Annotations

TypeScript classes extend JavaScript classes with full static typing — every property, parameter, and return value can (and should) be explicitly typed.

### Basic Typed Class

```typescript
class User {
  name: string;
  age: number;
  email: string;

  constructor(name: string, age: number, email: string) {
    this.name = name;
    this.age = age;
    this.email = email;
  }

  greet(): string {
    return `Hello, I'm ${this.name} and I'm ${this.age} years old.`;
  }

  isAdult(): boolean {
    return this.age >= 18;
  }
}

const user = new User("Alice", 30, "alice@example.com");
console.log(user.greet()); // "Hello, I'm Alice and I'm 30 years old."
```

### Shorthand Constructor Parameters

TypeScript lets you declare and initialize properties directly in the constructor signature — this is one of the most common TypeScript patterns:

```typescript
class Product {
  constructor(
    public name: string,
    public price: number,
    private sku: string,
    readonly createdAt: Date = new Date()
  ) {}

  getSku(): string {
    return this.sku;
  }

  toString(): string {
    return `${this.name} — $${this.price}`;
  }
}

const product = new Product("Laptop", 999.99, "LAP-001");
console.log(product.name);       // "Laptop"
console.log(product.toString()); // "Laptop — $999.99"
// console.log(product.sku);     // ❌ Error: 'sku' is private
```

> **Key Insight:** Adding `public`, `private`, `protected`, or `readonly` before a constructor parameter automatically creates and assigns the class property. Without an access modifier, it's just a local parameter.

### Typed Methods and Return Types

```typescript
class Calculator {
  private history: string[] = [];

  add(a: number, b: number): number {
    const result = a + b;
    this.history.push(`${a} + ${b} = ${result}`);
    return result;
  }

  subtract(a: number, b: number): number {
    const result = a - b;
    this.history.push(`${a} - ${b} = ${result}`);
    return result;
  }

  getHistory(): readonly string[] {
    return this.history;
  }

  clearHistory(): void {
    this.history = [];
  }
}

const calc = new Calculator();
calc.add(5, 3);        // 8
calc.subtract(10, 4);  // 6
console.log(calc.getHistory());
// ["5 + 3 = 8", "10 - 4 = 6"]
```

---

## 2. Access Modifiers

TypeScript provides three access modifiers that control the visibility of class members. These are enforced **at compile time** only — they don't exist at runtime in JavaScript.

| Modifier    | Same Class | Subclass | Outside Class |
|-------------|:----------:|:--------:|:-------------:|
| `public`    | ✅          | ✅        | ✅             |
| `protected` | ✅          | ✅        | ❌             |
| `private`   | ✅          | ❌        | ❌             |

### `public` — Default Visibility

All class members are `public` by default. Explicit `public` is used for clarity:

```typescript
class Circle {
  public radius: number; // Same as just: radius: number

  constructor(radius: number) {
    this.radius = radius;
  }

  public getArea(): number {
    return Math.PI * this.radius ** 2;
  }
}

const c = new Circle(5);
console.log(c.radius);     // ✅ 5
console.log(c.getArea());  // ✅ 78.54
```

### `private` — Class-Only Access

`private` members are accessible only within the declaring class:

```typescript
class BankAccount {
  private balance: number;
  private transactionLog: string[] = [];

  constructor(initialBalance: number) {
    this.balance = initialBalance;
  }

  deposit(amount: number): void {
    if (amount <= 0) throw new Error("Deposit must be positive.");
    this.balance += amount;
    this.logTransaction(`Deposit: +$${amount}`);
  }

  withdraw(amount: number): void {
    if (amount > this.balance) throw new Error("Insufficient funds.");
    this.balance -= amount;
    this.logTransaction(`Withdrawal: -$${amount}`);
  }

  getBalance(): number {
    return this.balance; // ✅ Accessible inside the class
  }

  private logTransaction(entry: string): void {
    this.transactionLog.push(`[${new Date().toISOString()}] ${entry}`);
  }
}

const account = new BankAccount(1000);
account.deposit(500);
console.log(account.getBalance()); // ✅ 1500
// account.balance = 9999;         // ❌ Error: 'balance' is private
// account.logTransaction("cheat"); // ❌ Error: 'logTransaction' is private
```

### `protected` — Class and Subclass Access

`protected` is like `private` but accessible in subclasses:

```typescript
class Animal {
  protected name: string;
  protected sound: string;

  constructor(name: string, sound: string) {
    this.name = name;
    this.sound = sound;
  }

  protected makeSound(): string {
    return `${this.name} says: ${this.sound}`;
  }
}

class Dog extends Animal {
  private breed: string;

  constructor(name: string, breed: string) {
    super(name, "Woof"); // ✅ Calling parent constructor
    this.breed = breed;
  }

  introduce(): string {
    // ✅ Can access protected members from parent
    return `${this.makeSound()} I'm a ${this.breed}.`;
  }
}

const dog = new Dog("Rex", "Labrador");
console.log(dog.introduce()); // "Rex says: Woof I'm a Labrador."
// dog.name;                  // ❌ Error: 'name' is protected
// dog.makeSound();           // ❌ Error: 'makeSound' is protected
```

### ECMAScript Private Fields (`#`)

TypeScript also supports the native JavaScript private field syntax using `#`. Unlike `private`, these are enforced **at runtime**:

```typescript
class SecureVault {
  #pin: number; // True private — enforced at runtime
  #contents: string[] = [];

  constructor(pin: number) {
    this.#pin = pin;
  }

  unlock(pin: number, item: string): boolean {
    if (pin === this.#pin) {
      this.#contents.push(item);
      return true;
    }
    return false;
  }
}

const vault = new SecureVault(1234);
vault.unlock(1234, "Gold"); // true
// vault.#pin;              // ❌ Error at compile AND runtime
```

> **`private` vs `#`:** Use `private` for TypeScript-only projects. Use `#` when you need true runtime enforcement or are targeting modern JS environments.

---

## 3. Readonly Class Properties

`readonly` properties can only be assigned during declaration or inside the constructor. After that, they're immutable.

```typescript
class Configuration {
  readonly appName: string;
  readonly version: string;
  readonly maxConnections: number;
  readonly createdAt: Date;

  constructor(appName: string, version: string, maxConnections: number = 100) {
    this.appName = appName;
    this.version = version;
    this.maxConnections = maxConnections;
    this.createdAt = new Date();
  }

  describe(): string {
    return `${this.appName} v${this.version} (max: ${this.maxConnections} connections)`;
  }
}

const config = new Configuration("MyApp", "2.1.0");
console.log(config.describe());  // "MyApp v2.1.0 (max: 100 connections)"
// config.appName = "HackedApp"; // ❌ Error: Cannot assign to 'appName' — it's readonly
```

### `readonly` with Arrays and Objects

Note that `readonly` prevents reassignment of the reference, but doesn't deep-freeze the contents:

```typescript
class Team {
  readonly name: string;
  readonly members: string[]; // The array reference is readonly, not its contents

  constructor(name: string, initialMembers: string[] = []) {
    this.name = name;
    this.members = initialMembers;
  }

  addMember(member: string): void {
    this.members.push(member); // ✅ Mutating the array is still allowed
    // this.members = [];      // ❌ Error: Can't reassign the array reference
  }
}

// For truly immutable arrays, use ReadonlyArray<T> or readonly T[]
class ImmutableTeam {
  readonly members: ReadonlyArray<string>;

  constructor(members: string[]) {
    this.members = members;
  }
  // Now members.push() is also disallowed ✅
}
```

---

## 4. Abstract Classes and Methods

Abstract classes serve as blueprints — they **cannot be instantiated directly** but can define a common structure that subclasses must implement.

```typescript
abstract class Shape {
  // Abstract method: no implementation, subclasses MUST implement
  abstract getArea(): number;
  abstract getPerimeter(): number;
  abstract getName(): string;

  // Concrete method: shared implementation all subclasses inherit
  describe(): string {
    return (
      `${this.getName()}: ` +
      `Area = ${this.getArea().toFixed(2)}, ` +
      `Perimeter = ${this.getPerimeter().toFixed(2)}`
    );
  }

  isLargerThan(other: Shape): boolean {
    return this.getArea() > other.getArea();
  }
}

// Concrete subclass — must implement all abstract methods
class Rectangle extends Shape {
  constructor(
    private width: number,
    private height: number
  ) {
    super();
  }

  getName(): string { return "Rectangle"; }
  getArea(): number { return this.width * this.height; }
  getPerimeter(): number { return 2 * (this.width + this.height); }
}

class Circle extends Shape {
  constructor(private radius: number) {
    super();
  }

  getName(): string { return "Circle"; }
  getArea(): number { return Math.PI * this.radius ** 2; }
  getPerimeter(): number { return 2 * Math.PI * this.radius; }
}

// const shape = new Shape(); // ❌ Error: Cannot create instance of abstract class

const rect = new Rectangle(5, 8);
const circ = new Circle(4);

console.log(rect.describe()); // "Rectangle: Area = 40.00, Perimeter = 26.00"
console.log(circ.describe()); // "Circle: Area = 50.27, Perimeter = 25.13"
console.log(rect.isLargerThan(circ)); // false (40 < 50.27)
```

### Abstract Classes with Abstract Properties

```typescript
abstract class Vehicle {
  abstract readonly brand: string;
  abstract readonly fuelType: "petrol" | "diesel" | "electric" | "hybrid";

  abstract startEngine(): string;

  // Concrete shared behavior
  describe(): string {
    return `${this.brand} (${this.fuelType})`;
  }
}

class Tesla extends Vehicle {
  readonly brand = "Tesla";
  readonly fuelType = "electric" as const;

  startEngine(): string {
    return "⚡ Silently humming...";
  }
}

class Ferrari extends Vehicle {
  readonly brand = "Ferrari";
  readonly fuelType = "petrol" as const;

  startEngine(): string {
    return "🔥 VROOOOM!";
  }
}

const cars: Vehicle[] = [new Tesla(), new Ferrari()];
cars.forEach(car => console.log(`${car.describe()} → ${car.startEngine()}`));
```

> **When to use abstract classes vs interfaces?**  
> Use **abstract classes** when you want to share concrete implementation logic alongside contracts.  
> Use **interfaces** when you only want to define a contract with no implementation.

---

## 5. Implementing Interfaces

A class can implement one or more interfaces, committing to satisfy all their contracts.

### Single Interface

```typescript
interface Serializable {
  serialize(): string;
  deserialize(data: string): void;
}

class UserProfile implements Serializable {
  constructor(
    public username: string,
    public email: string,
    public age: number
  ) {}

  serialize(): string {
    return JSON.stringify({
      username: this.username,
      email: this.email,
      age: this.age,
    });
  }

  deserialize(data: string): void {
    const parsed = JSON.parse(data);
    this.username = parsed.username;
    this.email = parsed.email;
    this.age = parsed.age;
  }
}

const profile = new UserProfile("alice", "alice@example.com", 28);
const serialized = profile.serialize();
console.log(serialized);
// {"username":"alice","email":"alice@example.com","age":28}
```

### Implementing Multiple Interfaces

```typescript
interface Printable {
  print(): void;
}

interface Loggable {
  log(message: string): void;
  getLogs(): string[];
}

interface Timestamped {
  createdAt: Date;
  updatedAt: Date;
}

// A class can implement multiple interfaces
class Document implements Printable, Loggable, Timestamped {
  createdAt: Date = new Date();
  updatedAt: Date = new Date();
  private logs: string[] = [];

  constructor(
    private title: string,
    private content: string
  ) {}

  print(): void {
    console.log(`=== ${this.title} ===\n${this.content}`);
  }

  log(message: string): void {
    this.logs.push(`[${new Date().toISOString()}] ${message}`);
    this.updatedAt = new Date();
  }

  getLogs(): string[] {
    return [...this.logs];
  }
}

const doc = new Document("Report", "Q3 earnings look great!");
doc.print();
doc.log("Document reviewed by manager");
doc.log("Approved for distribution");
console.log(doc.getLogs());
```

### Class Extending Another Class While Implementing Interfaces

```typescript
interface Flyable {
  fly(): string;
  maxAltitude: number;
}

interface Swimmable {
  swim(): string;
  maxDepth: number;
}

abstract class Bird {
  abstract makeSound(): string;

  breathe(): string {
    return "Breathing air...";
  }
}

// Extends a class AND implements interfaces
class Duck extends Bird implements Flyable, Swimmable {
  maxAltitude = 500;  // meters
  maxDepth = 1;       // meters

  makeSound(): string { return "Quack!"; }
  fly(): string { return "Flapping wings, soaring low"; }
  swim(): string { return "Paddling across the pond"; }
}

const duck = new Duck();
console.log(duck.makeSound()); // "Quack!"
console.log(duck.fly());       // "Flapping wings, soaring low"
console.log(duck.swim());      // "Paddling across the pond"
```

---

## 6. Class Generics

Generic classes let you create reusable, type-safe data structures and utilities.

### Basic Generic Class

```typescript
class Box<T> {
  private value: T;

  constructor(value: T) {
    this.value = value;
  }

  getValue(): T {
    return this.value;
  }

  setValue(newValue: T): void {
    this.value = newValue;
  }

  transform<U>(fn: (value: T) => U): Box<U> {
    return new Box(fn(this.value));
  }
}

const numberBox = new Box<number>(42);
console.log(numberBox.getValue()); // 42

const stringBox = new Box<string>("TypeScript");
console.log(stringBox.getValue()); // "TypeScript"

// Transform: Box<number> → Box<string>
const strBox = numberBox.transform(n => `Value is ${n}`);
console.log(strBox.getValue()); // "Value is 42"
```

### Generic Repository Pattern

```typescript
interface Entity {
  id: number;
}

class Repository<T extends Entity> {
  private items: T[] = [];
  private nextId = 1;

  create(item: Omit<T, "id">): T {
    const newItem = { ...item, id: this.nextId++ } as T;
    this.items.push(newItem);
    return newItem;
  }

  findById(id: number): T | undefined {
    return this.items.find(item => item.id === id);
  }

  findAll(): T[] {
    return [...this.items];
  }

  update(id: number, updates: Partial<Omit<T, "id">>): T | undefined {
    const index = this.items.findIndex(item => item.id === id);
    if (index === -1) return undefined;
    this.items[index] = { ...this.items[index], ...updates };
    return this.items[index];
  }

  delete(id: number): boolean {
    const index = this.items.findIndex(item => item.id === id);
    if (index === -1) return false;
    this.items.splice(index, 1);
    return true;
  }

  count(): number {
    return this.items.length;
  }
}

// Strongly typed for each entity
interface User extends Entity {
  name: string;
  email: string;
}

interface Post extends Entity {
  title: string;
  body: string;
  authorId: number;
}

const userRepo = new Repository<User>();
const postRepo = new Repository<Post>();

const alice = userRepo.create({ name: "Alice", email: "alice@example.com" });
const bob = userRepo.create({ name: "Bob", email: "bob@example.com" });

const post = postRepo.create({ title: "Hello World", body: "First post!", authorId: alice.id });

console.log(userRepo.findAll()); // [{id:1, name:"Alice",...}, {id:2, name:"Bob",...}]
console.log(postRepo.count());   // 1
```

### Generic Class with Multiple Type Parameters

```typescript
class Pair<K, V> {
  constructor(
    public readonly key: K,
    public readonly value: V
  ) {}

  swap(): Pair<V, K> {
    return new Pair(this.value, this.key);
  }

  toString(): string {
    return `(${this.key}, ${this.value})`;
  }
}

const pair = new Pair<string, number>("age", 30);
console.log(pair.toString());         // "(age, 30)"
console.log(pair.swap().toString());  // "(30, age)"
```

### Generic Class with Constraints

```typescript
interface Comparable<T> {
  compareTo(other: T): number; // negative, 0, or positive
}

class SortedList<T extends Comparable<T>> {
  private items: T[] = [];

  insert(item: T): void {
    let i = 0;
    while (i < this.items.length && this.items[i].compareTo(item) < 0) {
      i++;
    }
    this.items.splice(i, 0, item);
  }

  toArray(): T[] {
    return [...this.items];
  }
}

class Temperature implements Comparable<Temperature> {
  constructor(public readonly celsius: number) {}

  compareTo(other: Temperature): number {
    return this.celsius - other.celsius;
  }

  toString(): string {
    return `${this.celsius}°C`;
  }
}

const temps = new SortedList<Temperature>();
temps.insert(new Temperature(30));
temps.insert(new Temperature(15));
temps.insert(new Temperature(22));
console.log(temps.toArray().map(t => t.toString())); 
// ["15°C", "22°C", "30°C"]
```

---

## 7. Static Members

Static members belong to the **class itself**, not to any instance. They're shared across all instances and accessed via the class name.

### Static Properties and Methods

```typescript
class Counter {
  private static count: number = 0;
  private readonly id: number;

  constructor() {
    Counter.count++;
    this.id = Counter.count;
  }

  getId(): number {
    return this.id;
  }

  static getCount(): number {
    return Counter.count;
  }

  static reset(): void {
    Counter.count = 0;
  }
}

const c1 = new Counter();
const c2 = new Counter();
const c3 = new Counter();

console.log(Counter.getCount()); // 3
console.log(c1.getId());         // 1
console.log(c3.getId());         // 3

Counter.reset();
console.log(Counter.getCount()); // 0
```

### Static Factory Methods

Static methods are ideal for named constructors and factory patterns:

```typescript
class Color {
  private constructor(
    public readonly r: number,
    public readonly g: number,
    public readonly b: number
  ) {}

  // Factory methods — more descriptive than constructors
  static fromRGB(r: number, g: number, b: number): Color {
    if ([r, g, b].some(v => v < 0 || v > 255)) {
      throw new Error("RGB values must be between 0 and 255.");
    }
    return new Color(r, g, b);
  }

  static fromHex(hex: string): Color {
    const clean = hex.replace("#", "");
    const r = parseInt(clean.substring(0, 2), 16);
    const g = parseInt(clean.substring(2, 4), 16);
    const b = parseInt(clean.substring(4, 6), 16);
    return new Color(r, g, b);
  }

  // Named color constants
  static readonly RED = new Color(255, 0, 0);
  static readonly GREEN = new Color(0, 255, 0);
  static readonly BLUE = new Color(0, 0, 255);
  static readonly WHITE = new Color(255, 255, 255);
  static readonly BLACK = new Color(0, 0, 0);

  toHex(): string {
    return `#${[this.r, this.g, this.b]
      .map(v => v.toString(16).padStart(2, "0"))
      .join("")}`;
  }

  toString(): string {
    return `rgb(${this.r}, ${this.g}, ${this.b})`;
  }
}

// Constructor is private — must use factory methods
const red = Color.fromRGB(255, 0, 0);
const coral = Color.fromHex("#FF7F50");

console.log(Color.RED.toHex());    // "#ff0000"
console.log(coral.toString());     // "rgb(255, 127, 80)"
// new Color(0, 0, 0);             // ❌ Error: Constructor is private
```

### Static with Inheritance

```typescript
class MathUtils {
  static readonly PI = 3.14159265358979;

  static circleArea(radius: number): number {
    return MathUtils.PI * radius ** 2;
  }

  static clamp(value: number, min: number, max: number): number {
    return Math.min(Math.max(value, min), max);
  }

  static lerp(a: number, b: number, t: number): number {
    return a + (b - a) * t;
  }
}

console.log(MathUtils.circleArea(5));      // 78.539...
console.log(MathUtils.clamp(150, 0, 100)); // 100
console.log(MathUtils.lerp(0, 10, 0.5));   // 5
```

---

## 8. Decorators Introduction

Decorators are a **stage 3 TC39 proposal** that TypeScript has supported for years. They're functions that can modify classes, methods, properties, and parameters at definition time.

### Enabling Decorators

In your `tsconfig.json`:

```json
{
  "compilerOptions": {
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  }
}
```

### Class Decorators

A class decorator is a function that receives the class constructor and can modify or replace it:

```typescript
// Decorator factory — returns a decorator function
function Sealed(constructor: Function): void {
  Object.seal(constructor);
  Object.seal(constructor.prototype);
}

function AddTimestamp<T extends { new(...args: any[]): {} }>(constructor: T) {
  return class extends constructor {
    createdAt = new Date();
    updatedAt = new Date();
  };
}

@Sealed
@AddTimestamp
class Report {
  constructor(public title: string) {}
}

const report = new Report("Annual Summary");
console.log(report.title);     // "Annual Summary"
console.log((report as any).createdAt); // Date object added by decorator
```

### Method Decorators

Method decorators receive the target object, property name, and property descriptor:

```typescript
function Log(
  target: any,
  propertyKey: string,
  descriptor: PropertyDescriptor
): PropertyDescriptor {
  const originalMethod = descriptor.value;

  descriptor.value = function (...args: any[]) {
    console.log(`▶ Calling ${propertyKey}(${args.join(", ")})`);
    const result = originalMethod.apply(this, args);
    console.log(`◀ ${propertyKey} returned: ${result}`);
    return result;
  };

  return descriptor;
}

function Memoize(
  target: any,
  propertyKey: string,
  descriptor: PropertyDescriptor
): PropertyDescriptor {
  const originalMethod = descriptor.value;
  const cache = new Map<string, any>();

  descriptor.value = function (...args: any[]) {
    const key = JSON.stringify(args);
    if (cache.has(key)) {
      console.log(`⚡ Cache hit for ${propertyKey}(${key})`);
      return cache.get(key);
    }
    const result = originalMethod.apply(this, args);
    cache.set(key, result);
    return result;
  };

  return descriptor;
}

class MathService {
  @Log
  add(a: number, b: number): number {
    return a + b;
  }

  @Memoize
  fibonacci(n: number): number {
    if (n <= 1) return n;
    return this.fibonacci(n - 1) + this.fibonacci(n - 2);
  }
}

const service = new MathService();
service.add(3, 4);
// ▶ Calling add(3, 4)
// ◀ add returned: 7

service.fibonacci(10); // Computed
service.fibonacci(10); // ⚡ Cache hit
```

### Property Decorators

Property decorators receive the target and property name:

```typescript
function Validate(min: number, max: number) {
  return function (target: any, propertyKey: string): void {
    let value: number;

    const getter = function (): number {
      return value;
    };

    const setter = function (newValue: number): void {
      if (newValue < min || newValue > max) {
        throw new Error(
          `${propertyKey} must be between ${min} and ${max}. Got: ${newValue}`
        );
      }
      value = newValue;
    };

    Object.defineProperty(target, propertyKey, {
      get: getter,
      set: setter,
      enumerable: true,
      configurable: true,
    });
  };
}

class Person {
  name: string;

  @Validate(0, 150)
  age: number;

  constructor(name: string, age: number) {
    this.name = name;
    this.age = age;
  }
}

const person = new Person("Alice", 30); // ✅
console.log(person.age); // 30

try {
  person.age = 200; // ❌ Throws: age must be between 0 and 150. Got: 200
} catch (e) {
  console.error(e.message);
}
```

### Accessor Decorators

Decorators can also target getters and setters:

```typescript
function ReadOnly(
  target: any,
  propertyKey: string,
  descriptor: PropertyDescriptor
): PropertyDescriptor {
  descriptor.set = undefined;
  return descriptor;
}

class Circle {
  constructor(private _radius: number) {}

  @ReadOnly
  get diameter(): number {
    return this._radius * 2;
  }
}

const circle = new Circle(5);
console.log(circle.diameter); // 10
// circle.diameter = 20;      // ❌ Error: no setter (read-only)
```

### Decorator Execution Order

When multiple decorators are applied, they execute **bottom-up**:

```typescript
function First() {
  return function (target: any, key: string, desc: PropertyDescriptor) {
    console.log("First decorator applied");
  };
}

function Second() {
  return function (target: any, key: string, desc: PropertyDescriptor) {
    console.log("Second decorator applied");
  };
}

class Demo {
  @First()   // Applied second (outer)
  @Second()  // Applied first (inner)
  doSomething() {}
}

// Output:
// "Second decorator applied"
// "First decorator applied"
```

---

## 9. Parameter Decorators

Parameter decorators receive the target, method name, and parameter index. They're commonly used with metadata reflection (e.g., in frameworks like NestJS and Angular).

```typescript
import "reflect-metadata"; // npm install reflect-metadata

const REQUIRED_METADATA_KEY = Symbol("required");

// Mark a parameter as required
function Required(
  target: Object,
  propertyKey: string | symbol,
  parameterIndex: number
): void {
  const existingRequired: number[] =
    Reflect.getOwnMetadata(REQUIRED_METADATA_KEY, target, propertyKey) || [];

  existingRequired.push(parameterIndex);
  Reflect.defineMetadata(REQUIRED_METADATA_KEY, existingRequired, target, propertyKey);
}

// Method decorator to validate required parameters
function Validate(
  target: any,
  propertyKey: string,
  descriptor: PropertyDescriptor
): PropertyDescriptor {
  const originalMethod = descriptor.value;

  descriptor.value = function (...args: any[]) {
    const requiredParams: number[] =
      Reflect.getOwnMetadata(REQUIRED_METADATA_KEY, target, propertyKey) || [];

    requiredParams.forEach(paramIndex => {
      if (args[paramIndex] === undefined || args[paramIndex] === null) {
        throw new Error(
          `Parameter at index ${paramIndex} of "${propertyKey}" is required.`
        );
      }
    });

    return originalMethod.apply(this, args);
  };

  return descriptor;
}

class UserService {
  @Validate
  createUser(
    @Required username: string,
    @Required email: string,
    role: string = "user"
  ): string {
    return `Created user: ${username} (${email}) with role: ${role}`;
  }
}

const service = new UserService();
console.log(service.createUser("alice", "alice@example.com"));
// "Created user: alice (alice@example.com) with role: user"

try {
  service.createUser("bob", null!); // ❌ Throws: Parameter at index 1 is required.
} catch (e) {
  console.error(e.message);
}
```

### Simpler Parameter Decorator Example (No Reflect)

```typescript
function LogParam(
  target: Object,
  methodName: string,
  paramIndex: number
): void {
  console.log(
    `Decorating parameter ${paramIndex} of method "${methodName}" on ${target.constructor.name}`
  );
}

class OrderService {
  placeOrder(
    @LogParam customerId: string,
    @LogParam items: string[]
  ): void {
    console.log(`Order placed for ${customerId}`);
  }
}

// At class definition time, outputs:
// "Decorating parameter 1 of method "placeOrder" on OrderService"
// "Decorating parameter 0 of method "placeOrder" on OrderService"
```

> **Note:** Parameter decorators are evaluated from **right to left** (last parameter first).

---

## 10. Declaration Merging

Declaration merging is a TypeScript-only feature where multiple declarations with the same name are combined into a single definition.

### Interface Merging

The most common form — multiple `interface` declarations with the same name are merged:

```typescript
interface User {
  id: number;
  name: string;
}

// Later in the code or another file — merges with the first
interface User {
  email: string;
  createdAt: Date;
}

// Result: User has ALL four properties
const user: User = {
  id: 1,
  name: "Alice",
  email: "alice@example.com",
  createdAt: new Date(),
};
```

### Extending Third-Party Types

This is declaration merging's most powerful real-world use — augmenting external library types:

```typescript
// Express Request augmentation (common real-world pattern)
declare namespace Express {
  interface Request {
    user?: {
      id: string;
      roles: string[];
    };
    requestId?: string;
  }
}

// Now in your Express handlers:
// app.get("/profile", (req: Express.Request, res) => {
//   console.log(req.user?.id); // ✅ TypeScript knows about req.user
// });
```

### Namespace Merging

Namespaces can be merged with each other and with classes:

```typescript
// Merging a class with a namespace to add static-like members
class Validator {
  constructor(private value: string) {}

  isValid(): boolean {
    return this.value.length > 0;
  }
}

// Merge a namespace into the Validator class
namespace Validator {
  export function isEmail(value: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value);
  }

  export function isPhone(value: string): boolean {
    return /^\+?[\d\s\-()]{7,}$/.test(value);
  }
}

const v = new Validator("test@email.com");
console.log(v.isValid());                        // true (instance method)
console.log(Validator.isEmail("test@email.com")); // true (namespace function)
console.log(Validator.isPhone("+1 555-1234"));    // true
```

### Enum and Namespace Merging

You can add functionality to an enum by merging it with a namespace:

```typescript
enum Direction {
  Up = "UP",
  Down = "DOWN",
  Left = "LEFT",
  Right = "RIGHT",
}

namespace Direction {
  export function opposite(dir: Direction): Direction {
    const map: Record<Direction, Direction> = {
      [Direction.Up]: Direction.Down,
      [Direction.Down]: Direction.Up,
      [Direction.Left]: Direction.Right,
      [Direction.Right]: Direction.Left,
    };
    return map[dir];
  }

  export function isVertical(dir: Direction): boolean {
    return dir === Direction.Up || dir === Direction.Down;
  }
}

console.log(Direction.opposite(Direction.Up));       // "DOWN"
console.log(Direction.isVertical(Direction.Left));   // false
```

### Module Augmentation

Augmenting third-party module types (different from global namespace merging):

```typescript
// Augment an existing module's exported interface
import "some-library";

declare module "some-library" {
  interface LibraryConfig {
    customField: string; // Add a new field to an existing interface
  }
}
```

---

## 11. Putting It All Together

Here's a comprehensive example combining all of this week's concepts:

```typescript
// ============================================================
// A fully-featured generic event system with decorators
// ============================================================

// --- Interfaces ---
interface EventPayload {
  timestamp: Date;
  source: string;
}

interface Subscribable<T extends EventPayload> {
  subscribe(handler: (event: T) => void): () => void;
  unsubscribeAll(): void;
}

// --- Decorators ---
function LogEvent(
  target: any,
  key: string,
  descriptor: PropertyDescriptor
): PropertyDescriptor {
  const original = descriptor.value;
  descriptor.value = function (...args: any[]) {
    console.log(`[EventBus] ${key} called`);
    return original.apply(this, args);
  };
  return descriptor;
}

function Singleton<T extends { new(...args: any[]): {} }>(Base: T) {
  let instance: InstanceType<T>;
  return class extends Base {
    constructor(...args: any[]) {
      if (instance) return instance;
      super(...args);
      instance = this as InstanceType<T>;
    }
  };
}

// --- Abstract Base Class ---
abstract class BaseEventBus<T extends EventPayload>
  implements Subscribable<T> {
  protected readonly handlers: Array<(event: T) => void> = [];
  private static totalEvents: number = 0;

  abstract validate(event: T): boolean;

  subscribe(handler: (event: T) => void): () => void {
    this.handlers.push(handler);
    return () => {
      const idx = this.handlers.indexOf(handler);
      if (idx > -1) this.handlers.splice(idx, 1);
    };
  }

  @LogEvent
  publish(event: T): void {
    if (!this.validate(event)) {
      throw new Error("Invalid event payload");
    }
    BaseEventBus.totalEvents++;
    this.handlers.forEach(h => h(event));
  }

  unsubscribeAll(): void {
    this.handlers.length = 0;
  }

  static getTotalEvents(): number {
    return BaseEventBus.totalEvents;
  }
}

// --- Concrete Generic Implementation ---
interface UserEvent extends EventPayload {
  userId: string;
  action: "login" | "logout" | "update";
}

@Singleton
class UserEventBus extends BaseEventBus<UserEvent> {
  readonly busName = "UserEventBus";

  validate(event: UserEvent): boolean {
    return Boolean(event.userId && event.action && event.timestamp);
  }
}

// --- Declaration Merging: Extend interface post-definition ---
interface UserEvent {
  metadata?: Record<string, unknown>;
}

// --- Usage ---
const bus1 = new UserEventBus();
const bus2 = new UserEventBus(); // Same instance (Singleton)
console.log(bus1 === bus2); // true

const unsub = bus1.subscribe(event => {
  console.log(`User ${event.userId} performed: ${event.action}`);
  if (event.metadata) {
    console.log("  Metadata:", event.metadata);
  }
});

bus1.publish({
  userId: "user-123",
  action: "login",
  timestamp: new Date(),
  source: "auth-service",
  metadata: { ip: "192.168.1.1", browser: "Chrome" },
});
// [EventBus] publish called
// User user-123 performed: login
//   Metadata: { ip: '192.168.1.1', browser: 'Chrome' }

unsub(); // Unsubscribe

bus1.publish({
  userId: "user-456",
  action: "logout",
  timestamp: new Date(),
  source: "auth-service",
});
// No handler fired (unsubscribed)

console.log(`Total events published: ${BaseEventBus.getTotalEvents()}`); // 2
```

---

## 12. Exercises

### Exercise 1 — Shapes Library
Create an abstract `Shape` class with abstract methods `area()`, `perimeter()`, and `toString()`. Implement `Triangle`, `Rectangle`, and `Circle` classes. Add a static `compare(a: Shape, b: Shape)` method that returns the larger shape by area.

### Exercise 2 — Generic Stack
Build a `Stack<T>` class with `push`, `pop`, `peek`, `isEmpty`, and `size` operations. Add a `toArray()` method and a static `from<T>(array: T[])` factory method. Ensure all operations are type-safe.

### Exercise 3 — Access Modifier Practice
Design a `Library` class with:
- `private` collection of books
- `protected` method `findByISBN`
- `public` methods for borrowing and returning books
- `readonly` name and `static` libraryCount

### Exercise 4 — Decorator Suite
Create a decorator toolkit with:
- `@Deprecated(message)` — logs a warning when a method is called
- `@Retry(times)` — retries a failed async method N times
- `@Throttle(ms)` — prevents a method from being called more than once in N ms

### Exercise 5 — Declaration Merging
Create a `Config` interface, then augment it in two separate "modules" (just separate `interface Config` blocks) to add new properties. Create a class that implements the fully merged interface.

---

## Quick Reference Cheatsheet

```typescript
// Access modifiers
class Example {
  public x: number = 0;    // accessible anywhere
  protected y: number = 0; // accessible in class + subclasses
  private z: number = 0;   // accessible only in this class
  #w: number = 0;           // true private (JS native)
  readonly r: number = 0;  // immutable after construction
  static s: number = 0;    // belongs to class, not instance
}

// Abstract class
abstract class Base {
  abstract mustImplement(): void;  // subclasses must provide
  concrete() { /* shared logic */ }
}

// Implementing interfaces
class Impl extends Base implements InterfaceA, InterfaceB {
  mustImplement() { /* ... */ }
  // ... InterfaceA and InterfaceB methods
}

// Generic class
class Container<T extends object> {
  constructor(private value: T) {}
  get(): T { return this.value; }
}

// Class decorator
function Decorator(constructor: Function) { /* modify class */ }

// Method decorator
function MethodDec(target: any, key: string, desc: PropertyDescriptor) {
  // Wrap desc.value to intercept calls
}

// Parameter decorator
function ParamDec(target: Object, method: string, index: number) {
  // Record metadata about parameter
}

// Declaration merging
interface Config { host: string; }
interface Config { port: number; }   // Merged!
// Result: { host: string; port: number; }
```

---

## Summary

This week covered the full power of TypeScript's class system:

- **Type annotations** make class contracts explicit and catch errors early
- **Access modifiers** (`public`, `protected`, `private`) enforce encapsulation
- **`readonly`** prevents accidental mutation of important properties
- **Abstract classes** define shared structure and behavior for families of classes
- **Interfaces** let classes commit to multiple contracts simultaneously
- **Generics** make classes reusable across types without sacrificing safety
- **Static members** belong to the class itself, not instances
- **Decorators** enable powerful metaprogramming — logging, caching, validation, and more
- **Parameter decorators** work with metadata to build framework-level features
- **Declaration merging** lets you extend existing types — critical for library augmentation

**Next Week:** Modules, Namespaces, and Advanced Type Manipulation

---

*TypeScript Mastery Series — Week 5 of 12*
