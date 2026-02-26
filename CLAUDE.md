# Claude Agent Instructions — Fullstack TypeScript Application

> **Canon document. Every file you generate or modify MUST comply with these rules.**
> If a rule conflicts with a user's ad-hoc request, ask for clarification before proceeding.

---

## Table of Contents

1. [Architecture](#1-architecture)
2. [TypeScript Type System](#2-typescript-type-system)
3. [Validation & Trust Boundaries](#3-validation--trust-boundaries)
4. [Error Handling](#4-error-handling)
5. [State Management & State Machines](#5-state-management--state-machines)
6. [Testing Strategy](#6-testing-strategy)
7. [CSS & Responsive Design](#7-css--responsive-design)
8. [Accessibility (a11y)](#8-accessibility-a11y)
9. [Security](#9-security)
10. [Logging, Monitoring & Alerting](#10-logging-monitoring--alerting)
11. [Caching](#11-caching)
12. [Infrastructure Concerns](#12-infrastructure-concerns)
13. [Idempotency](#13-idempotency)
14. [Design Patterns & Anti-Patterns](#14-design-patterns--anti-patterns)
15. [Code Style & Functional Programming](#15-code-style--functional-programming)
16. [Dependency Management & Façades](#16-dependency-management--façades)
17. [Frontend Error Reporting](#17-frontend-error-reporting)
18. [Recommended Libraries](#18-recommended-libraries)
19. [API Response Contract & RFC 7807](#19-api-response-contract--rfc-7807)
20. [Cursor-Based Pagination](#20-cursor-based-pagination)
21. [Data Integrity](#21-data-integrity)
22. [HTTP Optimisation](#22-http-optimisation)
23. [Hono-Specific Patterns](#23-hono-specific-patterns)
24. [Frontend Performance](#24-frontend-performance)
25. [Explicit Resource Management](#25-explicit-resource-management)
26. [Checklist Before Every PR](#26-checklist-before-every-pr)

---

## 1. Architecture

### 1.1 Onion Architecture (Ports & Adapters)

Organise every bounded context into concentric layers. **Dependencies point inward only.**

```
┌──────────────────────────────────────────┐
│              Infrastructure              │  ← DB drivers, HTTP clients, queues
│  ┌────────────────────────────────────┐  │
│  │           Application              │  │  ← Use-cases / commands / queries
│  │  ┌──────────────────────────────┐  │  │
│  │  │          Domain              │  │  │  ← Entities, value objects, domain events
│  │  └──────────────────────────────┘  │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
```

- **Domain** – pure functions, zero I/O, zero framework imports.
- **Application** – orchestrates domain logic; defines port interfaces (repositories, gateways).
- **Infrastructure** – adapters that implement ports (Drizzle repo, ky HTTP client, Redis cache).

### 1.2 CQRS (Command / Query Separation)

Separate read models from write models:

```
features/
  orders/
    commands/
      createOrder.command.ts      # mutates state
      createOrder.handler.ts
    queries/
      getOrderById.query.ts       # reads state
      getOrderById.handler.ts
    domain/
      order.entity.ts
      order.valueObjects.ts
      order.events.ts
    ports/
      orderRepository.port.ts     # interface only
    adapters/
      drizzleOrderRepository.ts   # implements port
    validation/
      createOrder.schema.ts       # Zod schema
    __tests__/
      unit/
      integration/
      e2e/
```

- Command handlers return `Result<T, DomainError>`, never raw data.
- Query handlers may return plain DTOs.
- Never call a command handler from a query handler.

### 1.3 Feature-Based Folder Structure

```
src/
  features/
    auth/
    orders/
    payments/
    users/
  shared/
    lib/           # façades (see §16)
    result/        # Result type
    types/         # shared branded types, utility types
    validation/    # shared Zod schemas
    logging/       # wide-log helpers
    cache/         # in-memory cache abstraction
    monitoring/    # alert helpers
    errors/        # error taxonomy
  infrastructure/
    http/          # Hono adapter
    db/            # Drizzle client, migrations
    queue/         # BullMQ adapter
    cache/         # Redis adapter
  ui/              # (frontend)
    components/    # presentational, zero business logic
    hooks/         # React-specific orchestration
    views/         # page-level composition
    state/         # state machines, stores
    styles/        # design tokens, global CSS
    services/      # FE business logic (framework-agnostic)
    adapters/      # API clients (façaded)
```

### 1.4 Frontend Layer Segregation (Radical Decoupling)

Strictly separate three concerns so the view layer is replaceable:

| Layer | Contains | May Import |
|---|---|---|
| **View** (`ui/components/`) | Pure presentational markup + CSS. Accepts props, emits callbacks. Zero hooks beyond `useRef` for DOM measurement. | Design tokens, shared types |
| **React Logic** (`ui/hooks/`, `ui/state/`) | Hooks, context, state machines, subscriptions. | View layer, FE Services |
| **FE Business Logic** (`ui/services/`) | Framework-agnostic pure functions & classes: formatters, validators, calculators, transformation pipelines. | Shared types, shared validation |

> **Rule:** If you can test it without importing `react`, it belongs in `ui/services/`.

---

## 2. TypeScript Type System

### 2.1 Infer From Source of Truth — Never Duplicate

```ts
// ✅ Infer from Zod schema (single source of truth)
const CreateUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1),
});
type CreateUserInput = z.infer<typeof CreateUserSchema>;

// ✅ Infer from Drizzle schema (single source of truth)
type User = typeof users.$inferSelect;
type NewUser = typeof users.$inferInsert;

// ✅ Infer from constants
const ROLES = ["admin", "editor", "viewer"] as const;
type Role = (typeof ROLES)[number];

// ❌ NEVER create a parallel type manually
// type CreateUserInput = { email: string; name: string };
```

### 2.2 Branded Types with Predicate Functions

```ts
declare const __brand: unique symbol;
type Brand<T, B extends string> = T & { readonly [__brand]: B };

type UserId = Brand<string, "UserId">;
type OrderId = Brand<string, "OrderId">;
type Email = Brand<`${string}@${string}.${string}`, "Email">;
type PositiveInt = Brand<number, "PositiveInt">;
type NonEmptyString = Brand<string, "NonEmptyString">;
type ISODateString = Brand<string, "ISODateString">;
type Url = Brand<`${"http" | "https"}://${string}`, "Url">;
type Port = Brand<number, "Port">;

// Predicate / smart constructor — the ONLY way to create a branded value
function isEmail(value: string): value is Email {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value);
}

function toEmail(value: string): Result<Email, ValidationError> {
  return isEmail(value)
    ? ok(value)
    : err(new ValidationError(`Invalid email: ${value}`));
}

function toPositiveInt(n: number): Result<PositiveInt, ValidationError> {
  return Number.isInteger(n) && n > 0
    ? ok(n as PositiveInt)
    : err(new ValidationError(`Expected positive integer, got ${n}`));
}

function toPort(n: number): Result<Port, ValidationError> {
  return Number.isInteger(n) && n >= 1 && n <= 65535
    ? ok(n as Port)
    : err(new ValidationError(`Invalid port: ${n}`));
}
```

### 2.3 Exhaustiveness Checking

```ts
function assertNever(value: never, message?: string): never {
  throw new Error(message ?? `Unexpected value: ${JSON.stringify(value)}`);
}

// Usage in discriminated unions
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.side ** 2;
    default:
      return assertNever(shape);
  }
}
```

### 2.4 Deep Readonly by Default

```ts
import type { ReadonlyDeep } from "type-fest";

// All domain entities are deeply immutable
type Order = ReadonlyDeep<{
  id: OrderId;
  items: OrderItem[];
  status: OrderStatus;
  createdAt: ISODateString;
}>;

// Mutations return new copies
function addItem(order: Order, item: OrderItem): Order {
  return { ...order, items: [...order.items, item] };
}
```

### 2.5 Precise Template Literal Types

```ts
// ❌ Loose
type ApiRoute = string;

// ✅ Precise
type ApiVersion = "v1" | "v2";
type ApiRoute = `/api/${ApiVersion}/${string}`;

// ❌ Loose
type HexColor = string;

// ✅ Precise
type HexDigit = "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9" | "a" | "b" | "c" | "d" | "e" | "f";
type HexColor = `#${string}`;  // pragmatic compromise for 6-char hex
```

### 2.6 Use `type-fest` for Complex Utility Types

```ts
import type {
  ReadonlyDeep,
  SetRequired,
  SetOptional,
  Merge,
  PartialDeep,
  Simplify,
  Opaque,
  JsonValue,
  Promisable,
  RequireAtLeastOne,
  RequireExactlyOne,
  ConditionalKeys,
} from "type-fest";
```

Never re-implement utility types that `type-fest` already provides.

---

## 3. Validation & Trust Boundaries

### 3.1 Golden Rule: Trust Nothing

Validate **every** trust boundary:

| Boundary | What to validate |
|---|---|
| Query parameters | `z.object({ page: z.coerce.number().int().positive().catch(1) })` |
| Route parameters | Branded type + Zod |
| Request bodies | Full Zod schema |
| API responses (own backend) | Zod schema — backends deploy independently |
| API responses (third-party) | Zod schema — they change without notice |
| Environment variables | Zod schema at startup, fail fast |
| localStorage / sessionStorage | Zod schema, `.catch()` for every field |
| URL hash / fragment | Zod |
| WebSocket messages | Zod |
| File uploads | MIME type, size, extension, magic bytes |
| Database reads | Trust Drizzle types, but validate at ingestion |

### 3.2 Zod `.catch()` for Optional Fields

When an optional field fails validation, **do not crash the entire parse**. Use `.catch()` to degrade gracefully.

```ts
const UserPreferencesSchema = z.object({
  // Required — schema WILL fail if missing or invalid
  userId: z.string().uuid(),

  // Optional with safe fallback — schema WILL NOT fail
  theme: z.enum(["light", "dark"]).catch("light"),
  locale: z.string().min(2).max(5).catch("en"),
  pageSize: z.number().int().positive().max(100).catch(20),
  notifications: z.boolean().catch(true),
  lastSeenAt: z.coerce.date().catch(new Date(0)),
  favoriteCategories: z.array(z.string()).catch([]),
  experimental: z.record(z.unknown()).catch({}),
});
```

### 3.3 Environment Validation (Fail Fast)

```ts
// src/shared/validation/env.schema.ts
const EnvSchema = z.object({
  NODE_ENV: z.enum(["development", "production", "test"]),
  PORT: z.coerce.number().int().min(1).max(65535).catch(3000),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  API_RATE_LIMIT_WINDOW_MS: z.coerce.number().int().positive().catch(60000),
  API_RATE_LIMIT_MAX: z.coerce.number().int().positive().catch(100),
  LOG_LEVEL: z.enum(["debug", "info", "warn", "error"]).catch("info"),
  SENTRY_DSN: z.string().url().optional(),
});

export type Env = z.infer<typeof EnvSchema>;

export function parseEnv(): Env {
  const result = EnvSchema.safeParse(process.env);
  if (!result.success) {
    console.error("❌ Invalid environment variables:", result.error.flatten());
    process.exit(1);
  }
  return Object.freeze(result.data);
}
```

---

## 4. Error Handling

### 4.1 Result Type — Errors as First-Class Citizens

**Never use `try/catch` for expected, recoverable errors.** Reserve `try/catch` exclusively for truly unexpected failures (e.g., out-of-memory). Use a `Result` type instead.

```ts
// src/shared/result/result.ts
type Ok<T> = { readonly _tag: "Ok"; readonly value: T };
type Err<E> = { readonly _tag: "Err"; readonly error: E };
type Result<T, E = Error> = Ok<T> | Err<E>;

const ok = <T>(value: T): Ok<T> => ({ _tag: "Ok", value });
const err = <E>(error: E): Err<E> => ({ _tag: "Err", error });

function isOk<T, E>(result: Result<T, E>): result is Ok<T> {
  return result._tag === "Ok";
}

function isErr<T, E>(result: Result<T, E>): result is Err<E> {
  return result._tag === "Err";
}

// Map over Ok values
function mapResult<T, U, E>(result: Result<T, E>, fn: (v: T) => U): Result<U, E> {
  return isOk(result) ? ok(fn(result.value)) : result;
}

// Chain Results
function flatMap<T, U, E>(result: Result<T, E>, fn: (v: T) => Result<U, E>): Result<U, E> {
  return isOk(result) ? fn(result.value) : result;
}

// Unwrap or throw (use at boundaries only)
function unwrapOrThrow<T, E>(result: Result<T, E>): T {
  if (isOk(result)) return result.value;
  throw result.error instanceof Error ? result.error : new Error(String(result.error));
}
```

### 4.2 Domain Error Taxonomy

```ts
// src/shared/errors/domainErrors.ts
abstract class DomainError {
  abstract readonly _tag: string;
  abstract readonly message: string;
  readonly timestamp = new Date().toISOString();
}

class ValidationError extends DomainError {
  readonly _tag = "ValidationError" as const;
  constructor(readonly message: string, readonly field?: string) { super(); }
}

class NotFoundError extends DomainError {
  readonly _tag = "NotFoundError" as const;
  constructor(readonly resource: string, readonly id: string) {
    super();
  }
  get message() { return `${this.resource} not found: ${this.id}`; }
}

class ConflictError extends DomainError {
  readonly _tag = "ConflictError" as const;
  constructor(readonly message: string) { super(); }
}

class AuthorizationError extends DomainError {
  readonly _tag = "AuthorizationError" as const;
  constructor(readonly message: string = "Insufficient permissions") { super(); }
}

class RateLimitError extends DomainError {
  readonly _tag = "RateLimitError" as const;
  constructor(readonly retryAfterMs: number) { super(); }
  get message() { return `Rate limited. Retry after ${this.retryAfterMs}ms`; }
}

//  Add more as the domain requires. Map DomainError tags to HTTP status codes at the adapter level.
type AppError =
  | ValidationError
  | NotFoundError
  | ConflictError
  | AuthorizationError
  | RateLimitError;
```

### 4.3 Error Mapping at the Boundary

```ts
function domainErrorToHttpStatus(error: DomainError): number {
  switch (error._tag) {
    case "ValidationError": return 400;
    case "NotFoundError":   return 404;
    case "ConflictError":   return 409;
    case "AuthorizationError": return 403;
    case "RateLimitError":  return 429;
    default: return assertNever(error as never);
  }
}
```

---

## 5. State Management & State Machines

### 5.1 Finite State Machines to Prevent Impossible States

Use explicit state machines for any entity with a lifecycle. This eliminates invalid boolean combinations such as `{ isLoading: true, isError: true, data: [...] }`.

```ts
// ✅ Discriminated union — impossible states are unrepresentable
type AsyncState<T, E = Error> =
  | { readonly status: "idle" }
  | { readonly status: "loading" }
  | { readonly status: "success"; readonly data: T }
  | { readonly status: "error"; readonly error: E };

// ✅ Order lifecycle — only valid transitions are expressible
type OrderState =
  | { readonly status: "draft" }
  | { readonly status: "placed"; readonly placedAt: ISODateString }
  | { readonly status: "paid"; readonly paidAt: ISODateString; readonly paymentId: string }
  | { readonly status: "shipped"; readonly shippedAt: ISODateString; readonly trackingNumber: string }
  | { readonly status: "delivered"; readonly deliveredAt: ISODateString }
  | { readonly status: "cancelled"; readonly cancelledAt: ISODateString; readonly reason: string };
```

### 5.2 State Transition Functions

```ts
type OrderEvent =
  | { type: "PLACE" }
  | { type: "PAY"; paymentId: string }
  | { type: "SHIP"; trackingNumber: string }
  | { type: "DELIVER" }
  | { type: "CANCEL"; reason: string };

function transition(state: OrderState, event: OrderEvent): Result<OrderState, DomainError> {
  const now = new Date().toISOString() as ISODateString;
  switch (state.status) {
    case "draft":
      if (event.type === "PLACE") return ok({ status: "placed", placedAt: now });
      if (event.type === "CANCEL") return ok({ status: "cancelled", cancelledAt: now, reason: event.reason });
      return err(new ValidationError(`Cannot ${event.type} a draft order`));
    case "placed":
      if (event.type === "PAY") return ok({ status: "paid", paidAt: now, paymentId: event.paymentId });
      if (event.type === "CANCEL") return ok({ status: "cancelled", cancelledAt: now, reason: event.reason });
      return err(new ValidationError(`Cannot ${event.type} a placed order`));
    // ... exhaustive handling
    default:
      return assertNever(state);
  }
}
```

For complex UI state machines, consider using **XState** (via a façade — see §16).

---

## 6. Testing Strategy

### 6.1 Test Pyramid — Aim for 100% Code Coverage

| Layer | Tool | Proportion | What It Covers |
|---|---|---|---|
| **Unit** | `bun:test` | ~70% | Pure functions, domain logic, value objects, utilities, state transitions |
| **Integration** | `bun:test` + MSW + Testcontainers | ~20% | Use-case handlers with real adapters, DB queries, cross-module interactions |
| **E2E** | Playwright | ~10% | Critical user journeys, happy + error paths through the full stack |

### 6.2 Test-Driven Development (TDD) Cycle

1. **Red** — Write a failing test that describes the desired behavior.
2. **Green** — Write the minimal code to make the test pass.
3. **Refactor** — Improve the code while keeping tests green.

Every feature branch must include tests **before** or **alongside** implementation code.

### 6.3 Unit Tests

```ts
// ✅ Test the Result path explicitly — errors are first-class
describe("toEmail", () => {
  it("accepts valid email", () => {
    const result = toEmail("user@example.com");
    expect(isOk(result)).toBe(true);
  });

  it("rejects missing @", () => {
    const result = toEmail("invalid");
    expect(isErr(result)).toBe(true);
  });

  it("rejects empty string", () => {
    const result = toEmail("");
    expect(isErr(result)).toBe(true);
  });
});
```

### 6.4 Integration Tests — Use MSW, Not Manual Mocks

**Never** use `jest.mock()` or `mock.module()` to mock HTTP calls. Use **MSW** (Mock Service Worker) to intercept at the network level.

```ts
import { setupServer } from "msw/node";
import { http, HttpResponse } from "msw";

const server = setupServer(
  http.get("https://api.example.com/users/:id", ({ params }) =>
    HttpResponse.json({ id: params.id, name: "Alice" })
  )
);

beforeAll(() => server.listen({ onUnhandledRequest: "error" }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

it("fetches user through the façade", async () => {
  const result = await userService.getById("123");
  expect(isOk(result)).toBe(true);
  if (isOk(result)) expect(result.value.name).toBe("Alice");
});
```

### 6.5 E2E Tests (Playwright)

```ts
test("user can complete checkout", async ({ page }) => {
  await page.goto("/products");
  await page.getByRole("button", { name: /add to cart/i }).first().click();
  await page.getByRole("link", { name: /cart/i }).click();
  await page.getByRole("button", { name: /checkout/i }).click();
  await expect(page.getByText(/order confirmed/i)).toBeVisible();
});
```

### 6.6 Snapshot Tests for Frequently Changing Output

Use snapshots **only** for outputs that change often and are tedious to assert field-by-field (e.g., serialised API responses, rendered component trees). Keep snapshots small and focused. Update with `--update` flag when intentional changes occur.

```ts
it("serialises order DTO", () => {
  const dto = toOrderDTO(testOrder);
  expect(dto).toMatchSnapshot();
});
```

### 6.7 Testing Error Paths

Every test suite must cover:
- **Happy path**
- **Validation failures** (bad input)
- **Not-found / empty-state**
- **Authorization denied**
- **Network failures** (MSW returning 500 or timeout)
- **Concurrent / race conditions** where applicable

---

## 7. CSS & Responsive Design

### 7.1 Mobile-First

All CSS starts from the smallest viewport and adds complexity upward.

```css
/* Base: mobile */
.card {
  padding: var(--space-3);
  display: flex;
  flex-direction: column;
  gap: var(--space-2);
}

/* Tablet */
@media (min-width: 48rem) {
  .card {
    flex-direction: row;
    padding: var(--space-4);
  }
}

/* Desktop */
@media (min-width: 80rem) {
  .card {
    max-width: 60rem;
    margin-inline: auto;
  }
}
```

### 7.2 Design Tokens as CSS Custom Properties

```css
:root {
  /* Spacing scale */
  --space-1: 0.25rem;
  --space-2: 0.5rem;
  --space-3: 1rem;
  --space-4: 1.5rem;
  --space-5: 2rem;

  /* Typography */
  --font-sans: "Inter", system-ui, sans-serif;
  --font-mono: "Fira Code", monospace;

  /* Colors — semantic tokens */
  --color-surface: hsl(0 0% 100%);
  --color-on-surface: hsl(220 15% 15%);
  --color-primary: hsl(220 90% 56%);
  --color-error: hsl(0 72% 51%);

  /* Radii */
  --radius-sm: 0.25rem;
  --radius-md: 0.5rem;
}
```

### 7.3 Rules

- Prefer `rem` / `em` over `px`.
- Use logical properties (`margin-inline`, `padding-block`).
- Prefer `gap` over margins for flex/grid spacing.
- Use `clamp()` for fluid typography: `font-size: clamp(1rem, 0.5rem + 1vw, 1.25rem)`.
- No `!important` unless overriding third-party styles.
- Use container queries (`@container`) for component-level responsiveness when supported.

---

## 8. Accessibility (a11y)

### 8.1 Non-Negotiable Rules

- **Semantic HTML first** — use `<button>`, `<nav>`, `<main>`, `<article>`, `<dialog>`, etc.
- **All interactive elements are keyboard-accessible** — visible `:focus-visible` ring.
- **All images have `alt` text** — decorative images get `alt=""` and `aria-hidden="true"`.
- **Form inputs have associated `<label>` elements** — never rely on placeholder alone.
- **Colour contrast ≥ 4.5:1** for normal text (WCAG AA).
- **No information conveyed by colour alone** — always add an icon or text label.
- **ARIA only when HTML semantics are insufficient** — prefer native elements.
- **Live regions** (`aria-live="polite"`, `role="alert"`) for dynamic content.
- **Skip-to-content link** as the first focusable element.
- **`prefers-reduced-motion` media query** — disable animations for users who request it:
  ```css
  @media (prefers-reduced-motion: reduce) {
    *, *::before, *::after {
      animation-duration: 0.01ms !important;
      transition-duration: 0.01ms !important;
    }
  }
  ```

### 8.2 Automated Enforcement

- Run `axe-core` checks in every Playwright E2E test.
- Enable the `useSemanticElements`, `useValidAriaRole`, `useValidAriaValues`, and other a11y rules in **Biome**'s linter. Supplement with `@axe-core/playwright` in E2E.
- Include a11y checks in CI: `pa11y-ci` or Lighthouse CI.

---

## 9. Security

### 9.1 Input & Output

- **Validate all inputs** at trust boundaries (§3).
- **Sanitise HTML output** — use a library like `DOMPurify` behind a façade.
- **Parameterised queries only** — never concatenate user input into SQL. Drizzle handles this by default.
- **Escape user-generated content** rendered in templates.

### 9.2 Authentication & Authorization

- **Short-lived JWTs** (15 min access token) + **long-lived refresh tokens** (HTTP-only, Secure, SameSite=Strict cookies).
- **Rotate refresh tokens** on every use (rotation invalidates stolen tokens).
- **bcrypt/argon2** for password hashing — never SHA/MD5.
- **RBAC or ABAC** enforced at the **application layer**, not just the route level.
- **Middleware guards** on every route — no "open by default".

### 9.3 HTTP Security Headers

Set via middleware or reverse proxy:

```
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self';
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

### 9.4 Dependency Security

- `bun audit` (or `bun pm pack --dry-run` + `socket.dev`) in CI — fail on **high** or **critical** vulnerabilities.
- Pin exact versions in `bun.lock`.
- Use `socket.dev` or `snyk` for supply-chain monitoring.

### 9.5 Secrets

- **Never** commit secrets. Use `.env` files (`.gitignore`'d) and inject via CI/CD.
- Validate all env vars at startup with Zod (§3.3).
- Rotate secrets regularly; support graceful key rotation (accept old + new simultaneously during rollout).

### 9.6 CSRF Protection

- Use **SameSite=Strict** cookies + CSRF tokens for state-changing requests.
- For APIs consumed by SPAs, use the **double-submit cookie** pattern or **Origin header validation**.

---

## 10. Logging, Monitoring & Alerting

### 10.1 Wide (Structured) Logs

A "wide log" is a single structured event with **all relevant context** attached rather than multiple narrow log lines.

```ts
// src/shared/logging/logger.ts
interface WideLogEvent {
  readonly timestamp: string;
  readonly level: "debug" | "info" | "warn" | "error";
  readonly message: string;
  readonly service: string;
  readonly traceId: string;
  readonly spanId?: string;
  readonly userId?: string;
  readonly requestId?: string;
  readonly durationMs?: number;
  readonly statusCode?: number;
  readonly method?: string;
  readonly path?: string;
  readonly query?: Record<string, unknown>;
  readonly error?: {
    name: string;
    message: string;
    stack?: string;
    code?: string;
  };
  readonly metadata?: Record<string, unknown>;
}

// Usage — one event per request, rich with context
logger.info({
  message: "Order created",
  traceId,
  userId: session.userId,
  orderId: order.id,
  itemCount: order.items.length,
  totalCents: order.totalCents,
  durationMs: Date.now() - startTime,
  paymentProvider: "stripe",
  idempotencyKey,
});
```

### 10.2 Log Levels

| Level | Use |
|---|---|
| `debug` | Detailed diagnostic info (disabled in production) |
| `info` | Normal operational events: request handled, job completed |
| `warn` | Recoverable issues: retry succeeded, cache miss, deprecated usage |
| `error` | Failures requiring attention: unhandled rejection, integration failure |

### 10.3 Alerting Rules (Examples)

Configure in your monitoring platform (Datadog, Grafana, CloudWatch):

| Alert | Condition | Severity |
|---|---|---|
| Error rate spike | `count(level=error) / count(*) > 5%` over 5 min | P1 |
| Latency degradation | `p99(durationMs) > 2000` for 10 min | P2 |
| Auth failures | `count(statusCode=401) > 50` in 5 min | P1 |
| Rate limiting triggered | `count(statusCode=429) > 100` in 5 min | P2 |
| Unhandled rejection | Any `unhandledRejection` or `uncaughtException` | P1 |
| FE error spike | FE error report count > threshold | P2 |
| DB connection pool exhaustion | Available connections < 2 for 1 min | P1 |

### 10.4 Correlation

- Generate a `traceId` (UUID v4) at the entry point of every request.
- Propagate it via `AsyncLocalStorage` (Bun supports the Node.js `async_hooks` API) so every log within that request lifecycle includes it automatically.
- FE includes a `requestId` header that maps to the BE `traceId`.

---

## 11. Caching

### 11.1 In-Memory Cache in Front of GET Calls / DB Queries

```ts
// src/shared/cache/inMemoryCache.ts
interface CacheEntry<T> {
  readonly value: T;
  readonly expiresAt: number;
}

class InMemoryCache {
  private readonly store = new Map<string, CacheEntry<unknown>>();
  private readonly maxSize: number;

  constructor(maxSize = 1000) {
    this.maxSize = maxSize;
  }

  get<T>(key: string): T | undefined {
    const entry = this.store.get(key);
    if (!entry) return undefined;
    if (Date.now() > entry.expiresAt) {
      this.store.delete(key);
      return undefined;
    }
    return entry.value as T;
  }

  set<T>(key: string, value: T, ttlMs: number): void {
    if (this.store.size >= this.maxSize) {
      // Evict oldest entry (FIFO)
      const firstKey = this.store.keys().next().value;
      if (firstKey !== undefined) this.store.delete(firstKey);
    }
    this.store.set(key, { value, expiresAt: Date.now() + ttlMs });
  }

  invalidate(key: string): void {
    this.store.delete(key);
  }

  invalidatePattern(pattern: RegExp): void {
    for (const key of this.store.keys()) {
      if (pattern.test(key)) this.store.delete(key);
    }
  }

  clear(): void {
    this.store.clear();
  }
}
```

### 11.2 Cache-Aside Pattern for Query Handlers

```ts
async function getOrderByIdHandler(
  query: GetOrderByIdQuery,
  repo: OrderRepository,
  cache: InMemoryCache,
): Promise<Result<OrderDTO, AppError>> {
  const cacheKey = `order:${query.id}`;
  const cached = cache.get<OrderDTO>(cacheKey);
  if (cached) return ok(cached);

  const result = await repo.findById(query.id);
  if (isOk(result)) {
    cache.set(cacheKey, result.value, 60_000); // 1 min TTL
  }
  return result;
}
```

### 11.3 Cache Invalidation

- Invalidate on any **command** (write) that affects the cached resource.
- Use `invalidatePattern` for related cache entries.
- In distributed environments, use Redis pub/sub or similar for cross-instance invalidation.

---

## 12. Infrastructure Concerns

### 12.1 Rate Limiting

Apply at the infrastructure layer (reverse proxy / API gateway) **and** at the application layer as defense-in-depth.

```ts
// Application-level rate limiting (e.g. hono-rate-limiter or rate-limiter-flexible behind a façade)
import type { RateLimiterConfig } from "./rateLimiter.port";

const apiRateLimiter: RateLimiterConfig = {
  windowMs: 60_000,
  max: 100,
  standardHeaders: true,
  legacyHeaders: false,
  keyGenerator: (req) => req.ip ?? "unknown",
  handler: (_req, res) =>
    res.status(429).json({
      error: "Too many requests",
      retryAfterMs: 60_000,
    }),
};
```

### 12.2 Load Balancing

- Use a reverse proxy (NGINX, Caddy, or cloud ALB) in front of multiple Bun instances.
- Ensure the app is **stateless** — all state lives in the DB / Redis / external store.
- Health check endpoint: `GET /health` returns `200` with `{ status: "ok", uptime, version }`.

### 12.3 Graceful Shutdown

```ts
async function gracefulShutdown(server: HttpServer, db: DatabaseClient): Promise<void> {
  logger.info({ message: "Shutting down gracefully..." });
  server.close();           // stop accepting new connections
  await db.disconnect();    // drain DB pool
  await cache.clear();      // optional
  process.exit(0);
}

process.on("SIGTERM", () => gracefulShutdown(server, db));
process.on("SIGINT", () => gracefulShutdown(server, db));
```

---

## 13. Idempotency

### 13.1 Idempotency-Key Header for Mutating Operations

All `POST`, `PUT`, `PATCH` endpoints that create resources or trigger side effects (especially payments) **must** support an `Idempotency-Key` header.

```ts
// Middleware
// Hono middleware
async function idempotencyMiddleware(c: Context, next: Next) {
  const idempotencyKey = c.req.header("idempotency-key");
  if (!idempotencyKey) return next();

  const cachedResponse = await idempotencyStore.get(idempotencyKey);
  if (cachedResponse) {
    return c.json(cachedResponse.body, cachedResponse.statusCode);
  }

  await next();

  // Capture response after handler runs
  const body = await c.res.clone().json();
  idempotencyStore.set(idempotencyKey, {
    statusCode: c.res.status,
    body,
  }, 24 * 60 * 60 * 1000); // 24h TTL
}
```

### 13.2 Rules

- Client generates a UUID v4 as the idempotency key.
- Store must be **shared** across instances (Redis).
- **Lock** the key before processing to prevent concurrent execution of the same key.
- Return the stored response for duplicate requests without re-processing.

---

## 14. Design Patterns & Anti-Patterns

### 14.1 Patterns to Use

| Pattern | When |
|---|---|
| **Façade** | Wrap every external library (§16) |
| **Repository** | Abstract data access behind a port interface |
| **Factory** | Complex object construction, especially for domain entities |
| **Strategy** | Swappable algorithms (payment providers, notification channels) |
| **Observer / Event Emitter** | Domain events (order placed → send email, update inventory) |
| **Builder** | Complex query construction, test fixtures |
| **Decorator** | Cross-cutting concerns (logging, caching, auth) |
| **State Machine** | Entity lifecycle (§5) |

### 14.2 Anti-Patterns to Avoid

| Anti-Pattern | Correct Alternative |
|---|---|
| God class / God module | Split by single responsibility |
| Barrel files (`index.ts` re-exporting everything) | Import directly from the source module |
| `any` type | `unknown` + type narrowing |
| Nested `try/catch` | `Result` type composition |
| Boolean flags for state | Discriminated unions / state machines |
| Mutable global state | Dependency injection, `AsyncLocalStorage` |
| Magic strings / numbers | Constants, enums, branded types |
| Prop drilling > 2 levels | Context / composition / dependency injection |
| `useEffect` for data fetching | React Query / SWR (behind a façade) |
| Over-mocking in tests | MSW for network; real instances for domain logic |
| Premature optimisation | Measure first, optimise second |
| Circular dependencies | Restructure modules, use dependency inversion |

---

## 15. Code Style & Functional Programming

### 15.1 Prefer FP, Use OOP When It Makes Sense

- **Default to pure functions** with explicit inputs and outputs.
- Use **OOP** for entities with identity and lifecycle (domain entities, state machines, cache instances).
- Avoid classes for stateless logic — a plain function is simpler.

### 15.2 Specific Rules

```ts
// ❌ Useless temporary variable
const items = getItems();
return items;

// ✅ Direct return
return getItems();

// ❌ Imperative transformation
const names: string[] = [];
for (const user of users) {
  names.push(user.name);
}

// ✅ Declarative
const names = users.map((user) => user.name);

// ❌ Mutation
user.name = "Alice";

// ✅ Copy
const updatedUser = { ...user, name: "Alice" };

// ❌ Null checks scattered everywhere
if (user !== null && user !== undefined) { ... }

// ✅ Optional chaining + nullish coalescing
const name = user?.name ?? "Anonymous";

// ✅ Pipeline style (when readability improves)
const activeAdminEmails = users
  .filter((u) => u.isActive)
  .filter((u) => u.role === "admin")
  .map((u) => u.email);
```

### 15.3 Immutability

- Mark all function parameters as `readonly` when possible.
- Use `ReadonlyDeep` from `type-fest` for domain types.
- Use `Object.freeze()` for runtime immutability of configuration objects.
- Prefer `ReadonlyArray<T>` over `T[]` in type signatures.
- Prefer `ReadonlyMap` and `ReadonlySet` over mutable counterparts.

---

## 16. Dependency Management & Façades

### 16.1 Façade Every External Library

Every external dependency — including "staples" like React, Zod, Drizzle, ky — is accessed through a **thin façade**. This provides:

1. **Replaceability** — swap the underlying library without touching feature code.
2. **Testability** — mock the façade interface in tests.
3. **Consistent API** — normalise quirks and enforce project conventions.

```
src/shared/lib/
  http/
    httpClient.port.ts        # interface
    kyHttpClient.ts           # adapter (default)
    fetchHttpClient.ts        # alternative adapter
    httpClient.facade.ts      # factory that returns the active adapter
  validation/
    validator.port.ts
    zodValidator.ts
  orm/
    orm.port.ts
    drizzleOrm.ts
  ui/
    uiFramework.port.ts       # abstracts React-specific APIs
    reactUiFramework.ts
  state/
    stateMachine.port.ts
    xstateStateMachine.ts
  cache/
    cache.port.ts
    redisCache.ts
    inMemoryCache.ts
```

### 16.2 Example: HTTP Client Façade

```ts
// src/shared/lib/http/httpClient.port.ts
interface HttpClient {
  get<T>(url: string, options?: RequestOptions): Promise<Result<T, HttpError>>;
  post<T>(url: string, body: unknown, options?: RequestOptions): Promise<Result<T, HttpError>>;
  put<T>(url: string, body: unknown, options?: RequestOptions): Promise<Result<T, HttpError>>;
  delete<T>(url: string, options?: RequestOptions): Promise<Result<T, HttpError>>;
}

interface RequestOptions {
  readonly headers?: Readonly<Record<string, string>>;
  readonly params?: Readonly<Record<string, string>>;
  readonly timeoutMs?: number;
  readonly signal?: AbortSignal;
}

// src/shared/lib/http/kyHttpClient.ts — implements HttpClient using ky
// Feature code only imports HttpClient, never ky directly
```

---

## 17. Frontend Error Reporting

### 17.1 Send Critical FE Errors to a Dedicated Backend Endpoint

```ts
// src/ui/services/errorReporter.ts
interface FrontendError {
  readonly message: string;
  readonly stack?: string;
  readonly componentStack?: string;
  readonly url: string;
  readonly userAgent: string;
  readonly timestamp: string;
  readonly userId?: string;
  readonly sessionId: string;
  readonly metadata?: Record<string, unknown>;
  readonly severity: "warning" | "error" | "fatal";
}

async function reportError(error: FrontendError): Promise<void> {
  // Fire-and-forget with navigator.sendBeacon for reliability during page unload
  const payload = JSON.stringify(error);

  if (navigator.sendBeacon) {
    navigator.sendBeacon("/api/v1/client-errors", payload);
  } else {
    // Fallback with fetch, no await — we don't want to block the UI
    fetch("/api/v1/client-errors", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: payload,
      keepalive: true,
    }).catch(() => {
      // Swallow — if error reporting itself fails, do not recurse
    });
  }
}
```

### 17.2 Global Error Boundary (React)

```tsx
// Wrap the entire app; log to the endpoint above
class GlobalErrorBoundary extends React.Component<Props, State> {
  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo): void {
    reportError({
      message: error.message,
      stack: error.stack,
      componentStack: errorInfo.componentStack ?? undefined,
      url: window.location.href,
      userAgent: navigator.userAgent,
      timestamp: new Date().toISOString(),
      sessionId: getSessionId(),
      severity: "fatal",
    });
  }

  render(): React.ReactNode {
    if (this.state.hasError) {
      return <FallbackErrorPage />;
    }
    return this.props.children;
  }
}
```

### 17.3 Also Catch Unhandled Promise Rejections & Global Errors

```ts
window.addEventListener("unhandledrejection", (event) => {
  reportError({
    message: `Unhandled rejection: ${event.reason}`,
    stack: event.reason?.stack,
    url: window.location.href,
    userAgent: navigator.userAgent,
    timestamp: new Date().toISOString(),
    sessionId: getSessionId(),
    severity: "error",
  });
});

window.addEventListener("error", (event) => {
  reportError({
    message: event.message,
    stack: event.error?.stack,
    url: window.location.href,
    userAgent: navigator.userAgent,
    timestamp: new Date().toISOString(),
    sessionId: getSessionId(),
    severity: "error",
  });
});
```

---

## 18. Recommended Libraries

| Category | Library | Rationale |
|---|---|---|
| **Utility Types** | `type-fest` | Rich type utilities; do not re-invent |
| **Validation** | `zod` (via façade) | Runtime + static type inference |
| **HTTP** | `ky` (via façade) | Retry, timeout, hooks, tiny bundle |
| **API mocking** | `msw` | Network-level interception for tests |
| **Runtime & Package Manager** | `bun` | Fast runtime, built-in test runner, drop-in Node.js replacement |
| **Testing** | `bun:test` | Built-in, Jest-compatible API, fast |
| **E2E** | `playwright` | Cross-browser, reliable |
| **Coverage** | `v8` (via `bun:test --coverage`) | Native V8 coverage, fast |
| **State machines** | `xstate` (via façade) | Formal statecharts |
| **Data structures** | `mnemonist` or `immutable`| Battle-tested advanced data structures |
| **Date/time** | `temporal` polyfill or `date-fns` (via façade) | Immutable, tree-shakeable |
| **ORM** | `drizzle-orm` + `drizzle-kit` (via façade) | Type-safe, SQL-like API, lightweight, schema-as-code |
| **Logging** | `pino` (via façade) | Fast, structured JSON logging |
| **Rate limiting** | `rate-limiter-flexible` | In-memory + Redis support |
| **Sanitisation** | `DOMPurify` (via façade) | XSS prevention |
| **a11y lint** | Biome built-in a11y rules | Catches a11y issues at lint time (no extra plugin) |
| **a11y test** | `@axe-core/playwright` | Runtime a11y checks in E2E |
| **CSS lint** | `stylelint` | Enforce CSS conventions |
| **Linting + Formatting** | `biome` | Lint + format in one tool; extremely fast, no config sprawl |
| **Git hooks** | `husky` + Biome `--staged` flag | Pre-commit quality gates |
| **Server framework** | `hono` | Lightweight, edge-ready, type-safe routes + `hc` RPC client |
| **Hono validation** | `@hono/zod-validator` | Request validation middleware with Zod type inference |
| **Virtualisation** | `@tanstack/react-virtual` (via façade) | Windowed rendering for large lists, tables, grids |
| **Web Vitals** | `web-vitals` | Measure CLS, LCP, INP; report to monitoring |
| **Env parsing** | `zod` (built-in, §3.3) | Fail-fast env validation |
| **ID generation** | `nanoid` or `uuid` | Collision-resistant IDs |

> **Rule:** Before writing any non-trivial data structure or algorithm from scratch, search for a well-maintained library (`bun add`). Prefer installing `mnemonist` over hand-rolling an AVL tree, LRU cache, or trie.

---

## 19. API Response Contract & RFC 7807

### 19.1 Discriminated Union Response Envelope (Wire Format)

`ApiResponse<T>` is the **serialised wire format** for HTTP responses. It complements — not replaces — the in-process `Result<T, E>` type from §4.

| Concern | Type | Layer |
|---|---|---|
| In-process error handling | `Result<T, DomainError>` | Domain / Application |
| HTTP serialisation | `ApiResponse<T>` | Infrastructure (Hono handler) |
| FE after fetch | Parse `ApiResponse` → `Result<T, ProblemDetail>` | UI adapter |

```ts
// Wire format — what goes over HTTP
type ApiResponse<T> =
  | { readonly status: "success"; readonly data: T }
  | { readonly status: "error"; readonly error: ProblemDetail };
```

### 19.2 Bridging Result → ApiResponse in Hono Handlers

The handler unwraps the internal `Result` and maps it to the wire format:

```ts
app.get("/orders/:id", async (c) => {
  const result = await getOrderByIdHandler(c.req.param("id"));

  if (isOk(result)) {
    return c.json({ status: "success" as const, data: result.value });
  }
  // Maps DomainError → ProblemDetail
  return problemResponse(c, result.error);
});
```

### 19.3 Parsing ApiResponse → Result on the FE

On the frontend, parse the wire format **back into** a `Result` so the rest of the FE code uses the same in-process pattern:

```ts
async function fetchApi<T>(
  url: string,
  schema: z.ZodType<T>,
): Promise<Result<T, ProblemDetail>> {
  const res = await httpClient.get(url);
  if (isErr(res)) return res;

  const body = ApiResponseSchema(schema).safeParse(res.value);
  if (!body.success) return err({ type: "ParseError", title: "Invalid response", status: 0, traceId: "" });

  return body.data.status === "success"
    ? ok(body.data.data)
    : err(body.data.error);
}
```

### 19.4 RFC 7807 Problem Details for Errors

All error responses use `application/problem+json`:

```ts
interface ProblemDetail {
  readonly type: string;        // URI reference identifying the error type
  readonly title: string;       // Short human-readable summary
  readonly status: number;      // HTTP status code
  readonly detail?: string;     // Explanation specific to this occurrence
  readonly instance?: string;   // URI identifying this specific occurrence
  readonly traceId: string;     // Correlation ID for debugging
  readonly errors?: ReadonlyArray<{
    readonly field: string;
    readonly message: string;
  }>;
}

// Hono helper — maps DomainError to the wire format
function problemResponse(c: Context, error: DomainError): Response {
  const status = domainErrorToHttpStatus(error);
  return c.json(
    {
      status: "error" as const,
      error: {
        type: `https://api.example.com/errors/${error._tag}`,
        title: error._tag,
        status,
        detail: error.message,
        traceId: getTraceId(),
      },
    },
    status,
  );
}
```

---

## 20. Cursor-Based Pagination

Prefer **cursor/keyset** pagination over offset-based. Offset pagination degrades on large tables and produces inconsistent results under concurrent writes.

```ts
// Shared types
interface CursorPage<T> {
  readonly items: ReadonlyArray<T>;
  readonly nextCursor: string | null;
  readonly hasMore: boolean;
}

const PaginationParamsSchema = z.object({
  cursor: z.string().optional(),
  limit: z.coerce.number().int().min(1).max(100).catch(20),
});
type PaginationParams = z.infer<typeof PaginationParamsSchema>;

// Drizzle query example
async function listOrders(
  params: PaginationParams,
): Promise<CursorPage<Order>> {
  const rows = await db
    .select()
    .from(orders)
    .where(params.cursor ? gt(orders.id, params.cursor) : undefined)
    .orderBy(asc(orders.id))
    .limit(params.limit + 1); // fetch one extra to detect hasMore

  const hasMore = rows.length > params.limit;
  const items = hasMore ? rows.slice(0, -1) : rows;

  return {
    items,
    nextCursor: items.at(-1)?.id ?? null,
    hasMore,
  };
}
```

---

## 21. Data Integrity

### 21.1 Optimistic Locking

Use a `version` column on every mutable entity. Increment on every write. Return `409 Conflict` when the version in the `WHERE` clause matches zero rows.

```ts
// Drizzle example
async function updateOrder(
  id: OrderId,
  data: UpdateOrderInput,
  expectedVersion: number,
): Promise<Result<Order, ConflictError | NotFoundError>> {
  const result = await db
    .update(orders)
    .set({ ...data, version: expectedVersion + 1 })
    .where(and(eq(orders.id, id), eq(orders.version, expectedVersion)))
    .returning();

  if (result.length === 0) {
    const exists = await db.select({ id: orders.id }).from(orders).where(eq(orders.id, id));
    return exists.length === 0
      ? err(new NotFoundError("Order", id))
      : err(new ConflictError("Order was modified by another request. Refresh and retry."));
  }

  return ok(result[0]!);
}
```

### 21.2 Soft Deletes

**Never hard-delete user data.** Use a `deletedAt` column.

```ts
// Schema
const users = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  email: text("email").notNull(),
  deletedAt: timestamp("deleted_at"),
  // ...
});

// All queries exclude soft-deleted rows by default
const activeUsers = () =>
  db.select().from(users).where(isNull(users.deletedAt));

// "Delete" = set the timestamp
const softDelete = (id: UserId) =>
  db.update(users).set({ deletedAt: new Date() }).where(eq(users.id, id));
```

### 21.3 Audit Trail

Log every mutation to an append-only audit table:

```ts
const auditLog = pgTable("audit_log", {
  id: uuid("id").primaryKey().defaultRandom(),
  entityType: text("entity_type").notNull(),  // e.g. "Order"
  entityId: text("entity_id").notNull(),
  action: text("action").notNull(),            // "create" | "update" | "delete"
  actorId: text("actor_id").notNull(),
  before: jsonb("before"),                     // snapshot before mutation
  after: jsonb("after"),                       // snapshot after mutation
  timestamp: timestamp("timestamp").defaultNow().notNull(),
  traceId: text("trace_id"),
});
```

Write the audit row **in the same transaction** as the mutation.

### 21.4 Outbox Pattern for Reliable Events

When a domain event must trigger side effects (email, webhook, analytics), write the event to an `outbox` table **in the same DB transaction** as the mutation. A background poller or CDC (Change Data Capture) picks it up and publishes.

```ts
const outbox = pgTable("outbox", {
  id: uuid("id").primaryKey().defaultRandom(),
  eventType: text("event_type").notNull(),
  payload: jsonb("payload").notNull(),
  createdAt: timestamp("created_at").defaultNow().notNull(),
  processedAt: timestamp("processed_at"),
});

// Inside a transaction:
await db.transaction(async (tx) => {
  await tx.insert(orders).values(newOrder);
  await tx.insert(outbox).values({
    eventType: "order.created",
    payload: newOrder,
  });
});
```

This guarantees that either **both** the order and the event are persisted, or **neither** is — preventing lost or phantom events.

---

## 22. HTTP Optimisation

### 22.1 ETags & Conditional Requests

Return `ETag` headers on GET responses. Honour `If-None-Match` to return `304 Not Modified`.

```ts
// Hono middleware
async function etagMiddleware(c: Context, next: Next) {
  await next();

  if (c.req.method !== "GET" || c.res.status !== 200) return;

  const body = await c.res.clone().text();
  const etag = `"${await hash(body, "sha256")}"`;

  c.header("ETag", etag);

  if (c.req.header("If-None-Match") === etag) {
    c.res = new Response(null, { status: 304 });
  }
}
```

### 22.2 Compression

Enable Brotli / gzip for all responses. Use Hono's `compress` middleware:

```ts
import { compress } from "hono/compress";

app.use("*", compress());
```

For fine-grained control, set `Content-Encoding` conditionally based on `Accept-Encoding`.

### 22.3 Request Coalescing / Deduplication

When multiple identical GET requests arrive simultaneously (thundering herd on cache miss), execute only once and share the result:

```ts
class RequestCoalescer {
  private readonly inflight = new Map<string, Promise<unknown>>();

  async dedupe<T>(key: string, fn: () => Promise<T>): Promise<T> {
    const existing = this.inflight.get(key);
    if (existing) return existing as Promise<T>;

    const promise = fn().finally(() => this.inflight.delete(key));
    this.inflight.set(key, promise);
    return promise;
  }
}

// Usage in a query handler
const coalescer = new RequestCoalescer();

async function getUser(id: UserId) {
  return coalescer.dedupe(`user:${id}`, () => repo.findById(id));
}
```

### 22.4 AbortController Everywhere

Pass `AbortSignal` to **every** async operation. Cancel in-flight work on route change (FE) or request abort (BE).

```ts
// Hono — propagate client abort
app.get("/orders/:id", async (c) => {
  const signal = c.req.raw.signal; // AbortSignal from the incoming request
  const order = await orderService.getById(c.req.param("id"), { signal });
  // If client disconnects, signal fires and downstream calls can abort
  return c.json({ status: "success", data: order });
});

// FE — cancel on component unmount
useEffect(() => {
  const controller = new AbortController();
  fetchOrders({ signal: controller.signal });
  return () => controller.abort();
}, []);
```

---

## 23. Hono-Specific Patterns

### 23.1 Type-Safe Routes with `hc` Client

Leverage Hono's path-parameter inference and the `hc` RPC client for **end-to-end type-safe API calls without codegen**.

```ts
// Backend — define typed routes
const app = new Hono()
  .get("/users/:id", async (c) => {
    const id = c.req.param("id");
    return c.json({ id, name: "Alice" });
  })
  .post("/users", zValidator("json", CreateUserSchema), async (c) => {
    const body = c.req.valid("json");
    return c.json({ id: "new-id", ...body }, 201);
  });

export type AppType = typeof app;

// Frontend — fully typed client (no codegen)
import { hc } from "hono/client";
import type { AppType } from "../server/app";

const client = hc<AppType>("http://localhost:3000");

// client.users[":id"].$get({ param: { id: "123" } }) — fully typed params, response, etc.
const res = await client.users[":id"].$get({ param: { id: "123" } });
const data = await res.json(); // { id: string; name: string } — inferred!
```

### 23.2 Streaming Responses

Use `c.stream()` or `c.streamText()` for large payloads, AI responses, or server-sent events:

```ts
app.get("/export/orders", async (c) => {
  return c.stream(async (stream) => {
    let cursor: string | undefined;
    do {
      const page = await listOrders({ cursor, limit: 100 });
      for (const order of page.items) {
        await stream.write(JSON.stringify(order) + "\n");
      }
      cursor = page.nextCursor ?? undefined;
    } while (cursor);
  });
});
```

### 23.3 Zod Validator Middleware

Use `@hono/zod-validator` to validate request bodies, query params, and route params in a single declaration:

```ts
import { zValidator } from "@hono/zod-validator";

app.post(
  "/orders",
  zValidator("json", CreateOrderSchema),
  zValidator("header", z.object({ "idempotency-key": z.string().uuid() })),
  async (c) => {
    const body = c.req.valid("json");   // fully typed
    const headers = c.req.valid("header");
    // ...
  },
);
```

---

## 24. Frontend Performance

### 24.1 List Virtualisation

For any list that **could** exceed ~50 visible items, use **virtualisation** (windowing). Render only the items in the viewport + a small overscan buffer.

```ts
// Use @tanstack/react-virtual (behind a façade)
import { useVirtualizer } from "@tanstack/react-virtual";

function VirtualList({ items }: { items: ReadonlyArray<Item> }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 48,   // estimated row height in px
    overscan: 5,
  });

  return (
    <div ref={parentRef} style={{ height: "100%", overflow: "auto" }}>
      <div style={{ height: virtualizer.getTotalSize(), position: "relative" }}>
        {virtualizer.getVirtualItems().map((row) => (
          <div
            key={row.key}
            style={{
              position: "absolute",
              top: 0,
              transform: `translateY(${row.start}px)`,
              height: `${row.size}px`,
              width: "100%",
            }}
          >
            <ListItem item={items[row.index]!} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

**Rules:**
- Virtualise **tables, feeds, select dropdowns, and autocomplete results** when the potential item count is unbounded.
- Pair with cursor-based pagination (§20) for infinite scroll.
- Always provide `estimateSize` — avoid layout thrashing.

### 24.2 Optimistic Updates

Apply mutations to the UI **immediately** before the server confirms, then reconcile.

```ts
// Pattern using a state machine
type OptimisticState<T> =
  | { status: "idle"; data: T }
  | { status: "pending"; data: T; optimistic: T; rollback: T }
  | { status: "confirmed"; data: T }
  | { status: "rolledBack"; data: T; error: DomainError };

// Example: toggling a "like"
async function toggleLike(postId: PostId, liked: boolean) {
  // 1. Apply optimistic update immediately
  updatePostCache(postId, (post) => ({
    ...post,
    liked,
    likeCount: post.likeCount + (liked ? 1 : -1),
  }));

  // 2. Send request
  const result = await api.post(`/posts/${postId}/like`, { liked });

  // 3. Reconcile
  if (isErr(result)) {
    // Roll back to previous state
    updatePostCache(postId, (post) => ({
      ...post,
      liked: !liked,
      likeCount: post.likeCount + (liked ? -1 : 1),
    }));
    reportError({ message: "Failed to update like", severity: "warning" });
  }
}
```

**Rules:**
- Always implement **rollback** on failure.
- Show a subtle, non-blocking toast on rollback — never break the flow.
- Use optimistic updates for low-risk, high-frequency actions (likes, toggles, reordering).
- For high-risk actions (payments, deletes), wait for server confirmation.

### 24.3 Skeleton Loading States

**Never** show raw spinners. Use skeleton screens that match the layout of the content being loaded.

```css
.skeleton {
  background: linear-gradient(
    90deg,
    var(--color-surface-dim) 25%,
    var(--color-surface) 50%,
    var(--color-surface-dim) 75%
  );
  background-size: 200% 100%;
  animation: skeleton-shimmer 1.5s infinite;
  border-radius: var(--radius-sm);
}

@keyframes skeleton-shimmer {
  0% { background-position: 200% 0; }
  100% { background-position: -200% 0; }
}

@media (prefers-reduced-motion: reduce) {
  .skeleton { animation: none; }
}
```

### 24.4 Web Vitals Monitoring

Capture Core Web Vitals (CLS, LCP, INP) and send to a dedicated endpoint:

```ts
import { onCLS, onLCP, onINP } from "web-vitals";

function sendVital(metric: { name: string; value: number; id: string }) {
  navigator.sendBeacon(
    "/api/v1/web-vitals",
    JSON.stringify({
      name: metric.name,
      value: metric.value,
      id: metric.id,
      url: window.location.href,
      timestamp: new Date().toISOString(),
      sessionId: getSessionId(),
    }),
  );
}

onCLS(sendVital);
onLCP(sendVital);
onINP(sendVital);
```

Alert when any metric regresses beyond the "good" threshold for > 5% of sessions.

### 24.5 Performance Budgets

Set budgets in CI (Lighthouse CI or `bundlesize`):

| Metric | Budget |
|---|---|
| Total JS bundle (gzip) | ≤ 150 KB |
| LCP | ≤ 2.5 s |
| CLS | ≤ 0.1 |
| INP | ≤ 200 ms |
| Total Blocking Time | ≤ 300 ms |

Fail the pipeline if any budget is exceeded. Budgets are non-negotiable.

---

## 25. Explicit Resource Management

Use the `using` keyword (TC39 Explicit Resource Management, TS 5.2+) for any resource that must be cleaned up: DB connections, file handles, temp files, locks.

```ts
// Drizzle transaction with auto-dispose
async function processOrder(id: OrderId) {
  await using tx = await db.transaction();
  // tx is automatically rolled back if an error is thrown,
  // or committed if the block completes normally

  const order = await tx.select().from(orders).where(eq(orders.id, id));
  // ...
}

// Temp file that cleans itself up
class TempFile implements AsyncDisposable {
  constructor(readonly path: string) {}

  async [Symbol.asyncDispose]() {
    await Bun.file(this.path).exists() && await unlink(this.path);
  }
}

async function processUpload(file: File) {
  await using tmp = new TempFile(`/tmp/${crypto.randomUUID()}`);
  await Bun.write(tmp.path, file);
  // ... process file
} // tmp.path is automatically deleted here
```

**Rules:**
- Prefer `using` / `await using` over manual `try/finally` cleanup.
- Implement `Disposable` / `AsyncDisposable` on custom resource wrappers.
- Bun supports this natively.

---

## 26. Checklist Before Every PR

- [ ] All new code is covered by tests (unit + integration as appropriate).
- [ ] Error paths are tested explicitly.
- [ ] `Result` type used for recoverable errors — no `try/catch` for expected failures.
- [ ] Zod schemas validate all trust boundaries.
- [ ] Optional Zod fields use `.catch()` with sane defaults.
- [ ] No `any` — use `unknown` + narrowing.
- [ ] All types inferred from source of truth (Zod, Drizzle, `as const`).
- [ ] Branded types with predicates for domain identifiers.
- [ ] Exhaustiveness checking with `assertNever` on discriminated unions.
- [ ] `ReadonlyDeep` on domain types.
- [ ] CSS is mobile-first; no `px` for layout; uses design tokens.
- [ ] a11y: semantic HTML, labels, contrast, keyboard nav.
- [ ] Security: no secrets in code, CSP headers, parameterised queries.
- [ ] Wide structured logs with `traceId` on all operations.
- [ ] HTTP mocks use MSW, not `mock.module()`.
- [ ] External libraries accessed through façades.
- [ ] State machines for entities with lifecycle.
- [ ] Idempotency-Key on mutating endpoints.
- [ ] In-memory cache in front of repeated GET / DB reads.
- [ ] No useless temp variables; prefer direct returns and FP chaining.
- [ ] FE critical errors report to `/api/v1/client-errors`.
- [ ] No high/critical vulnerabilities (use `socket.dev` or equivalent supply-chain scanning).
- [ ] API errors use RFC 7807 `ProblemDetail` format.
- [ ] All list endpoints use cursor-based pagination.
- [ ] Mutable entities have `version` column (optimistic locking).
- [ ] No hard deletes — use `deletedAt` soft-delete.
- [ ] Mutations write an audit trail row in the same transaction.
- [ ] Side-effecting domain events use the outbox pattern.
- [ ] GET responses include `ETag`; honour `If-None-Match`.
- [ ] Response compression (Brotli/gzip) enabled.
- [ ] `AbortSignal` propagated through all async chains.
- [ ] Hono routes use `hc<AppType>` for type-safe FE client.
- [ ] Large lists (> 50 items) are virtualised.
- [ ] Optimistic updates have rollback logic on failure.
- [ ] Skeleton loading states — no raw spinners.
- [ ] Web Vitals (CLS, LCP, INP) reported to monitoring.
- [ ] Performance budgets enforced in CI.
- [ ] `using` / `await using` for resources requiring cleanup.

---

## Appendix A: Additional Claude-Specific Directives

1. **When generating a new feature**, scaffold the full feature folder first (domain, commands, queries, ports, adapters, validation, `__tests__/`).
2. **When modifying existing code**, read and understand the surrounding context before editing. Do not break existing patterns.
3. **Never generate dead code** — every line must be reachable and tested.
4. **Prefer composition over inheritance.**
5. **Every public function must have a JSDoc comment** explaining purpose, params, return value, and thrown/returned errors.
6. **File length limit: 300 lines.** If a file exceeds this, split by responsibility.
7. **No default exports** — named exports only, for better refactoring and grep-ability.
8. **Strict TypeScript config:**
   ```jsonc
   {
     "compilerOptions": {
       "strict": true,
       "noUncheckedIndexedAccess": true,
       "exactOptionalPropertyTypes": true,
       "noImplicitReturns": true,
       "noFallthroughCasesInSwitch": true,
       "noPropertyAccessFromIndexSignature": true,
       "forceConsistentCasingInFileNames": true,
       "verbatimModuleSyntax": true
     }
   }
   ```
9. **Biome** must be configured with the `recommended` + `all` ruleset and run on pre-commit (via `husky` + `biome check --staged`).
10. **Every API endpoint must be documented** with OpenAPI / Swagger annotations or a co-located `.schema.ts` file.
11. **Database migrations must be reversible** — every `up` migration has a corresponding `down`.
12. **Feature flags** for risky deployments — wrap new behaviour behind flags, not branches.
13. **Timeouts on every external call** — HTTP, DB, Redis, third-party APIs. No indefinite waits.
14. **Retry with exponential backoff + jitter** for transient failures on external calls.
15. **Circuit breaker pattern** for third-party integrations that may go down — prevent cascade failures.
16. **Health check endpoint** (`GET /health`) returning service status, uptime, and dependency health.
17. **Structured concurrency** — use `Promise.allSettled` over `Promise.all` when partial failures are acceptable. Always handle every settled result.
18. **Dead letter queue** for failed async jobs — never silently drop messages.
