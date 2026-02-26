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
19. [Checklist Before Every PR](#19-checklist-before-every-pr)

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
| **Env parsing** | `zod` (built-in, §3.3) | Fail-fast env validation |
| **ID generation** | `nanoid` or `uuid` | Collision-resistant IDs |

> **Rule:** Before writing any non-trivial data structure or algorithm from scratch, search for a well-maintained library (`bun add`). Prefer installing `mnemonist` over hand-rolling an AVL tree, LRU cache, or trie.

---

## 19. Checklist Before Every PR

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
