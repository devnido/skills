---
name: hexagonal-architecture
description: >
  Enforces hexagonal architecture (Ports & Adapters) with vertical slicing for all
  TypeScript projects: NestJS monoliths, NestJS microservices, Vue.js SPAs, React SPAs,
  React Native apps, and Next.js apps. Use this skill whenever the user creates a project,
  adds a module, scaffolds a feature, generates a use case, creates a repository, adds a
  controller, creates a component, adds a screen, or does anything involving file/folder
  structure. Also trigger when the user asks about architecture, folder structure, naming
  conventions, how to organize code, where to place a file, or how layers connect. Even
  if the user doesn't explicitly mention "hexagonal" or "architecture", use this skill
  any time the task involves generating, moving, or restructuring TypeScript project files.
---

# Hexagonal Architecture Skill

This skill enforces a consistent hexagonal architecture (Ports & Adapters) across all
TypeScript projects. Before generating any file or folder, read the appropriate reference
file for the project type.

## Project Type Detection

Identify the project type from context (package.json, existing structure, or user description):

| Type | Indicators | Reference File |
|---|---|---|
| NestJS Monolith | NestJS + multiple domains/modules | `references/backend-nestjs.md` |
| NestJS Microservice | NestJS + single domain | `references/backend-nestjs.md` |
| Vue.js SPA | Vue 3 + Vue Router | `references/frontend-spa-vue.md` |
| React SPA | React + React Router (no Next.js) | `references/frontend-spa-react.md` |
| React Native | Expo + React Navigation | `references/frontend-mobile-react-native.md` |
| Next.js | Next.js + App Router | `references/frontend-nextjs.md` |

**Always read the reference file before generating any structure.**

> If you cannot determine the project type from `package.json`, folder structure, or user
> description, **ask the user before generating anything**. Never guess the project type.

---

## Universal Rules (apply to ALL project types)

### Layers
Every project has exactly 3 layers (backend) or 4 layers (frontend):
1. **domain/** — entities, value objects, ports (interfaces), domain errors, domain events
2. **application/** — use cases, DTOs, application errors
3. **infrastructure/** — adapters, clients, models, schemas, controllers
4. **presentation/** — screens, view models, components, routes, store *(frontend only)*

**Backend infrastructure is split into `driving/` and `driven/`:**
- `infrastructure/driving/` — entry points that **invoke** the application (HTTP controllers, RPC consumers, message consumers, CLI handlers, cron jobs). Organized by **protocol** then **vertical slice per action**.
- `infrastructure/driven/` — outbound adapters that the application **invokes** (repositories, message publishers, HTTP clients to other services, email senders, etc.). Organized by technical concern (`persistence/`, `messaging/`, `http/`, `email/`).

This split makes the hexagonal direction explicit: driving adapters push into the hexagon, driven adapters are pushed by the hexagon.

### Naming Conventions — Classes & Interfaces
| Concept | Convention | Example |
|---|---|---|
| Port (interface) | `<Entity><Role>` | `UserRepository` |
| Adapter (impl) | `<Infra><Entity><Role>` | `MongoUserRepository`, `HttpUserRepository` |
| Use case | `<Entity><Action>` | `UserCreator`, `UserFinder`, `UserUpdater` |
| Use case folder | `<entity>-<action>/` | `user-creator/` |
| Use case Output (backend) | `<UseCase>Output` | `SpotsFinderOutput`, `SpotCreatorOutput` |
| Use case Mapper (backend) | `<UseCase>Mapper` | `SpotsFinderMapper`, `SpotCreatorMapper` |
| Paginated container (domain) | `Paginated<T>` | `Paginated<SpotsFinderOutput>` |
| Domain exception | `<Entity><Reason>Exception` | `UserNotFoundException`, `UserAlreadyExistsException` |
| HTTP exception (backend) | `DownstreamServiceErrorException` | `DownstreamServiceErrorException` |
| HTTP exception (frontend) | `HttpServiceException` | `HttpServiceException` |
| Props (frontend) | `<Action><Entity>Props` | `CreateUserProps`, `FindUserProps` |
| Command (backend) | `<Action><Entity>Command` | `CreateUserCommand`, `UpdateUserCommand` |
| Query (backend) | `<Action><Entity>Query` | `FindUserQuery`, `ListUsersQuery` |
| Event (backend) | `<Entity><Action>Event` | `UserCreatedEvent`, `OrderCancelledEvent` |
| Port function Props | `<FunctionName>Props` | `FindByEmailProps`, `SaveUserProps` |
| Value object | `<Entity><Field>ValueObject` | `UserEmailValueObject` |
| Screen (SPA) | `<Screen>Screen.tsx\|vue` | `UserProfileScreen.tsx` |
| ViewModel (SPA) | `use<Screen>ViewModel.ts` | `useUserProfileViewModel.ts` |
| Page (Next.js) | `<Screen>Page.tsx` | `UserProfilePage.tsx` |
| Client Component (Next.js) | `<Screen>Client.tsx` | `UserProfileClient.tsx` |
| Server Action (Next.js) | `<action>.action.ts` | `update-user.action.ts` |
| Store (Vue) | `<module>.store.ts` (Pinia) | `user.store.ts` |
| Store (React/RN) | `<module>.store.ts` (Zustand) | `user.store.ts` |
| Form Model (frontend) | `<Action><Entity>FormModel` | `CreateUserFormModel`, `ChangePasswordFormModel` |
| Form Mapper (frontend) | `<Action><Entity>FormMapper` | `CreateUserFormMapper`, `ChangePasswordFormMapper` |
| Screen Mapper (frontend) | `<Screen>Mapper` | `LoginScreenMapper`, `UserProfileScreenMapper` |
| Presentation Model (frontend) | `<FreeName>Model` | `UserSummaryModel`, `UserPermissionsModel` |
| Driving Controller (backend) | `<Action><Protocol>Controller` | `CreateUserHttpController`, `ProductCreatedRpcController` |
| Driving Mapper (backend) | `<Action>Mapper` (request DTO → Command/Query) | `FindAllUsersMapper`, `CreateUserMapper` |
| Request DTO (backend) | `<Action>RequestDto` | `CreateUserRequestDto` |
| Single Response wrapper | `SingleResponse<T>` | `SingleResponse<SpotCreatorOutput>` |
| Paginated Response wrapper | `PaginatedResponse<T>` | `PaginatedResponse<SpotsFinderOutput>` |
| Layout (SPA) | `<Scope>Layout.tsx\|vue` | `PublicLayout.tsx`, `PrivateLayout.tsx` |
| Layout ViewModel (SPA) | `use<Scope>LayoutViewModel.ts` | `usePrivateLayoutViewModel.ts` |

### Naming Conventions — File Suffixes
| Concept | Suffix | Example |
|---|---|---|
| Entity | `.entity.ts` | `user.entity.ts` |
| Value Object | `.value-object.ts` | `user-email.value-object.ts` |
| Port (interface) | `.repository.ts` / `.port.ts` | `user.repository.ts` |
| Adapter | `.repository.ts` | `mongo-user.repository.ts`, `http-user.repository.ts` |
| Use Case | `.use-case.ts` | `user-creator.use-case.ts` |
| Use case Output (backend) | `.output.ts` | `spots-finder.output.ts` |
| Use case Mapper (backend) | `.mapper.ts` (in the use-case `mapper/` folder) | `spots-finder.mapper.ts` |
| Props (frontend) | `.props.ts` | `create-user.props.ts` |
| Command (backend) | `.command.ts` | `create-user.command.ts` |
| Query (backend) | `.query.ts` | `find-user.query.ts` |
| Event (backend) | `.event.ts` | `user-created.event.ts` |
| Port function Props | `.props.ts` | `find-by-email.props.ts` |
| Request DTO (backend) | `.request.dto.ts` | `create-user.request.dto.ts` |
| Driving Mapper (backend) | `.mapper.ts` | `find-all-users.mapper.ts` |
| HTTP Controller (backend) | `<action>.http.controller.ts` | `create-user.http.controller.ts` |
| RPC Controller (backend) | `<action>.rpc.controller.ts` | `product-created.rpc.controller.ts` |
| Messaging Controller (backend) | `<action>.messaging.controller.ts` | `order-placed.messaging.controller.ts` |
| Domain Exception | `.exception.ts` | `user-not-found.exception.ts` |
| Schema (DB) | `.schema.ts` | `user.schema.ts` |
| DI Tokens | `.di-tokens.ts` | `user.di-tokens.ts` |
| Test | `.spec.ts` | `user-creator.use-case.spec.ts` |
| Screen Mapper (frontend) | `<screen-kebab>.mapper.ts` | `login-screen.mapper.ts`, `user-profile-screen.mapper.ts` |
| Presentation Model (frontend) | `.model.ts` (free name + `Model` suffix) | `user-summary.model.ts`, `user-permissions.model.ts` |
| Form Model (frontend) | `.form-model.ts` | `create-user.form-model.ts` |
| Form Mapper (frontend) | `.form-mapper.ts` | `create-user.form-mapper.ts` |

All file names use **kebab-case**. Never use camelCase or PascalCase for file names.

### Result Pattern (manual)
Every project includes a manual `Result<T>` class in `src/base/lib/domain/result.ts`. The error type is **always `DomainException`** — it is not a generic parameter:

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
- Use `Result.ok(value)` and `Result.err(error)` in **domain** and **application** layers of ALL projects
- The error side is always `DomainException` — any concrete exception (e.g. `UserNotFoundException`) works because it `extends DomainException`
- In **backend**: convert error results to `HttpException` at the infrastructure boundary via a global interceptor
- In **frontend**: use `try/catch` in infrastructure and presentation layers
- Never throw domain errors — always return `Result.err(new DomainException())`

### Domain Base Classes
Every project includes these abstract base classes in `src/base/lib/domain/`. **Generate each file exactly as shown — do not rename classes:**

```typescript
// src/base/lib/domain/command.base.ts
export abstract class Command {}

// src/base/lib/domain/query.base.ts
export abstract class Query {}

// src/base/lib/domain/props.base.ts
export abstract class Props {}

// src/base/lib/domain/domain-exception.base.ts
export abstract class DomainException {}

// src/base/lib/domain/output.base.ts (backend only — base for use-case Outputs)
export abstract class Output {}

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
export class UserNotFoundException extends DomainException { ... }

// domain/events/user-created.event.ts (backend only)
import { Event } from '@/base/lib/domain/event.base'
export class UserCreatedEvent extends Event {
  constructor(public readonly userId: string, public readonly email: string) { super() }
}

// application/use-cases/spots-finder/spots-finder.output.ts (backend only)
import { Output } from '@/base/lib/domain/output.base'
export class SpotsFinderOutput extends Output { ... }
```

### Paginated Container (domain — universal)
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

### Domain Entity Rules
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

**2. Instantiation rules — `new <Entity>(...)` is only allowed in two places:**

   **a) Inside an infrastructure → domain mapper** (the canonical case). Adapters receive raw data from a data source (DB row, HTTP response, ORM model) and a mapper rebuilds the entity:
   ```typescript
   // infrastructure/mappers/user.mapper.ts
   export class UserMapper {
     static toDomain(raw: UserRecord): User {
       return new User(raw.id, raw.email, raw.name, raw.created_at, raw.updated_at)
     }
   }
   ```

   **b) Inside a use case, only when ALL of the following hold:**
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

### Port Parameters Convention
Port functions (repository methods, service methods) receive their parameters as domain classes. There are 3 cases:

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

### UseCase Base Class
Every project includes an abstract `UseCase` class in `src/base/lib/application/use-case.base.ts`. **Generate this file exactly as shown — do not rename the class, generics, or method:**

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

All use cases **extend** this class (never `implements`). **Backend use cases return an
`Output` (or `Paginated<Output>`) — never a domain entity or an infra DTO.** Frontend use cases
return domain data (the presentation layer adapts it via Screen Mapper / Presentation Model).
```typescript
// Backend — single result
export class SpotCreator extends UseCase<CreateSpotCommand, SpotCreatorOutput> {
  execute(command: CreateSpotCommand): Promise<Result<SpotCreatorOutput>> {
    // ...
  }
}

// Backend — paginated collection
export class SpotsFinder extends UseCase<FindSpotsQuery, Paginated<SpotsFinderOutput>> {
  execute(query: FindSpotsQuery): Promise<Result<Paginated<SpotsFinderOutput>>> {
    // ...
  }
}

// Frontend — returns domain data
export class UserCreator extends UseCase<CreateUserProps, User> {
  execute(props: CreateUserProps): Promise<Result<User>> {
    // ...
  }
}
```

### Use Case Output & Mapper Convention (backend only)

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
  async execute(query: FindSpotsQuery): Promise<Result<Paginated<SpotsFinderOutput>>> {
    const result = await this.spotRepository.matching(/* criteria */)
    if (result.isErr()) return Result.err(result.getError())

    const { items, total } = result.getValue()
    return Result.ok(
      new Paginated(items.map(SpotsFinderMapper.toOutput), total, query.pageNumber, query.pageSize),
    )
  }
}
```

**4. Controller** wraps the Output directly (no separate response DTO) and derives `totalPages`
at the boundary:
```typescript
async find(@Query() dto: FindSpotsQueryDto): Promise<PaginatedResponse<SpotsFinderOutput>> {
  const result = await this.spotsFinder.execute(FindSpotsHttpMapper.toQuery(dto))
  if (result.isErr()) throw result.getError()

  const { items, total, page, limit } = result.getValue()
  const totalPages = total === 0 ? 0 : Math.ceil(total / limit)
  return new PaginatedResponse(items, { page, limit, total, totalPages })
}
```

> A single-result use case returns the Output directly (`UseCase<CreateSpotCommand, SpotCreatorOutput>`)
> and its controller wraps it in `SingleResponse<SpotCreatorOutput>`.

### Form Model & Form Mapper Convention (frontend only)

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

### Screen Mapper & Presentation Model Convention (frontend only)

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

### Driving Vertical Slice Convention (backend only)

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

### Response Wrapper Convention (backend only)

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
- The **Output** (built by the use-case Mapper) already includes only the fields the API
  exposes — internal, derived, and sensitive fields are stripped there, not in the controller.

### Testing Conventions
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
| Vue.js | Vitest | `@vue/test-utils` |
| React SPA | Vitest | `@testing-library/react` |
| React Native | Jest (included with Expo) | `@testing-library/react-native` |
| Next.js | Vitest | `@testing-library/react` |

Test files live in a `tests/` folder at the root level, mirroring the `src/` structure:
```
src/modules/user/application/use-cases/user-creator/user-creator.use-case.ts
tests/modules/user/application/use-cases/user-creator/user-creator.use-case.spec.ts
```

### Code Quality
All projects use `husky` + `lint-staged` for pre-commit hooks that run linting and formatting automatically.

### Dependency Rule (enforced via `eslint-plugin-boundaries`)
- Modules **never** import from other modules
- If something is needed in more than one module → move it to `shared/` (monolith/frontend)
- `domain/` has zero external dependencies
- `application/` depends only on `domain/`
- `infrastructure/` depends on `application/` and `domain/`
- `presentation/` depends on `application/` and `domain/`

These rules are **enforced at lint time** using `eslint-plugin-boundaries`. Every project installs it as a dev dependency and configures ESLint to define boundary elements (one per layer) with `default: 'disallow'` and explicit `allow` rules matching the list above. Violations fail lint and therefore fail the `lint-staged` pre-commit hook.

See each reference file for the project-specific ESLint configuration — backend and frontend differ in layer count and folder structure.

### src/base Structure
Every project has a `src/base/` folder with technical cross-cutting concerns:

```
src/base/
├── config/          ← technical infrastructure (env, http, logger, di, etc.)
├── constants/
│   └── index.ts     ← project-wide constants (export const VARIABLE_NAME = value)
└── lib/             ← abstract base classes extended by modules
    ├── domain/         ← result.ts, command.base.ts, query.base.ts, props.base.ts, domain-exception.base.ts, event.base.ts, value-object.base.ts
    ├── application/    ← use-case.base.ts
    ├── infrastructure/ ← single-response.ts, paginated-response.ts (backend only)
    └── utils/          ← pure utility functions grouped by concern
```

### Environment Variables Convention

Every project follows these rules for environment variables:

**Who creates each file:**
- `.env.example` — **created by the agent** when scaffolding the project. Contains all required variables with empty or placeholder values. **Committed to the repo.**
- `.env` / `.env.local` — **created by the developer** with real values. **Never committed** (added to `.gitignore`).
- `.env.development` / `.env.production` — optional, created by the developer per environment.

**Variable prefix by project type** (enforced by the platform, not optional):
- **NestJS backend**: no prefix — accessed via `process.env.VAR_NAME`, read through `EnvVarsService`
- **Vite (Vue, React)**: `VITE_` prefix — only `VITE_*` variables are exposed to the browser via `import.meta.env`
- **Next.js**: `NEXT_PUBLIC_` prefix for client-accessible variables; no prefix for server-only variables
- **Expo (React Native)**: `EXPO_PUBLIC_` prefix — only `EXPO_PUBLIC_*` variables are bundled into the app

**Access pattern**: never read `process.env` or `import.meta.env` directly outside of `src/base/config/env/`. All code that needs an env var imports from the centralized env config. This means:
- One place to validate all env vars at startup
- One place to fix a missing var
- Type-safe access everywhere else

See each reference file for the full implementation (schema, service/config, `.env.example`).

### Constants Convention
All project-wide constants live in `src/base/constants/index.ts`. Use `UPPER_SNAKE_CASE` for names:
```typescript
// src/base/constants/index.ts
export const API_BASE_URL = 'https://api.example.com'
export const MAX_RETRY_ATTEMPTS = 3
export const DEFAULT_PAGE_SIZE = 20
```

### Utils Convention
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

Backend also includes:
```
src/base/
├── health/          ← healthcheck controller and module
└── config/
    ├── context/     ← correlation ID, request context
    ├── messaging/   ← message broker client (RabbitMQ by default)
    └── nestjs/      ← global exception filter, logger interceptor
```

---

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

---

## Workflow — Creating a New Project

When the user asks to create a project from scratch, follow these steps:

1. **Confirm project type** with the user if not obvious
2. **Verify the official documentation** — Before scaffolding, fetch the official "getting started" page of the framework to confirm the recommended way to create a new project and ensure you are using the latest version. Use these URLs:

   | Framework | Official Docs |
   |---|---|
   | NestJS | https://docs.nestjs.com |
   | Vue.js | https://vuejs.org/guide/quick-start |
   | React (Vite) | https://vite.dev/guide |
   | React Native | https://docs.expo.dev/get-started/create-a-project |
   | Next.js | https://nextjs.org/docs/getting-started/installation |

   This step is critical — CLI commands and recommended flags change between versions. Never rely solely on the commands in the reference files; always cross-check with the official docs.

3. **Read the reference file** for that project type (see table above)
4. **Scaffold the project** using the verified CLI command from the official docs
5. **Install dependencies** listed in the reference file for that project type
6. **Create `src/base/`** with config and lib folders as described in the reference
7. **Create the first module** (if the user specified a domain) following the folder structure
8. **Register DI** — wire up ports and adapters in the DI container (NestJS module or Awilix)
9. **Register routes** — add module routes to the router/navigation config (except Next.js, which uses file system)

---

## Workflow — Adding a Module or Feature

When the user asks to add a module, feature, use case, or any new files:

1. **Detect the project type** from `package.json`, existing folder structure, or user description
2. **Read the reference file** for that project type
3. **Create the module folder structure** inside `src/modules/<domain>/` (monolith/frontend) or `src/app/` (microservice)
4. **Generate files** following naming conventions (both class names and file suffixes)
5. **Respect the dependency rule** — domain has no imports from outer layers
6. **Register in DI** — add providers to the NestJS module or register in Awilix container
7. **Add routes** if the module has HTTP endpoints (backend) or screens (frontend)
8. **Generate tests** in `tests/` mirroring the `src/` path: `src/.../foo.ts` → `tests/.../foo.spec.ts`

Do not skip steps 6-8 unless the user explicitly says so.

---

## Reference Files

For full folder structures, DI patterns, framework-specific conventions, project setup commands, and code examples, read the appropriate reference:

- **NestJS (monolith or microservice)** → read `references/backend-nestjs.md`
- **Vue.js SPA** → read `references/frontend-spa-vue.md`
- **React SPA** → read `references/frontend-spa-react.md`
- **React Native (Expo)** → read `references/frontend-mobile-react-native.md`
- **Next.js** → read `references/frontend-nextjs.md`
