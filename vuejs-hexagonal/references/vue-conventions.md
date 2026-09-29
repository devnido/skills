# Vue.js Conventions — Hexagonal Architecture

These are the canonical rules and code templates for a **Vue.js SPA** built with hexagonal
architecture (Ports & Adapters) and vertical slicing. Read this file together with
`frontend-spa-vue.md` (setup, structure, DI, MVVM, router, layouts) before generating any
structure. `SKILL.md` carries only a digest of these rules — **this file is the canonical
source.** Generate every template below exactly as shown — do not rename classes, generics,
or methods.

## Table of Contents

1. [Result Pattern](#result-pattern)
2. [Domain Base Classes](#domain-base-classes)
3. [Paginated Container](#paginated-container)
4. [Domain Entity Rules](#domain-entity-rules)
5. [Value Object Convention](#value-object-convention)
6. [Port Parameters Convention](#port-parameters-convention)
7. [UseCase Base Class](#usecase-base-class)
8. [Form Model & Form Mapper](#form-model--form-mapper-convention)
9. [Screen Mapper & Presentation Model](#screen-mapper--presentation-model-convention)
10. [Auth & Session Convention](#auth--session-convention)
11. [Testing Conventions](#testing-conventions)
12. [Code Quality](#code-quality)
13. [Dependency Rule](#dependency-rule-enforced-via-eslint-plugin-boundaries)
14. [src/base Structure](#srcbase-structure)
15. [Environment Variables Convention](#environment-variables-convention)
16. [Constants Convention](#constants-convention)
17. [Utils Convention](#utils-convention)
18. [Quick Reference: Use Case File Structure](#quick-reference-use-case-file-structure)

---

## Result Pattern

> **The `Result<T>` pattern is the SPA's error channel.** Nothing in `domain/` or
> `application/` ever throws: failures travel as `Result.err(...)` up to the ViewModel, which
> is the single unwrap point. If you are porting rules from a NestJS backend, note the
> difference — the backend throws and maps exceptions in a global filter; in the SPA the only
> `throw` is the ViewModel's, inside a Pinia Colada `query` / `mutation` function, and the
> query engine plays the role of that filter.

The project includes a manual `Result<T>` class in `src/base/lib/domain/result.ts`. The error type is **always `DomainException`** — it is not a generic parameter:

```typescript
// src/base/lib/domain/result.ts
import type { DomainException } from './domain-exception.base'

export class Result<T> {
  private constructor(
    private readonly value?: T,
    private readonly error?: DomainException,
  ) {}

  static ok<T>(value: T): Result<T> {
    return new Result<T>(value, undefined)
  }

  static err<T = never>(error: DomainException): Result<T> {
    return new Result<T>(undefined, error)
  }

  isOk(): boolean {
    return this.error === undefined
  }

  isErr(): boolean {
    return this.error !== undefined
  }

  getValue(): T {
    if (this.isErr()) throw new Error('Cannot get value of an error result')
    return this.value as T
  }

  getError(): DomainException {
    if (this.isOk()) throw new Error('Cannot get error of a success result')
    return this.error as DomainException
  }
}
```

Rules:
- Use `Result.ok(value)` and `Result.err(error)` in the **domain** and **application** layers
- The error side is always `DomainException` — any concrete exception (e.g. `UserNotFoundException`) works because it `extends DomainException`
- Infrastructure adapters `try/catch` the HTTP client and convert **every** failure into `Result.err(...)` (mapping known statuses to domain exceptions; the `HttpServiceException` itself — it extends `DomainException` — is the generic fallback). Use cases never see exceptions: the **ViewModel** is the frontend's driving boundary — inside its Pinia Colada `query` / `mutation` function it unwraps with `if (result.isErr()) throw result.getError()`, the query engine captures that `DomainException` and exposes it as `error`, and the ViewModel maps its `code` to UI error state. Outside those functions it never throws, and `try/catch` is not its error channel (see the MVVM Convention and *Server State with Pinia Colada* in `frontend-spa-vue.md`).
- In **domain**, **application** and **infrastructure**, never throw domain errors — always return `Result.err(new UserNotFoundException(id))`. The only `throw` of a `DomainException` in the SPA is the ViewModel's, inside a `query` / `mutation` function.

## Domain Base Classes

The project includes these abstract base classes in `src/base/lib/domain/`. **Generate each file exactly as shown — do not rename classes:**

```typescript
// src/base/lib/domain/command.base.ts
export abstract class Command {
  private readonly _brand!: 'Command'
}

// src/base/lib/domain/query.base.ts
export abstract class Query {
  private readonly _brand!: 'Query'
}

// src/base/lib/domain/props.base.ts
export abstract class Props {
  private readonly _brand!: 'Props'
}

// src/base/lib/domain/domain-exception.base.ts
export abstract class DomainException extends Error {
  abstract readonly code: string

  constructor(message: string) {
    super(message)
    this.name = new.target.name
  }
}
```

Why these shapes:
- The `_brand` field is a definite-assignment private member: it emits no JavaScript, but it makes
  each base class **nominal**. Without it, TypeScript's structural typing lets any object satisfy
  an empty class, so `UseCase<I extends Input>` would accept anything. Private members are compared
  nominally, so `Command`, `Query`, and `Props` stay mutually incompatible.
- `DomainException` extends `Error` so every domain error carries a real `message` and stack trace.
  Each concrete exception declares a stable `code` (UPPER_SNAKE_CASE) that the **ViewModel** maps to
  user-facing UI state — never map by class name (`constructor.name` breaks under minification).

All Props (and any `Command` / `Query` you choose to model) and every domain error must extend these base classes:
```typescript
// domain/props/create-user.props.ts
import { Props } from '@/base/lib/domain/props.base'
export class CreateUserProps extends Props { ... }

// domain/exceptions/user-not-found.exception.ts
import { DomainException } from '@/base/lib/domain/domain-exception.base'
export class UserNotFoundException extends DomainException {
  readonly code = 'USER_NOT_FOUND'
  constructor(id: string) { super(`User ${id} not found`) }
}
```

## Paginated Container

The project includes a generic `Paginated<T>` class in `src/base/lib/domain/paginated.ts`.
It is the canonical return shape for any use case that yields a collection. **Generate this file
exactly as shown:**

```typescript
// src/base/lib/domain/paginated.ts
export class Paginated<T> {
  readonly items: T[]
  readonly total: number
  readonly page: number
  readonly limit: number

  constructor(items: T[], total: number, page: number, limit: number) {
    this.items = items
    this.total = total
    this.page = page
    this.limit = limit
  }
}
```

Rules:
- A use case returning a collection returns `Paginated<Entity>`, never a raw array.
- **No `totalPages` field.** `totalPages` is a derived presentation value computed in the
  presentation layer (Screen Mapper / ViewModel), never in the application layer.
- `Paginated<T>` is instantiated directly (`new Paginated(items, total, page, limit)`), so its
  file is `paginated.ts` (not `*.base.ts`, which is reserved for abstract bases meant to be
  extended).

## Domain Entity Rules

Domain entities (e.g. `User`, `Product`, `Spot`) represent persisted business objects and follow strict construction rules.

**1. Required attributes — every domain entity MUST have:**
- `id` — non-nullable, assigned by the persistence layer (or generated upstream when justified)
- `createdAt: Date` — non-nullable, set when the entity is first persisted
- `updatedAt: Date` — non-nullable, updated on every mutation

These three fields are **never optional and never nullable**. An object missing any of them is not a valid domain entity.

```typescript
// domain/entities/user.entity.ts
export class User {
  constructor(
    public readonly id: string,
    public readonly email: string,
    public readonly name: string,
    public readonly createdAt: Date,
    public readonly updatedAt: Date,
  ) {}
}
```

There is **no** shared `Entity` base class. Each entity declares its own fields explicitly.

**2. Instantiation rules — `new <Entity>(...)` is only allowed in three places:**

   **a) Inside an infrastructure → domain mapper** (the canonical case). Adapters receive raw data from a data source (DB row, HTTP response, ORM model) and a mapper rebuilds the entity:
   ```typescript
   // infrastructure/mappers/user.mapper.ts
   export class UserMapper {
     static toDomain(raw: UserRecord): User {
       return new User(raw.id, raw.email, raw.name, raw.created_at, raw.updated_at)
     }
   }
   ```

   **b) Inside a test builder** (`tests/modules/<module>/builders/`) to create fixtures — builders live in `tests/`, never in `src/` (see *Testing Conventions*).

   **c) Inside a use case, only when ALL of the following hold:**
   - Every required field (`id`, `createdAt`, `updatedAt`, plus all business fields) is already available in memory — no nullables, no placeholders, no `new Date()` fillers for `createdAt` of something that hasn't been persisted yet.
   - There is a **justifiable purpose** for constructing it in the use case rather than delegating to an adapter (e.g. assembling an entity from already-fetched pieces, in-memory projection, test fixtures inside the use case is **not** a valid reason).
   - If in doubt, delegate to the adapter and let the mapper build it.

**3. Forbidden:**
- ❌ Instantiating an entity in presentation, application orchestration, or anywhere else.
- ❌ Instantiating an entity to represent data that does **not yet exist** in the data source (e.g. "build a `User` to pass to `userRepository.create(user)`"). For creation/modification flows, pass a `Command`, `Query`, or `Props` to the adapter — the adapter persists and returns the fully-formed entity.
- ❌ Making `id`, `createdAt`, or `updatedAt` optional, nullable, or defaulted in the constructor.

**4. Data flow for create/update:**
```
Presentation (FormModel)
  → FormMapper.toProps()
  → UseCase.execute(props)
  → Adapter.create(props)         ← adapter persists, receives id/timestamps from DB
  → Mapper.toDomain(record)       ← entity instantiated HERE
  → returned up the stack as User
```
The use case **never** calls `new User(...)` to hand it to the adapter for creation.

## Value Object Convention

Domain value objects come in two shapes — pick by whether the value carries behavior:

- **Validated single-value VO** — a `class` that guards one primitive and enforces an invariant
  on construction (e.g. an email). Named `<Entity><Field>ValueObject`, file
  `<entity>-<field>.value-object.ts`, may extend `value-object.base.ts`.
- **Structural data VO** — a plain **`interface`** describing a multi-field embedded value with
  no behavior (e.g. `Location`, `Image`, `AddedBySkater`, `LikedBySkater`, `CommentedBySkater`).
  No `ValueObject` suffix; file `<name>.ts` inside a `domain/value-objects/` folder.

**Placement** (business value, never `src/base/` — that is for technical machinery only):
- A VO that is **business shared by ≥2 modules** (e.g. `Image`, `Location`) lives in
  `src/modules/shared/domain/value-objects/`.
- A VO **embedded in / owned by a single aggregate** lives in that module's own
  `domain/value-objects/` (e.g. the spot-only `AddedBySkater` / `LikedBySkater` /
  `CommentedBySkater`).

## Port Parameters Convention

Port functions (repository methods, service methods) receive their parameters as domain classes, and always return a `Promise<Result<T>>`. There are 2 cases.

**Case 1 — Same params as the use case:** reuse the Input directly (Props/Command/Query)
```typescript
// The port receives the same Props the use case received
export interface UserRepository {
  save(props: CreateUserProps): Promise<Result<User>>
}
```

**Case 2 — Different params:** create a `<FunctionName>Props` class in `domain/props/`
```typescript
// domain/props/find-by-email.props.ts
import { Props } from '@/base/lib/domain/props.base'
export class FindByEmailProps extends Props {
  constructor(public readonly email: string) { super() }
}

// The port receives a specific Props class
export interface UserRepository {
  findByEmail(props: FindByEmailProps): Promise<Result<User>>
}
```

## UseCase Base Class

The project includes an abstract `UseCase` class in `src/base/lib/application/use-case.base.ts`. **Generate this file exactly as shown — do not rename the class, generics, or method:**

```typescript
// src/base/lib/application/use-case.base.ts
import type { Result } from '../domain/result'
import type { Command } from '../domain/command.base'
import type { Query } from '../domain/query.base'
import type { Props } from '../domain/props.base'

type Input = Command | Query | Props

export abstract class UseCase<I extends Input, O> {
  abstract execute(input: I): Result<O> | Promise<Result<O>>
}
```

All use cases **extend** this class (never `implements`). A use case returns **domain data
wrapped in a `Result`** — the presentation layer adapts it via Screen Mapper / Presentation
Model. It never returns a Presentation Model, a raw HTTP DTO, or an unwrapped value.
```typescript
// Single result
export class UserCreator extends UseCase<CreateUserProps, User> {
  execute(props: CreateUserProps): Promise<Result<User>> {
    // ...
  }
}

// Paginated collection
export class UsersFinder extends UseCase<FindUsersProps, Paginated<User>> {
  execute(props: FindUsersProps): Promise<Result<Paginated<User>>> {
    // ...
  }
}
```

**DI (Awilix):** there is no framework DI container — use cases and adapters are registered in
the typed Awilix container (`src/base/config/di/container.ts`): a use case under its own name,
a port under the port's key bound to its adapter (e.g.
`userRepository: asFunction((cradle: Cradle) => new HttpUserRepository(cradle.httpClient))`).
Every registration is wired **by hand with `asFunction`** — `asClass` +
`InjectionMode.CLASSIC` resolves by constructor parameter names and therefore breaks in any
minified production build. Only ViewModels resolve from the container. See the DI section of
`frontend-spa-vue.md`.

## Form Model & Form Mapper Convention

When a screen has a form, the presentation layer defines:

1. **Form Model** — an `interface` in `presentation/models/` that represents the form fields as the UI sees them
2. **Form Mapper** — a static class in `presentation/mappers/` that converts the Form Model into the domain `Props` class before calling the use case

This keeps the presentation layer decoupled from the domain: the form works with its own model, and the mapper handles the translation.

```typescript
// presentation/models/create-user.form-model.ts
export interface CreateUserFormModel {
  name: string
  email: string
  passwordConfirmation: string  // UI-only field, not part of domain Props
}

// presentation/mappers/create-user.form-mapper.ts
import type { CreateUserFormModel } from '../models/create-user.form-model'
import { CreateUserProps } from '../../domain/props/create-user.props'

export class CreateUserFormMapper {
  static toProps(form: CreateUserFormModel): CreateUserProps {
    return new CreateUserProps(form.name, form.email)
  }
}
```

The ViewModel uses the mapper before calling the use case:
```typescript
// inside ViewModel
const props = CreateUserFormMapper.toProps(formData)
const result = await userCreator.execute(props)
```

Rules:
- Form Models are **interfaces** (not classes) — they represent plain form state
- Form Mappers are **static classes** — no instantiation needed
- The mapper receives the Form Model and returns a domain `Props` instance
- Fields that exist only in the UI (e.g. `passwordConfirmation`) are stripped by the mapper
- File naming: `<action>-<entity>.form-model.ts` and `<action>-<entity>.form-mapper.ts`

## Screen Mapper & Presentation Model Convention

Every screen that consumes domain entities defines a **Screen Mapper** that converts those entities into one or more **Presentation Models** tailored to what the UI actually renders. This is the only place in the frontend that translates domain → UI.

**File and class naming:**
- The mapper file is named after the **screen** in kebab-case: `login-screen.mapper.ts`, `user-profile-screen.mapper.ts`. The class is `<Screen>Mapper`.
- One mapper per screen, even if the screen consumes multiple entities. A single mapper can map several different entities — that is why it is screen-scoped, not entity-scoped.
- Presentation Model files use the `.model.ts` suffix and the class/interface ends in `Model`. The base name is **free** (let context decide): a single screen often needs more than one model (`UserSummaryModel`, `UserPermissionsModel`, `RecentActivityModel`), and forcing the screen name on every model would be misleading.

**Method naming inside the mapper:**
Each mapping method is named after its source and target with the pattern `<source>To<target>`, where the source/target use the original class names (entity, model, dto):
- `<Entity>DomainTo<Name>Model` — domain entity → presentation model
- `<Name>ModelTo<Entity>Domain` — presentation model → domain entity (when needed)

This keeps the direction explicit and avoids ambiguous `toModel` / `fromModel` names when one mapper handles multiple types.

```typescript
// presentation/models/user-summary.model.ts
export interface UserSummaryModel {
  fullName: string
  initials: string
  joinedLabel: string  // pre-formatted for the UI, e.g. "Joined 2 months ago"
}

// presentation/models/user-permissions.model.ts
export interface UserPermissionsModel {
  canEdit: boolean
  canDelete: boolean
}

// presentation/mappers/user-profile-screen.mapper.ts
import type { User } from '@/modules/user/domain/user.entity'
import type { UserSummaryModel } from '../models/user-summary.model'
import type { UserPermissionsModel } from '../models/user-permissions.model'

export class UserProfileScreenMapper {
  static userDomainToUserSummaryModel(user: User): UserSummaryModel {
    return {
      fullName: `${user.firstName} ${user.lastName}`,
      initials: `${user.firstName[0]}${user.lastName[0]}`.toUpperCase(),
      joinedLabel: formatRelative(user.createdAt),
    }
  }

  static userDomainToUserPermissionsModel(user: User): UserPermissionsModel {
    return {
      canEdit: user.role === 'admin' || user.role === 'editor',
      canDelete: user.role === 'admin',
    }
  }
}
```

Rules:
- **Only map fields the UI actually uses**. Never spread an entity or copy fields the screen does not render.
- One mapper per screen. The mapper file is named after the screen.
- Method names follow `<Source>To<Target>` using the real class names — never generic `toModel` / `fromModel`.
- Presentation Models are interfaces (not classes) unless behavior is needed.
- Models live in `presentation/models/` with free base names + `Model` suffix.

## Auth & Session Convention

Token/session handling is **technical machinery**, not business: it lives in
`src/base/config/auth/`, next to the HTTP client that consumes it. The auth *flows*
(login, register, logout screens) are a regular `modules/auth/` module that follows every
convention above — use cases, ports, adapters, MVVM — nothing about them is special.

### 1. Token storage port

`src/base/config/auth/token-storage.port.ts` — one interface, one adapter per platform:

```typescript
// src/base/config/auth/token-storage.port.ts
export interface AuthTokens {
  accessToken: string
  refreshToken: string
}

export interface TokenStorage {
  get(): Promise<AuthTokens | null>
  save(tokens: AuthTokens): Promise<void>
  clear(): Promise<void>
}
```

| Adapter | Storage |
|---|---|
| `local-storage.token-storage.ts` | `localStorage` (accepted XSS trade-off; prefer an in-memory access token + a refresh cookie when the API supports it) |

### 2. HTTP client integration (request header + 401 refresh)

The base HTTP client (`src/base/config/http/`) owns both auth interceptors:

- **Request interceptor** — injects `Authorization: Bearer <accessToken>` plus any stable
  client-identity headers the API requires (e.g. a `device-id` UUID generated once and
  persisted alongside the tokens).
- **Response interceptor (401)** — performs a **single-flight refresh**: the first 401
  triggers the refresh call, concurrent 401s await the same promise, and each queued
  request retries **once** with the new token. Never retry a request more than once.
- The refresh call uses a **bare client** (no interceptors) — refreshing through the
  intercepted client recurses when the refresh itself returns 401.
- If the refresh fails: `clear()` the storage and notify the session store so layouts
  redirect to login. The HTTP client **never navigates** — navigation is presentation's job.

```typescript
// src/base/config/auth/refresh-session.ts (sketch — bareClient is a plain axios
// instance with baseURL only, created in this file, with NO interceptors)
import type { AuthTokens, TokenStorage } from './token-storage.port'

let refreshing: Promise<AuthTokens | null> | null = null

export function refreshSession(storage: TokenStorage): Promise<AuthTokens | null> {
  refreshing ??= doRefresh(storage).finally(() => {
    refreshing = null
  })
  return refreshing
}

async function doRefresh(storage: TokenStorage): Promise<AuthTokens | null> {
  const tokens = await storage.get()
  if (!tokens) return null
  try {
    const { data } = await bareClient.post('/auth/refresh-token', tokens)
    await storage.save(data)
    return data
  } catch {
    await storage.clear()
    return null
  }
}
```

### 3. Session state

- Current-user state (`isAuthenticated`, profile summary) lives in the **auth module's
  store** (`modules/auth/presentation/store/auth.store.ts`). It is cross-screen state, so
  the MVVM rules apply: only ViewModels read/write it, Views never import it.
- `PrivateLayout` / `usePrivateLayoutViewModel` guards private screens by reading that
  store (see the Layouts Convention in `frontend-spa-vue.md`).
- Tokens never live in the store — the store holds *who is logged in*, the storage holds
  *the credentials*. Components never see tokens.

## Testing Conventions

| Layer | Test Type | Mock Strategy |
|---|---|---|
| `domain/` | Unit tests — pure, no mocks | None |
| `application/` | Unit tests | Mock ports (interfaces) |
| `infrastructure/` | Unit tests | Mock HTTP clients, ORM clients, etc. |
| `presentation/` | Component tests | Mock the ViewModel composable |

Testing stack: **Vitest** as the runner and **`@vue/test-utils`** for component tests.

Test files live in a `tests/` folder at the root level, mirroring the `src/` structure:
```
src/modules/user/application/use-cases/user-creator/user-creator.use-case.ts
tests/modules/user/application/use-cases/user-creator/user-creator.use-case.spec.ts
```

### Test builders

Shared fixtures live in `tests/modules/<module>/builders/` as `build<Entity>(overrides?)`
functions. A builder is the only place in `tests/` allowed to call `new <Entity>(...)`:

```typescript
// tests/modules/user/builders/user.builder.ts
import { User } from '@/modules/user/domain/entities/user.entity'

export function buildUser(overrides: Partial<User> = {}): User {
  return new User(
    overrides.id ?? 'user-1',
    overrides.email ?? 'tony@hawk.cl',
    overrides.name ?? 'Tony Hawk',
    overrides.createdAt ?? new Date('2026-01-15T10:00:00Z'),
    overrides.updatedAt ?? new Date('2026-01-15T10:00:00Z'),
  )
}
```

### Canonical test templates

Generate tests from these templates — do not improvise the mocking strategy.

**1. Use case — mock the ports, assert on the returned `Result`:**

```typescript
// tests/modules/user/application/use-cases/user-creator/user-creator.use-case.spec.ts
import { UserCreator } from '@/modules/user/application/use-cases/user-creator/user-creator.use-case'
import { CreateUserProps } from '@/modules/user/domain/props/create-user.props'
import type { UserRepository } from '@/modules/user/domain/ports/user.repository'
import { UserAlreadyExistsException } from '@/modules/user/domain/exceptions/user-already-exists.exception'
import { Result } from '@/base/lib/domain/result'
import { buildUser } from '../../../builders/user.builder'

describe('UserCreator', () => {
  const save = vi.fn()
  const userRepository: UserRepository = { save }
  const useCase = new UserCreator(userRepository)

  it('returns the entity when the repository saves', async () => {
    save.mockResolvedValue(Result.ok(buildUser({ id: 'user-1' })))

    const result = await useCase.execute(new CreateUserProps('Tony', 'tony@hawk.cl'))

    expect(result.isOk()).toBe(true)
    expect(result.getValue().id).toBe('user-1')
  })

  it('propagates the domain error the repository returned', async () => {
    save.mockResolvedValue(Result.err(new UserAlreadyExistsException('tony@hawk.cl')))

    const result = await useCase.execute(new CreateUserProps('Tony', 'tony@hawk.cl'))

    expect(result.isErr()).toBe(true)
    expect(result.getError().code).toBe('USER_ALREADY_EXISTS')
  })
})
```

Rules: mock **only the ports**; assert errors by `code`, never by exception class name; the use
case never throws, so never assert with `rejects`; never touch the network.

**2. Screen Mapper / Form Mapper — pure, no mocks:**

```typescript
// tests/modules/user/presentation/mappers/user-profile-screen.mapper.spec.ts
import { UserProfileScreenMapper } from '@/modules/user/presentation/mappers/user-profile-screen.mapper'
import { buildUser } from '../../builders/user.builder'

describe('UserProfileScreenMapper', () => {
  it('maps the entity to the model the screen renders', () => {
    const user = buildUser({ firstName: 'Tony', lastName: 'Hawk' })

    const model = UserProfileScreenMapper.userDomainToUserSummaryModel(user)

    expect(model.fullName).toBe('Tony Hawk')
    expect(model.initials).toBe('TH')
  })

  it('exposes only the fields the UI uses', () => {
    const model = UserProfileScreenMapper.userDomainToUserSummaryModel(buildUser())

    expect(model).not.toHaveProperty('password') // internal/sensitive fields stripped
  })
})
```

**3. Adapter — mock the HTTP client, assert that every failure becomes a `Result.err`:**

```typescript
// tests/modules/user/infrastructure/repositories/http-user.repository.spec.ts
import { HttpUserRepository } from '@/modules/user/infrastructure/repositories/http-user.repository'
import { HttpServiceException } from '@/base/config/http/exception/http-service.exception'
import { CreateUserProps } from '@/modules/user/domain/props/create-user.props'
import type { AxiosHttpClient } from '@/base/config/http/axios.http-client'

describe('HttpUserRepository', () => {
  const post = vi.fn()
  const httpClient = { post } as unknown as AxiosHttpClient
  const repository = new HttpUserRepository(httpClient)

  it('maps the raw response to a domain entity', async () => {
    post.mockResolvedValue({
      data: { id: 'user-1', name: 'Tony Hawk', email: 'tony@hawk.cl', created_at: '2026-01-15', updated_at: '2026-01-15' },
      status: 201,
      headers: {},
    })

    const result = await repository.save(new CreateUserProps('Tony Hawk', 'tony@hawk.cl'))

    expect(result.isOk()).toBe(true)
    expect(result.getValue().id).toBe('user-1')
  })

  it('converts a known status into the specific domain exception', async () => {
    post.mockRejectedValue(new HttpServiceException(409, 'Conflict'))

    const result = await repository.save(new CreateUserProps('Tony Hawk', 'tony@hawk.cl'))

    expect(result.isErr()).toBe(true)
    expect(result.getError().code).toBe('USER_ALREADY_EXISTS')
  })

  it('falls back to the HttpServiceException for any other failure', async () => {
    post.mockRejectedValue(new HttpServiceException(500, 'Internal error'))

    const result = await repository.save(new CreateUserProps('Tony Hawk', 'tony@hawk.cl'))

    expect(result.getError().code).toBe('HTTP_SERVICE_ERROR')
  })
})
```

**4. ViewModel** — see *Testing the ViewModel* in the MVVM Convention of
`frontend-spa-vue.md`. The pattern is always: override the container registrations with
`asValue(mock)`, run the composable through the `withSetup` helper (a fresh app with Pinia and
Pinia Colada per test, so the cache starts empty), `await flushPromises()`, and assert **only
on what the ViewModel returns** (the View's contract) — never on internals or the cache, and
never mocking axios or `@pinia/colada` directly.

## Code Quality

The project uses `husky` + `lint-staged` for pre-commit hooks that run linting and formatting automatically. See `frontend-spa-vue.md` for the exact configuration.

## Dependency Rule (enforced via `eslint-plugin-boundaries`)

- Modules **never** import from other modules
- If something is needed in more than one module → move it to `modules/shared/`
- `domain/` has zero external dependencies
- `application/` depends only on `domain/`
- `infrastructure/` depends on `application/` and `domain/`
- `presentation/` depends on `application/` and `domain/`

These rules are **enforced at lint time** using `eslint-plugin-boundaries`, installed as a dev dependency and configured with one boundary element per layer, `default: 'disallow'`, and explicit `allow` rules matching the list above. Violations fail lint and therefore fail the `lint-staged` pre-commit hook.

See *Boundary Enforcement* in `frontend-spa-vue.md` for the ready-to-copy ESLint configuration.

## src/base Structure

The project has a `src/base/` folder with technical cross-cutting concerns:

```
src/base/
├── config/          ← technical infrastructure (env, http, auth, logger, di, colada, etc.)
├── constants/
│   └── index.ts     ← project-wide constants (export const VARIABLE_NAME = value)
└── lib/             ← abstract base classes & generic technical machinery
    ├── domain/         ← result.ts, command.base.ts, query.base.ts, props.base.ts, domain-exception.base.ts, value-object.base.ts, paginated.ts, criteria/ (Criteria pattern VOs, when used)
    ├── application/    ← use-case.base.ts
    └── utils/          ← pure utility functions grouped by concern
```

> **Criteria pattern placement.** Dynamic filter + order + pagination is *technical
> machinery*, not business: its domain value objects live in `src/base/lib/domain/criteria/`
> and the query-string converter in `src/base/lib/infrastructure/criteria/` — never in a module
> or in `modules/shared/`. Repositories then expose a single `matching(criteria)` method
> instead of many `findByX`. For the full pattern, see the `criteria-pattern` skill.

## Environment Variables Convention

The project follows these rules for environment variables:

**Who creates each file:**
- `.env.example` — **created by the agent** when scaffolding the project. Contains all required variables with empty or placeholder values. **Committed to the repo.**
- `.env` / `.env.local` — **created by the developer** with real values. **Never committed** (added to `.gitignore`).
- `.env.development` / `.env.production` — optional, created by the developer per environment.

**Variable prefix** (enforced by Vite, not optional): every variable the app reads must be
prefixed **`VITE_`** — only `VITE_*` variables are exposed to the browser via `import.meta.env`.
Anything without the prefix is silently `undefined` at runtime.

> Everything in `.env` ends up in the browser bundle. Never put a secret (API secret, private
> key, service credential) in a `VITE_` variable — an SPA has no server-only environment.

**Access pattern**: never read `import.meta.env` directly outside of `src/base/config/env/`. All code that needs an env var imports from the centralized env config. This means:
- One place to validate all env vars at startup
- One place to fix a missing var
- Type-safe access everywhere else

See the *Environment Variables* section of `frontend-spa-vue.md` for the full implementation (zod schema, `env.config.ts`, `.env.example`).

## Constants Convention

All project-wide constants live in `src/base/constants/index.ts`. Use `UPPER_SNAKE_CASE` for names:
```typescript
// src/base/constants/index.ts
export const API_BASE_URL = 'https://api.example.com'
export const MAX_RETRY_ATTEMPTS = 3
export const DEFAULT_PAGE_SIZE = 20
```

## Utils Convention

Utility functions live in `src/base/lib/utils/`, grouped by concern — one file per topic, multiple functions per file:
```
src/base/lib/utils/
├── index.ts              ← re-exports all utils for clean imports
├── date.utils.ts         ← formatDate, parseDate, diffInDays, ...
├── string.utils.ts       ← capitalize, slugify, truncate, ...
├── number.utils.ts       ← formatCurrency, roundTo, clamp, ...
├── array.utils.ts        ← chunk, unique, groupBy, ...
├── object.utils.ts       ← deepClone, pick, omit, ...
└── validation.utils.ts   ← isEmail, isUrl, isEmpty, ...
```

Rules:
- All functions must be **pure** — no external dependencies, no side effects
- File name convention: `<concern>.utils.ts`
- If a file grows beyond ~30 functions, split by sub-concern (e.g. `date-format.utils.ts`, `date-parse.utils.ts`)
- `index.ts` re-exports everything so consumers import from one place: `import { formatDate, slugify } from '@/base/lib/utils'`

## Quick Reference: Use Case File Structure

Every use case lives in its own folder. The input class (Props) lives in `domain/props/`:

```
src/modules/user/domain/props/
├── create-user.props.ts          ← CreateUserProps class
└── find-users.props.ts           ← FindUsersProps class

src/modules/user/application/use-cases/user-creator/
└── user-creator.use-case.ts      ← receives CreateUserProps, returns Result<User>

src/modules/user/application/use-cases/users-finder/
└── users-finder.use-case.ts      ← receives FindUsersProps, returns Result<Paginated<User>>
```

There is **no** `Output` class and **no** use-case mapper in a Vue SPA: the use case returns
domain data, and the **Screen Mapper** (`presentation/mappers/`) turns it into Presentation
Models. That pair is the frontend equivalent of the backend's Output + Mapper.

**Tests:**
```
tests/modules/user/application/use-cases/user-creator/
└── user-creator.use-case.spec.ts

tests/modules/user/presentation/mappers/
└── user-profile-screen.mapper.spec.ts
```
