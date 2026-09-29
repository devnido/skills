# Universal Conventions — Hexagonal Architecture

These conventions apply to **ALL project types** (NestJS monolith/microservice, React SPA,
React Native, Next.js; Vue SPAs are covered by the `vuejs-hexagonal` skill). Read this file together with the project-type reference
before generating any structure. `SKILL.md` carries only a digest of these rules — **this
file is the canonical source for the full rules and the code templates.** Generate every
template below exactly as shown — do not rename classes, generics, or methods.

## Table of Contents

1. [Result Pattern](#result-pattern)
2. [Domain Base Classes](#domain-base-classes)
3. [Paginated Container](#paginated-container)
4. [Domain Entity Rules](#domain-entity-rules)
5. [Value Object Convention](#value-object-convention)
6. [Port Parameters Convention](#port-parameters-convention)
7. [UseCase Base Class](#usecase-base-class)
8. [Use Case Output & Mapper (backend)](#use-case-output--mapper-convention-backend-only)
9. [Form Model & Form Mapper (frontend)](#form-model--form-mapper-convention-frontend-only)
10. [Screen Mapper & Presentation Model (frontend)](#screen-mapper--presentation-model-convention-frontend-only)
11. [Driving Vertical Slice (backend)](#driving-vertical-slice-convention-backend-only)
12. [Response Wrapper Convention (backend)](#response-wrapper-convention-backend-only)
13. [Auth & Session Convention (frontend)](#auth--session-convention-frontend-only)
14. [Testing Conventions](#testing-conventions)
15. [Code Quality](#code-quality)
16. [Dependency Rule](#dependency-rule-enforced-via-eslint-plugin-boundaries)
17. [src/base Structure](#srcbase-structure)
18. [Environment Variables Convention](#environment-variables-convention)
19. [Constants Convention](#constants-convention)
20. [Utils Convention](#utils-convention)
21. [Quick Reference: Use Case File Structure](#quick-reference-use-case-file-structure)

---

## Result Pattern

> **Scope: frontend only.** The `Result<T>` pattern below is the **frontend** error channel
> (SPAs, React Native, Next.js). The **backend (NestJS) does not use `Result`** — its domain
> and application layers **throw `DomainException` directly**, and a global
> `DomainExceptionFilter` maps the thrown exception to the HTTP response. There is no
> `Result` type, no `result.ts`, and no Result-unwrapping interceptor in the backend. See the
> backend reference for the throw/return style.

Each **frontend** project includes a manual `Result<T>` class in `src/base/lib/domain/result.ts`. The error type is **always `DomainException`** — it is not a generic parameter:

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

Rules (frontend):
- Use `Result.ok(value)` and `Result.err(error)` in the **domain** and **application** layers of every **frontend** project
- The error side is always `DomainException` — any concrete exception (e.g. `UserNotFoundException`) works because it `extends DomainException`
- Infrastructure adapters `try/catch` the HTTP client and convert **every** failure into `Result.err(...)` (mapping known statuses to domain exceptions; the `HttpServiceException` itself — it extends `DomainException` — is the generic fallback). Use cases and ViewModels never see exceptions: the **ViewModel** is the frontend's driving boundary — it unwraps the `Result` with `isErr()` and maps the `DomainException`'s `code` to UI error state. It never re-throws, and `try/catch` is not its error channel (see the MVVM Convention in each SPA reference). In **Next.js** the unwrap point is the Server Component for reads (`notFound()` or throw — failing the build on SSG pages) and the Server Action for mutations (return a serializable `{ error }`); see the Next.js reference.
- In frontend **domain** and **application**, never throw domain errors — always return `Result.err(new UserNotFoundException(id))`. The frontend never throws a `DomainException`.

Backend error handling (no `Result`): domain and application **throw** the `DomainException`
directly (`throw new UserNotFoundException(id)`) and return plain values; the global
`DomainExceptionFilter` is the only domain-error-to-HTTP mechanism.

Backend **adapters** wrap every technical operation (DB query, HTTP request, queue publish,
S3 call, …) in a `try/catch`. On failure they **log** the error (stack + operation name) and
**throw an infrastructure `DomainException` tied to that adapter** — `DatabaseErrorException`,
`DownstreamServiceErrorException`, `MessagingErrorException`, etc. (`httpStatus 500`) — so the
raw technical error never escapes the adapter. Business invariants (not found, duplicate) are
thrown from the *successful* result, not from the `catch`. See the backend reference for the
canonical adapter example.

## Domain Base Classes

Every project includes these abstract base classes in `src/base/lib/domain/`. **Generate each file exactly as shown — do not rename classes:**

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

// src/base/lib/domain/output.base.ts (backend only — base for use-case Outputs)
export abstract class Output {
  private readonly _brand!: 'Output'
}

// src/base/lib/domain/event.base.ts (backend only)
export interface EventMetadata {
  eventId: string
  occurredAt: Date
}

export abstract class Event implements EventMetadata {
  readonly eventId: string = crypto.randomUUID()
  readonly occurredAt: Date = new Date()
}
```

Why these shapes:
- The `_brand` field is a definite-assignment private member: it emits no JavaScript, but it makes
  each base class **nominal**. Without it, TypeScript's structural typing lets any object satisfy
  an empty class, so `UseCase<I extends Input>` would accept anything. Private members are compared
  nominally, so `Command`, `Query`, `Props`, and `Output` stay mutually incompatible.
- `DomainException` extends `Error` so every domain error carries a real `message` and stack trace
  and can be safely thrown at the controller boundary. Each concrete exception declares a stable
  `code` (UPPER_SNAKE_CASE) that the global exception filter uses to map to an `HttpException` —
  never map by class name (`constructor.name` breaks under minification).

All Commands, Queries, Props, Events, domain errors, and use-case Outputs must extend these base classes:
```typescript
// domain/props/create-user.command.ts (backend)
import { Command } from '@/base/lib/domain/command.base'
export class CreateUserCommand extends Command { ... }

// domain/props/create-user.props.ts (frontend)
import { Props } from '@/base/lib/domain/props.base'
export class CreateUserProps extends Props { ... }

// domain/exceptions/user-not-found.exception.ts
import { DomainException } from '@/base/lib/domain/domain-exception.base'
export class UserNotFoundException extends DomainException {
  readonly code = 'USER_NOT_FOUND'
  constructor(id: string) { super(`User ${id} not found`) }
}

// domain/events/user-created.event.ts (backend only)
import { Event } from '@/base/lib/domain/event.base'
export class UserCreatedEvent extends Event {
  constructor(public readonly userId: string, public readonly email: string) { super() }
}

// application/use-cases/spots-finder/spots-finder.output.ts (backend only)
import { Output } from '@/base/lib/domain/output.base'
export class SpotsFinderOutput extends Output { ... }
```

## Paginated Container

Every project includes a generic `Paginated<T>` class in `src/base/lib/domain/paginated.ts`.
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
- A use case returning a collection returns `Paginated<Output>` (backend) or `Paginated<Entity>`
  (frontend), never a raw array.
- **No `totalPages` field.** `totalPages` is a derived presentation value computed at the
  infrastructure boundary (the controller, when wrapping in `PaginatedResponse`), not in the
  application layer.
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

Port functions (repository methods, service methods) receive their parameters as domain classes. There are 3 cases.

> **Return types below show the frontend form (`Result<T>`).** In the **backend**, ports return
> the value directly (`Promise<User>`, `Promise<void>` for publishers) and **throw** a
> `DomainException` on the error path — there is no `Result` wrapper.

**Case 1 — Same params as the use case:** reuse the Input directly (Command/Query/Props)
```typescript
// The port receives the same Command the use case received
export interface UserRepository {
  save(command: CreateUserCommand): Promise<Result<User>>
}
```

**Case 2 — Publishing an event (backend only):** create an Event class in `domain/events/`
```typescript
// The port publishes an event to a message queue
export interface EventBus {
  publish(event: UserCreatedEvent): Promise<Result<void>>
}
```

**Case 3 — Different params:** create a `<FunctionName>Props` class in `domain/props/`
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

Every project includes an abstract `UseCase` class in `src/base/lib/application/use-case.base.ts`. **Generate this file exactly as shown — do not rename the class, generics, or method:**

```typescript
// Frontend: src/base/lib/application/use-case.base.ts
import type { Result } from '../domain/result'
import type { Command } from '../domain/command.base'
import type { Query } from '../domain/query.base'
import type { Props } from '../domain/props.base'

type Input = Command | Query | Props

export abstract class UseCase<I extends Input, O> {
  abstract execute(input: I): Result<O> | Promise<Result<O>>
}
```

In the **backend** the base does not import `Result` — `execute` returns the value directly
and throws on the error path:
```typescript
// Backend: src/base/lib/application/use-case.base.ts
export abstract class UseCase<I extends Input, O> {
  abstract execute(input: I): O | Promise<O>
}
```

All use cases **extend** this class (never `implements`). **Backend use cases return an
`Output` (or `Paginated<Output>`) — never a domain entity or an infra DTO.** Frontend use cases
return domain data (the presentation layer adapts it via Screen Mapper / Presentation Model).
```typescript
// Backend — single result (returns the Output directly; throws a DomainException on error)
export class SpotCreator extends UseCase<CreateSpotCommand, SpotCreatorOutput> {
  execute(command: CreateSpotCommand): Promise<SpotCreatorOutput> {
    // ...
  }
}

// Backend — paginated collection
export class SpotsFinder extends UseCase<FindSpotsQuery, Paginated<SpotsFinderOutput>> {
  execute(query: FindSpotsQuery): Promise<Paginated<SpotsFinderOutput>> {
    // ...
  }
}

// Frontend — returns domain data wrapped in a Result
export class UserCreator extends UseCase<CreateUserProps, User> {
  execute(props: CreateUserProps): Promise<Result<User>> {
    // ...
  }
}
```

**Backend DI:** a use case implements no interface, so it needs **no DI token** — register it as a
plain provider and inject it by class (`constructor(private readonly spotsFinder: SpotsFinder)`).
DI tokens are reserved for **ports** (interfaces) bound to an adapter — repositories, gateways,
event buses — e.g. `{ provide: SPOT_REPOSITORY, useClass: MongoSpotRepository }`.

## Use Case Output & Mapper Convention (backend only)

A backend use case never returns a domain entity or an infrastructure DTO. It returns a
dedicated **Output** class, and a colocated **Mapper** turns the port's result (usually a
domain entity) into that Output. This keeps the application layer self-contained: it owns its
own return contract and never reaches into `infrastructure/`.

**Folder layout** — every use case folder holds three things:
```
application/use-cases/spots-finder/
├── spots-finder.use-case.ts      ← SpotsFinder (orchestration)
├── spots-finder.output.ts        ← SpotsFinderOutput extends Output (return contract)
└── mapper/
    └── spots-finder.mapper.ts     ← SpotsFinderMapper (domain entity → Output)
```

**1. Output** (`<use-case>.output.ts`, class `<UseCase>Output extends Output`):
- A curated selection of the fields the API actually exposes — it strips heavy arrays,
  internal flags, and anything sensitive.
- Dates are serialized to ISO strings here (the Output is what the controller hands to the
  response wrapper, so it carries the final shape).
- May reuse domain value-object types (e.g. `Location`) since `application → domain` is allowed.

```typescript
// application/use-cases/spots-finder/spots-finder.output.ts
import { Output } from '@/base/lib/domain/output.base'
import { type Location } from '@/modules/shared/domain/value-objects/location'

export class SpotsFinderOutput extends Output {
  readonly id: string
  readonly name: string
  readonly location: Location
  readonly totalLikes: number
  readonly createdAt: string // ISO 8601
  // …only the fields the API exposes
  constructor(props: SpotsFinderOutputProps) {
    super()
    // assign each field from props
  }
}
```

**2. Mapper** (`mapper/<use-case>.mapper.ts`, static class `<UseCase>Mapper`):
- One static method, `toOutput(entity): <UseCase>Output`, mapping the port's domain entity to
  the Output. (Add `toOutputList` only if a caller needs it; usually `items.map(Mapper.toOutput)`
  is enough.)
- The method name stays terse (`toOutput`) — the class name already names the use case.

```typescript
// application/use-cases/spots-finder/mapper/spots-finder.mapper.ts
import { type Spot } from '@/modules/spot/domain/entities/spot.entity'
import { SpotsFinderOutput } from '../spots-finder.output'

export class SpotsFinderMapper {
  static toOutput(spot: Spot): SpotsFinderOutput {
    return new SpotsFinderOutput({
      id: spot.id,
      name: spot.name,
      location: { ...spot.location },
      totalLikes: spot.totalLikes,
      createdAt: spot.createdAt.toISOString(),
      // …
    })
  }
}
```

**3. Use case** orchestrates only — call the port, map entities through the Mapper, wrap a
collection in `Paginated`. No field-by-field assembly, no `totalPages`, no infra imports:

```typescript
// application/use-cases/spots-finder/spots-finder.use-case.ts
export class SpotsFinder extends UseCase<FindSpotsQuery, Paginated<SpotsFinderOutput>> {
  async execute(query: FindSpotsQuery): Promise<Paginated<SpotsFinderOutput>> {
    const { items, total } = await this.spotRepository.matching(/* criteria */)
    return new Paginated(
      items.map(SpotsFinderMapper.toOutput),
      total,
      query.pageNumber,
      query.pageSize,
    )
  }
}
```

**4. Controller** wraps the Output directly (no separate response DTO) and derives `totalPages`
at the boundary. Any `DomainException` thrown by the use case propagates to the global
`DomainExceptionFilter`:
```typescript
async find(@Query() dto: FindSpotsQueryDto): Promise<PaginatedResponse<SpotsFinderOutput>> {
  const { items, total, page, limit } = await this.spotsFinder.execute(
    FindSpotsHttpMapper.toQuery(dto),
  )
  const totalPages = total === 0 ? 0 : Math.ceil(total / limit)
  return new PaginatedResponse(items, { page, limit, total, totalPages })
}
```

> A single-result use case returns the Output directly (`UseCase<CreateSpotCommand, SpotCreatorOutput>`)
> and its controller wraps it in `SingleResponse<SpotCreatorOutput>`.

## Form Model & Form Mapper Convention (frontend only)

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

The ViewModel (SPAs) or Server Action (Next.js) uses the mapper:
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

## Screen Mapper & Presentation Model Convention (frontend only)

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

## Driving Vertical Slice Convention (backend only)

`infrastructure/driving/` is organized first by **protocol** (`http/`, `rpc/`, `messaging/`, `cli/`) and inside each protocol by **action folder** — one folder per endpoint / consumer / handler. Each action folder contains everything that endpoint needs, colocated:

```
infrastructure/
├── driven/
│   ├── persistence/
│   │   ├── mongo-user.repository.ts
│   │   └── schemas/
│   │       └── user.schema.ts
│   └── messaging/
│       └── publishers/
│           └── rabbit-event-bus.publisher.ts
└── driving/
    ├── http/
    │   ├── create-user/
    │   │   ├── dto/
    │   │   │   └── create-user.request.dto.ts
    │   │   ├── mapper/
    │   │   │   └── create-user.mapper.ts
    │   │   └── create-user.http.controller.ts
    │   └── find-all-users/
    │       ├── dto/
    │       │   └── find-all-users.request.dto.ts   ← pagination/filter query params
    │       ├── mapper/
    │       │   └── find-all-users.mapper.ts         ← request DTO → Query only
    │       └── find-all-users.http.controller.ts
    └── rpc/
        └── product-created/
            ├── dto/
            │   └── product-created.request.dto.ts
            ├── mapper/
            │   └── product-created.mapper.ts
            └── product-created.rpc.controller.ts
```

Rules:
- **One folder per action.** Folder name in kebab-case matches the action (e.g. `create-user/`, `find-all-users/`, `product-created/`).
- **`dto/` is singular** — contains the **request** side only: `<action>.request.dto.ts` (the inbound HTTP/RPC shape). There is **no response DTO**: the use-case **Output** is the response contract (see *Use Case Output & Mapper Convention*). Omit `dto/` when the action takes no input.
- **`mapper/` is singular** — contains `<action>.mapper.ts`, mapping the **request DTO → Command/Query** only. Omit the folder if the action takes no input. The class is `<Action>Mapper`. (Distinct from the application-layer use-case `<UseCase>Mapper`, which maps domain entity → Output.)
- **Controller file** is `<action>.<protocol>.controller.ts`. The class is `<Action><Protocol>Controller` (e.g. `CreateUserHttpController`, `ProductCreatedRpcController`).
- The controller is the **only** thing that knows about the protocol. The mapper, DTOs, and use case are protocol-agnostic in shape (the DTO names happen to live next to the protocol because they are protocol-shaped).

**Mapper method naming** — the driving mapper handles the **request side only** (response
shaping lives in the application use-case Mapper → Output, see *Use Case Output & Mapper
Convention*):
- `<Action>RequestDtoTo<Action><Command|Query>` — request DTO → application input

```typescript
// infrastructure/driving/http/create-user/mapper/create-user.mapper.ts
import { CreateUserCommand } from '@/modules/user/domain/props/create-user.command'
import type { CreateUserRequestDto } from '../dto/create-user.request.dto'

export class CreateUserMapper {
  static createUserRequestDtoToCreateUserCommand(dto: CreateUserRequestDto): CreateUserCommand {
    return new CreateUserCommand(dto.name, dto.email)
  }
}
```

## Response Wrapper Convention (backend only)

Every controller response is wrapped in one of two generic envelopes defined in `src/base/lib/infrastructure/`:

```typescript
// src/base/lib/infrastructure/single-response.ts
export class SingleResponse<T> {
  constructor(public readonly data: T) {}
}

// src/base/lib/infrastructure/paginated-response.ts
export interface PaginationMeta {
  page: number
  limit: number
  total: number
  totalPages: number
}

export class PaginatedResponse<T> {
  constructor(
    public readonly data: T[],
    public readonly pagination: PaginationMeta,
  ) {}
}
```

Rules:
- Single-record endpoints return `SingleResponse<<UseCase>Output>`.
- List/paginated endpoints return `PaginatedResponse<<UseCase>Output>`, built from the use
  case's `Paginated<Output>`.
- The controller **derives `totalPages` at the boundary** (`total === 0 ? 0 : Math.ceil(total / limit)`) —
  the application `Paginated<T>` carries only `items / total / page / limit`.
- **Never** return a raw entity, raw array, or naked DTO from a controller.
- The controller returns the **wrapper instance** (`new SingleResponse(...)` /
  `new PaginatedResponse(...)`) **directly**. The use case returns the Output (no `Result`);
  any `DomainException` it throws is handled by the global `DomainExceptionFilter`, which maps
  its `code` to the proper `HttpException` (see the backend reference). A global metadata
  interceptor enriches these wrapper instances on the way out.
- The **Output** (built by the use-case Mapper) already includes only the fields the API
  exposes — internal, derived, and sensitive fields are stripped there, not in the controller.

## Auth & Session Convention (frontend only)

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

| Platform | Adapter | Storage |
|---|---|---|
| React SPA | `local-storage.token-storage.ts` | `localStorage` (accepted XSS trade-off; prefer in-memory access token + refresh cookie if the API supports it) |
| React Native | `secure-store.token-storage.ts` | `expo-secure-store` |
| Next.js | — (no client adapter) | **httpOnly cookies** set by a Server Action / Route Handler; middleware reads them (see the Next.js reference) |

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
  store (SPAs and React Native). Next.js guards with middleware instead (see its reference).
- Tokens never live in the store — the store holds *who is logged in*, the storage holds
  *the credentials*. Components never see tokens.

### 4. Build-time / server-side auth (Next.js)

- SSG pages authenticate at build time with a **service/guest token** obtained at build
  start using credentials from env vars (e.g. a fixed device UUID) — never a real user's
  token, and never `NEXT_PUBLIC_`-prefixed.
- Request-time rendering uses the request-scoped container seeded from the httpOnly
  cookie (`createRequestContainer(token)` — see the Next.js reference).
- Client Components never see tokens; they mutate through Server Actions.

## Testing Conventions

| Layer | Test Type | Mock Strategy |
|---|---|---|
| `domain/` | Unit tests — pure, no mocks | None |
| `application/` | Unit tests | Mock ports (interfaces) |
| `infrastructure/` | Unit tests | Mock HTTP clients, ORM clients, etc. |
| `presentation/` | Component tests | Mock view models / composables |

Testing frameworks per project type:
| Project | Framework | Component Testing |
|---|---|---|
| NestJS | Jest (included with NestJS) | — |
| React SPA | Vitest | `@testing-library/react` |
| React Native | Jest (included with Expo) | `@testing-library/react-native` |
| Next.js | Vitest | `@testing-library/react` |

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

Generate tests from these templates — do not improvise the mocking strategy. The examples
use Vitest (`vi.*`); in Jest projects (NestJS, React Native) replace `vi` with `jest` —
the API is otherwise identical.

**1. Backend use case — mock the ports, assert on the returned Output (and the thrown error):**

```typescript
// tests/modules/user/application/use-cases/user-creator/user-creator.use-case.spec.ts
import { UserCreator } from '@/modules/user/application/use-cases/user-creator/user-creator.use-case'
import { CreateUserCommand } from '@/modules/user/domain/props/create-user.command'
import type { UserRepository } from '@/modules/user/domain/ports/user.repository'
import { UserAlreadyExistsException } from '@/modules/user/domain/exceptions/user-already-exists.exception'
import { buildUser } from '../../../builders/user.builder'

describe('UserCreator', () => {
  const save = vi.fn()
  const userRepository: UserRepository = { save }
  const useCase = new UserCreator(userRepository)

  it('returns the Output when the repository saves', async () => {
    save.mockResolvedValue(buildUser({ id: 'user-1' }))

    const output = await useCase.execute(new CreateUserCommand('Tony', 'tony@hawk.cl'))

    expect(output.id).toBe('user-1') // asserts on the Output, never the entity
  })

  it('propagates the domain error when the repository throws', async () => {
    save.mockRejectedValue(new UserAlreadyExistsException('tony@hawk.cl'))

    await expect(
      useCase.execute(new CreateUserCommand('Tony', 'tony@hawk.cl')),
    ).rejects.toMatchObject({ code: 'USER_ALREADY_EXISTS' })
  })
})
```

Rules: mock **only the ports** (never the use-case Mapper or the Output); assert errors by
`code`, never by exception class name; never touch the network or a DB. (On the **frontend**,
ports resolve a `Result` and the use case returns a `Result` — assert with `isOk()`/`isErr()`
instead; see the SPA references.)

**2. Use-case Mapper — pure, no mocks:**

```typescript
// tests/modules/user/application/use-cases/user-creator/mapper/user-creator.mapper.spec.ts
import { UserCreatorMapper } from '@/modules/user/application/use-cases/user-creator/mapper/user-creator.mapper'
import { buildUser } from '../../../../builders/user.builder'

describe('UserCreatorMapper', () => {
  it('maps the entity to the Output with ISO dates', () => {
    const user = buildUser({ createdAt: new Date('2026-01-15T10:00:00Z') })

    const output = UserCreatorMapper.toOutput(user)

    expect(output.id).toBe(user.id)
    expect(output.createdAt).toBe('2026-01-15T10:00:00.000Z')
  })

  it('exposes only the API fields', () => {
    const output = UserCreatorMapper.toOutput(buildUser())

    expect(output).not.toHaveProperty('password') // internal/sensitive fields stripped
  })
})
```

**3. Adapter — mock the technical client, assert the error conversion** (frontend shown — it
converts failures into `Result.err`; a **backend** adapter instead **throws** a
`DomainException` / `DownstreamServiceErrorException`, so its test asserts with `rejects`):

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

**4. ViewModel (frontend)** — see *Testing the ViewModel* inside each SPA reference's MVVM
Convention (it needs the framework's hook/composable testing utility). The pattern is
always: override the container registrations with `asValue(mock)`, drive the hook, and
assert **only on what the ViewModel returns** (the View's contract) — never on internals,
and never mocking axios/fetch directly.

## Code Quality

All projects use `husky` + `lint-staged` for pre-commit hooks that run linting and formatting automatically.

## Dependency Rule (enforced via `eslint-plugin-boundaries`)

- Modules **never** import from other modules
- If something is needed in more than one module → move it to `shared/` (monolith/frontend)
- `domain/` has zero external dependencies
- `application/` depends only on `domain/`
- `infrastructure/` depends on `application/` and `domain/`
- `presentation/` depends on `application/` and `domain/`

These rules are **enforced at lint time** using `eslint-plugin-boundaries`. Every project installs it as a dev dependency and configures ESLint to define boundary elements (one per layer) with `default: 'disallow'` and explicit `allow` rules matching the list above. Violations fail lint and therefore fail the `lint-staged` pre-commit hook.

See each project-type reference for the project-specific ESLint configuration — backend and frontend differ in layer count and folder structure.

## src/base Structure

Every project has a `src/base/` folder with technical cross-cutting concerns:

```
src/base/
├── config/          ← technical infrastructure (env, http, auth, logger, di, etc.)
├── constants/
│   └── index.ts     ← project-wide constants (export const VARIABLE_NAME = value)
└── lib/             ← abstract base classes & generic technical machinery
    ├── domain/         ← result.ts, command.base.ts, query.base.ts, props.base.ts, domain-exception.base.ts, event.base.ts, value-object.base.ts, output.base.ts (backend), paginated.ts, criteria/ (Criteria pattern VOs, when used)
    ├── application/    ← use-case.base.ts
    ├── infrastructure/ ← single-response.ts, paginated-response.ts, criteria/ (Criteria→DB converter) (backend only)
    └── utils/          ← pure utility functions grouped by concern
```

> **Criteria pattern placement.** Dynamic filter + order + pagination is *technical
> machinery*, not business: its domain value objects live in `src/base/lib/domain/criteria/`
> and the DB-specific converter in `src/base/lib/infrastructure/criteria/` — never in a module
> or in `modules/shared/`. Repositories then expose a single `matching(criteria)` method
> instead of many `findByX`. For the full pattern, see the `criteria-pattern` skill.

Backend also includes:
```
src/base/
├── health/          ← healthcheck controller and module
└── config/
    ├── context/     ← correlation ID, request context
    ├── messaging/   ← message broker client (RabbitMQ by default)
    └── nestjs/      ← global exception filter, domain exception filter, logger interceptor
```

## Environment Variables Convention

Every project follows these rules for environment variables:

**Who creates each file:**
- `.env.example` — **created by the agent** when scaffolding the project. Contains all required variables with empty or placeholder values. **Committed to the repo.**
- `.env` / `.env.local` — **created by the developer** with real values. **Never committed** (added to `.gitignore`).
- `.env.development` / `.env.production` — optional, created by the developer per environment.

**Variable prefix by project type** (enforced by the platform, not optional):
- **NestJS backend**: no prefix — accessed via `process.env.VAR_NAME`, read through `EnvVarsService`
- **Vite (React)**: `VITE_` prefix — only `VITE_*` variables are exposed to the browser via `import.meta.env`
- **Next.js**: `NEXT_PUBLIC_` prefix for client-accessible variables; no prefix for server-only variables
- **Expo (React Native)**: `EXPO_PUBLIC_` prefix — only `EXPO_PUBLIC_*` variables are bundled into the app

**Access pattern**: never read `process.env` or `import.meta.env` directly outside of `src/base/config/env/`. All code that needs an env var imports from the centralized env config. This means:
- One place to validate all env vars at startup
- One place to fix a missing var
- Type-safe access everywhere else

See each project-type reference for the full implementation (schema, service/config, `.env.example`).

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

Every use case lives in its own folder. The input class (Props, Command, or Query) lives in `domain/props/`:

**Frontend:**
```
src/.../domain/props/
└── create-user.props.ts          ← CreateUserProps class

src/.../application/use-cases/user-creator/
└── user-creator.use-case.ts      ← receives CreateUserProps
```

**Backend:** the use-case folder holds the use case, its Output, and its Mapper:
```
src/.../domain/props/
├── create-user.command.ts        ← CreateUserCommand class (write operations)
└── find-user.query.ts            ← FindUserQuery class (read operations)

src/.../application/use-cases/user-creator/
├── user-creator.use-case.ts      ← receives CreateUserCommand, returns UserCreatorOutput
├── user-creator.output.ts        ← UserCreatorOutput extends Output
└── mapper/
    └── user-creator.mapper.ts     ← UserCreatorMapper (domain entity → Output)

src/.../application/use-cases/user-finder/
├── user-finder.use-case.ts       ← receives FindUserQuery, returns Paginated<UserFinderOutput>
├── user-finder.output.ts         ← UserFinderOutput extends Output
└── mapper/
    └── user-finder.mapper.ts      ← UserFinderMapper (domain entity → Output)
```

**Tests:**
```
tests/.../application/use-cases/user-creator/
├── user-creator.use-case.spec.ts
└── mapper/
    └── user-creator.mapper.spec.ts
```
