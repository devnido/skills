---
name: hexagonal-architecture
description: >
  Enforces hexagonal architecture (Ports & Adapters) with vertical slicing for all
  TypeScript projects: NestJS monoliths, NestJS microservices, React SPAs, React Native
  apps, and Next.js apps. Use this skill whenever the user creates a project,
  adds a module, scaffolds a feature, generates a use case, creates a repository, adds a
  controller, creates a component, adds a screen, or does anything involving file/folder
  structure. Also trigger when the user asks about architecture, folder structure, naming
  conventions, how to organize code, where to place a file, or how layers connect. Even
  if the user doesn't explicitly mention "hexagonal" or "architecture", use this skill
  any time the task involves generating, moving, or restructuring TypeScript project files.
  Do NOT trigger for non-TypeScript work, standalone scripts, tooling/config-only changes
  (CI, linters, dotfiles), or repositories that are not one of the five supported project
  types (e.g. documentation or skills repos). Do NOT trigger for Vue.js SPAs — those are
  covered by the `vuejs-hexagonal` skill.
---

# Hexagonal Architecture Skill

This skill enforces a consistent hexagonal architecture (Ports & Adapters) across all
TypeScript projects. This file is the router and rule digest. **Before generating any file
or folder, read TWO reference files:**

1. `references/universal-conventions.md` — full universal rules and the canonical code
   templates (Result, base classes, Paginated, wrappers, entity/VO/port rules, env/utils
   conventions). **Mandatory for every task** — the digest below summarizes it but the
   reference is the source of truth.
2. The project-type reference (table below) — framework-specific structure, DI, and setup.

## Project Type Detection

Identify the project type from context (package.json, existing structure, or user description):

| Type | Indicators | Reference File |
|---|---|---|
| NestJS Monolith | NestJS + multiple domains/modules | `references/backend-nestjs.md` |
| NestJS Microservice | NestJS + single domain | `references/backend-nestjs.md` |
| React SPA | React + React Router (no Next.js) | `references/frontend-spa-react.md` |
| React Native | Expo + React Navigation | `references/frontend-mobile-react-native.md` |
| Next.js | Next.js + App Router | `references/frontend-nextjs.md` |

**Always read the reference files before generating any structure.**

> **Vue.js SPA** (Vue 3 + Vue Router) is **not** covered here: use the `vuejs-hexagonal`
> skill instead.

> If you cannot determine the project type from `package.json`, folder structure, or user
> description, **ask the user before generating anything**. Never guess the project type.

### Monorepos (multiple projects in one repo)

- The **nearest `package.json`** walking up from the file you are touching defines the
  project and its type. A root `package.json` with `workspaces` (or `pnpm-workspace.yaml`,
  `turbo.json`, `nx.json`) is **not a project** — it is orchestration; never detect the
  type from it.
- Every path in this skill (`src/`, `tests/`, `.env.example`, ESLint config) is relative
  to **that project's root**, never the repo root.
- **Each project is its own hexagon.** Never import code from a sibling project — services
  talk only through their public contracts (REST API, message broker). This is the
  "modules never import other modules" rule, one level up.
- The duplication of `src/base/` across sibling projects is **deliberate** (it is the
  price of each service's autonomy). Do not "deduplicate" it into a repo-level shared
  folder or workspace package — extracting a shared package is an explicit user decision,
  never the agent's initiative.
- A task that spans two projects is N sub-tasks: re-read the matching reference when
  entering each project. If it is unclear which project a change belongs to, **ask**.

---

## Layers

Every project has exactly 3 layers (backend) or 4 layers (frontend):
1. **domain/** — entities, value objects, ports (interfaces), domain errors, domain events
2. **application/** — use cases, DTOs, application errors
3. **infrastructure/** — adapters, clients, models, schemas, controllers
4. **presentation/** — screens, view models, components, routes, store *(frontend only)*

**Backend infrastructure is split into `driving/` and `driven/`:**
- `infrastructure/driving/` — entry points that **invoke** the application (HTTP controllers, RPC consumers, message consumers, CLI handlers, cron jobs). Organized by **protocol** then **vertical slice per action**.
- `infrastructure/driven/` — outbound adapters that the application **invokes** (repositories, message publishers, HTTP clients to other services, email senders, etc.). Organized by technical concern (`persistence/`, `messaging/`, `http/`, `email/`).

This split makes the hexagonal direction explicit: driving adapters push into the hexagon, driven adapters are pushed by the hexagon.

## Naming Conventions — Classes & Interfaces

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
| Screen (SPA) | `<Screen>Screen.tsx` | `UserProfileScreen.tsx` |
| ViewModel (SPA) | `use<Screen>ViewModel.ts` | `useUserProfileViewModel.ts` |
| Page (Next.js) | `<Screen>Page.tsx` | `UserProfilePage.tsx` |
| Client Component (Next.js) | `<Screen>Client.tsx` | `UserProfileClient.tsx` |
| Server Action (Next.js) | `<action>.action.ts` | `update-user.action.ts` |
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
| Layout (SPA) | `<Scope>Layout.tsx` | `PublicLayout.tsx`, `PrivateLayout.tsx` |
| Layout ViewModel (SPA) | `use<Scope>LayoutViewModel.ts` | `usePrivateLayoutViewModel.ts` |

## Naming Conventions — File Suffixes

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

## Universal Rules — Digest

Full rules and canonical code templates live in `references/universal-conventions.md`.
Summary of the non-negotiables:

- **Error handling** — **Frontend (Result pattern):** `domain/` and `application/` return `Result<T>` (`Result.ok` / `Result.err`, error side always a `DomainException` subclass); adapters `try/catch` the HTTP client and convert every failure to `Result.err`; the ViewModel unwraps the `Result` and maps the error `code` to UI state — it never throws. **Backend (no Result):** `domain/` and `application/` **throw** `DomainException` directly and return plain values; a global `DomainExceptionFilter` maps the thrown exception's `code` to an `HttpException` (the **only** error-mapping mechanism — no `Result`, no unwrap interceptor). **Backend adapters wrap every technical operation (DB/HTTP/queue/S3 call) in `try/catch`: on failure they log the error and throw an infra `DomainException` tied to that adapter (`DatabaseErrorException`, `DownstreamServiceErrorException`, …, `httpStatus 500`); business "not found"/"duplicate" exceptions are thrown from the successful result, not from the `catch`.**
- **Domain base classes** (`src/base/lib/domain/`) — `Command`, `Query`, `Props`, `Output` (made nominal via a private `_brand` field) and `DomainException extends Error` (abstract stable `code`, message via `super`); `Event` with `eventId`/`occurredAt` (backend). Every input, output, error, and event extends its base. Generate these files from the templates in the universal reference — never improvise their shape.
- **`Paginated<T>`** (`src/base/lib/domain/paginated.ts`) — canonical return for collections: `items / total / page / limit`. No `totalPages` — it is derived at the controller boundary.
- **Domain entities** — `id`, `createdAt`, `updatedAt` are required and non-nullable; no shared `Entity` base class. `new <Entity>(...)` is allowed only in infrastructure→domain mappers (canonical) or, exceptionally, in a use case when every field already exists in memory. Never build an entity for data that doesn't exist yet — pass the Command/Props to the adapter; it persists and returns the entity.
- **Value objects** — validated single-value VO = class `<Entity><Field>ValueObject`; structural data VO = plain interface without suffix. Business VOs shared by ≥2 modules go in `modules/shared/domain/value-objects/`; aggregate-owned VOs in the module. Never in `src/base/` (technical machinery only).
- **Port file isolation** — a port file (`domain/ports/<entity>.repository.ts`) declares **only** the port interface(s) and nothing else. Every supporting type it references — argument shapes and return/result wrappers — is extracted to its own file under `domain/props/`, never declared inline in the port file. Example: `SpotRepository` holds only the interface; its `MatchingSpots` result and its `CreateSpotProps` argument each live in `domain/props/`.
- **Port parameters** — ports receive domain classes: reuse the use case's Command/Query/Props when the params match exactly; an `Event` for publishers; otherwise a dedicated `<FunctionName>Props` in `domain/props/` (suffix `.props.ts`) containing **only the fields that port method needs** (not the whole command). A port's return/result wrapper (e.g. `MatchingSpots = { items; total }`) is likewise its own file in `domain/props/`.
- **UseCase base** — every use case `extends UseCase<I, O>` (never `implements`), so any constructor it declares **must call `super()`** — omitting it is a compile error (TS2377). Backend use cases return an `Output` or `Paginated<Output>` — never a domain entity or infra DTO; frontend use cases return domain data. Backend DI: use cases are plain providers injected by class; DI tokens are reserved for ports bound to adapters.
- **Frontend DI (Awilix)** — wire every registration by hand with `asFunction((cradle) => new Foo(cradle.bar))`. **Never `asClass` + `InjectionMode.CLASSIC`**: it resolves by constructor parameter names and breaks in every minified production build (`Could not resolve 'e'`) while passing dev forever. Applies to React, React Native and Next.js (its server bundle is minified too); NestJS uses its own container and is unaffected.
- **Output & Mapper (backend)** — each use-case folder holds `<uc>.use-case.ts`, `<uc>.output.ts` (curated API fields, dates as ISO strings), and `mapper/<uc>.mapper.ts` with a static `toOutput(entity)`.
- **Form Model & Form Mapper (frontend)** — forms use an interface FormModel in `presentation/models/` plus a static FormMapper in `presentation/mappers/` that converts to domain `Props`, stripping UI-only fields.
- **Screen Mapper & Presentation Models (frontend)** — one mapper per screen (`<Screen>Mapper`), methods named `<source>To<target>` (never `toModel`/`fromModel`), models are interfaces with only the fields the UI renders.
- **MVVM (SPAs & React Native only — never Next.js)** — the Screen (View) is passive: it consumes only its ViewModel's return and never imports `domain/`/`application/`, the DI container, stores, or mappers. The ViewModel (`use<Screen>ViewModel`) receives route params as arguments, self-initializes, builds Props classes, unwraps the use case's `Result`, and returns only Presentation Models + UI state + actions. Next.js uses Server Components + Server Actions instead — there the unwrap point is the Server Component (reads: `notFound()` or throw) and the Server Action (mutations: return a serializable `{ error }`).
- **Driving vertical slice (backend)** — `infrastructure/driving/<protocol>/<action>/` with `dto/` (request side only — the use-case Output is the response contract), `mapper/` (request DTO → Command/Query), and `<action>.<protocol>.controller.ts`. Only the controller knows the protocol.
- **Response wrappers (backend)** — every controller returns `SingleResponse<Output>` or `PaginatedResponse<Output>` **directly** (never a `Result`, never a raw entity/array/DTO), deriving `totalPages` at the boundary.
- **Testing** — domain: pure unit tests; application: unit tests with mocked ports; infrastructure: unit tests with mocked clients; presentation: component tests. Tests live in `tests/` mirroring `src/`. **Generate tests from the canonical templates** in the universal reference (use case, mapper, adapter; ViewModel template in each SPA reference) — mock only at the port / use-case seam; entity fixtures come from builders in `tests/modules/<module>/builders/`.
- **Dependency rule** (enforced with `eslint-plugin-boundaries`) — a module never imports another module's **internals** (entities, adapters, use cases); shared *business* code goes to `shared/`; `domain/` depends on nothing; `application/` → `domain/`; `infrastructure/` and `presentation/` → `application/` + `domain/`. **Exception — cross-module ports (monolith):** a module MAY import another module's **port interface + its DI token** and inject it (the consumer imports the providing module, which `exports` the token). This is the only sanctioned inter-module import; without it every cross-module contract would be forced into `shared/`. A use case may inject such a port (e.g. `SpotCreator` injecting the skater module's `SKATER_REPOSITORY`) — it depends on the interface, never on the other module's adapter.
- **`src/base/`** — technical cross-cutting only: `config/`, `constants/index.ts` (UPPER_SNAKE_CASE), `lib/` (base classes + pure `utils/` grouped by concern). The Criteria pattern lives in `base/lib/**/criteria/` (see the `criteria-pattern` skill).
- **Env vars** — the agent creates `.env.example` (committed); platform prefixes are mandatory (`VITE_`, `NEXT_PUBLIC_`, `EXPO_PUBLIC_`, none for NestJS); only `src/base/config/env/` reads `process.env` / `import.meta.env`.
- **Auth & session (frontend)** — tokens are technical machinery in `src/base/config/auth/` (`TokenStorage` port; localStorage / SecureStore / httpOnly cookies per platform); the base HTTP client injects `Authorization` + stable identity headers (e.g. `device-id`) and runs a **single-flight refresh on 401** through a bare client; session state lives in the auth module's store (MVVM rules apply — tokens never enter the store); login/register are a regular `modules/auth/` module. Next.js SSG authenticates with a build-time guest/service token from env vars.
- **Code quality** — every project uses `husky` + `lint-staged` pre-commit hooks running lint and format.

---

## Workflow — Creating a New Project

When the user asks to create a project from scratch, follow these steps:

1. **Confirm project type** with the user if not obvious
2. **Verify the official documentation** — Before scaffolding, fetch the official "getting started" page of the framework to confirm the recommended way to create a new project and ensure you are using the latest version. Use these URLs:

   | Framework | Official Docs |
   |---|---|
   | NestJS | https://docs.nestjs.com |
   | React (Vite) | https://vite.dev/guide |
   | React Native | https://docs.expo.dev/get-started/create-a-project |
   | Next.js | https://nextjs.org/docs/getting-started/installation |

   This step is critical — CLI commands and recommended flags change between versions. Never rely solely on the commands in the reference files; always cross-check with the official docs.

3. **Read `references/universal-conventions.md` and the project-type reference** (see table above)
4. **Scaffold the project** using the verified CLI command from the official docs
5. **Install dependencies** listed in the reference file for that project type
6. **Create `src/base/`** with config and lib folders as described in the references
7. **Create the first module** (if the user specified a domain) following the folder structure
8. **Register DI** — wire up ports and adapters in the DI container (NestJS module or Awilix)
9. **Register routes** — add module routes to the router/navigation config (except Next.js, which uses file system)

---

## Workflow — Adding a Module or Feature

When the user asks to add a module, feature, use case, or any new files:

1. **Detect the project type** from `package.json`, existing folder structure, or user description
2. **Read `references/universal-conventions.md` and the project-type reference**
3. **Create the module folder structure** inside `src/modules/<domain>/` (monolith/frontend) or `src/app/` (microservice)
4. **Generate files** following naming conventions (both class names and file suffixes)
5. **Respect the dependency rule** — domain has no imports from outer layers
6. **Register in DI** — add providers to the NestJS module or register in Awilix container
7. **Add routes** if the module has HTTP endpoints (backend) or screens (frontend)
8. **Generate tests** in `tests/` mirroring the `src/` path: `src/.../foo.ts` → `tests/.../foo.spec.ts`

Do not skip steps 6-8 unless the user explicitly says so.

---

## Reference Files

- **Universal conventions (all project types)** → read `references/universal-conventions.md` — full rules and canonical code templates for everything in the digest above
- **NestJS (monolith or microservice)** → read `references/backend-nestjs.md`
- **React SPA** → read `references/frontend-spa-react.md`
- **React Native (Expo)** → read `references/frontend-mobile-react-native.md`
- **Next.js** → read `references/frontend-nextjs.md`

---

## Maintaining This Skill

This SKILL.md is deliberately thin: it loads in full on every trigger, so every line here
costs context in every coding task. The full rules and code templates live in
`references/universal-conventions.md`, which is read on demand. Keep that contract:

- **If a rule keeps getting violated in practice** (the agent skips the reference and
  generates something wrong), the fix is to **promote that specific rule to the digest
  above** — one bullet, no code blocks. Do NOT paste the full section back into this file.
- **New conventions** go into `references/universal-conventions.md` (or the project-type
  reference if framework-specific), plus at most one digest bullet here.
- Never duplicate code templates between this file and the references — single source of
  truth is the reference; drift is worse than verbosity.
- Any edit to this file must be mirrored in Spanish in `README.es.md` (see the
  `create-skill` skill, which automates this).
