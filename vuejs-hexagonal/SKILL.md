---
name: vuejs-hexagonal
description: >
  Enforces hexagonal architecture (Ports & Adapters) with vertical slicing and MVVM in
  Vue.js SPAs (Vue 3 + Vue Router + Pinia + Vite + Awilix). Use this skill whenever the
  user creates a Vue project, adds a module or feature, scaffolds a use case, creates a
  repository/adapter, adds a screen, component, composable, ViewModel, store, route or
  layout, or does anything involving file/folder structure in a Vue codebase. Also
  trigger when the user asks about architecture, folder structure, naming conventions,
  MVVM, ViewModels, screen/form mappers, presentation models, DI with Awilix, the Result
  pattern, or where to place a file in a Vue app — including phrases like "arquitectura
  hexagonal en Vue", "crear un módulo Vue", "nuevo caso de uso en el front", "dónde va
  este archivo", "cómo estructuro el proyecto Vue", "agregar una pantalla", "crear un
  ViewModel", "add a Vue module", "scaffold a Vue feature", "new screen", "vue hexagonal".
  Even if the user doesn't explicitly say "hexagonal" or "architecture", use this skill
  any time the task involves generating, moving, or restructuring files in a Vue.js SPA.
  Do NOT trigger for non-Vue projects (NestJS, React, React Native, Next.js — those are
  covered by the broader `hexagonal-architecture` skill), for tooling/config-only changes
  (CI, linters, dotfiles), or for pure styling/visual-design work with no structural impact.
---

# Vue.js Hexagonal Architecture Skill

This skill enforces a consistent hexagonal architecture (Ports & Adapters) with vertical
slicing and MVVM in **Vue.js SPAs**. This file is the router and rule digest. **Before
generating any file or folder, read BOTH reference files:**

1. `references/vue-conventions.md` — canonical rules and code templates (Result, base
   classes, `Paginated`, entity/VO/port rules, form & screen mappers, auth, testing,
   env/constants/utils). **Mandatory for every task** — the digest below summarizes it,
   but the reference is the source of truth.
2. `references/frontend-spa-vue.md` — Vue-specific setup, project structure, HTTP client,
   DI with Awilix, MVVM, router, layouts, and a full module example.

## Scope

| Applies to | Does NOT apply to |
|---|---|
| Vue 3 SPA (Composition API) + Vue Router + Pinia, built with Vite | NestJS, React SPA, React Native, Next.js → use the `hexagonal-architecture` skill |

Confirm the project is a Vue SPA from `package.json` (`vue`, `vue-router`, `vite`) or the
existing structure. **If you cannot determine the project type, ask the user before
generating anything.** Never guess.

### Monorepos (multiple projects in one repo)

- The **nearest `package.json`** walking up from the file you are touching defines the
  project and its type. A root `package.json` with `workspaces` (or `pnpm-workspace.yaml`,
  `turbo.json`, `nx.json`) is **not a project** — it is orchestration; never detect the
  type from it.
- Every path in this skill (`src/`, `tests/`, `.env.example`, ESLint config) is relative
  to **that project's root**, never the repo root.
- **Each project is its own hexagon.** Never import code from a sibling project — the SPA
  talks to other services only through their public contracts (the REST API). The
  duplication of `src/base/` across sibling projects is **deliberate**; do not
  "deduplicate" it into a workspace package — extracting a shared package is an explicit
  user decision, never the agent's initiative.
- If the change belongs to a non-Vue sibling project, stop and use the
  `hexagonal-architecture` skill for that project instead.

---

## Layers

Every module has exactly 4 layers:
1. **domain/** — entities, value objects, ports (interfaces), props, domain exceptions
2. **application/** — use cases
3. **infrastructure/** — adapters (HTTP repositories), infra mappers, DTOs/records
4. **presentation/** — screens (views), ViewModels, components, models, mappers, store, routes

The SPA's **driving side** is the presentation layer: the **ViewModel is the driving
adapter** (the SPA analog of a backend controller) — it is the only place that resolves
use cases from the DI container and the only place that unwraps a `Result`. The **driven
side** is `infrastructure/`: HTTP repositories implementing the domain's ports.

## Naming Conventions — Classes & Interfaces

| Concept | Convention | Example |
|---|---|---|
| Port (interface) | `<Entity><Role>` | `UserRepository` |
| Adapter (impl) | `<Infra><Entity><Role>` | `HttpUserRepository` |
| Use case | `<Entity><Action>` | `UserCreator`, `UserFinder`, `UserUpdater` |
| Use case folder | `<entity>-<action>/` | `user-creator/` |
| Paginated container (domain) | `Paginated<T>` | `Paginated<User>` |
| Domain exception | `<Entity><Reason>Exception` | `UserNotFoundException`, `UserAlreadyExistsException` |
| HTTP exception | `HttpServiceException` | `HttpServiceException` |
| Props | `<Action><Entity>Props` | `CreateUserProps`, `FindUserProps` |
| Port function Props | `<FunctionName>Props` | `FindByEmailProps`, `SaveUserProps` |
| Value object | `<Entity><Field>ValueObject` | `UserEmailValueObject` |
| Screen (View) | `<Screen>Screen.vue` | `UserProfileScreen.vue` |
| ViewModel | `use<Screen>ViewModel.ts` | `useUserProfileViewModel.ts` |
| Store (Pinia) | `<module>.store.ts` | `user.store.ts` |
| Form Model | `<Action><Entity>FormModel` | `CreateUserFormModel` |
| Form Mapper | `<Action><Entity>FormMapper` | `CreateUserFormMapper` |
| Screen Mapper | `<Screen>Mapper` | `LoginScreenMapper`, `UserProfileScreenMapper` |
| Presentation Model | `<FreeName>Model` | `UserSummaryModel`, `UserPermissionsModel` |
| Layout | `<Scope>Layout.vue` | `PublicLayout.vue`, `PrivateLayout.vue` |
| Layout ViewModel | `use<Scope>LayoutViewModel.ts` | `usePrivateLayoutViewModel.ts` |

## Naming Conventions — File Suffixes

| Concept | Suffix | Example |
|---|---|---|
| Entity | `.entity.ts` | `user.entity.ts` |
| Value Object | `.value-object.ts` | `user-email.value-object.ts` |
| Port (interface) | `.repository.ts` / `.port.ts` | `user.repository.ts` |
| Adapter | `.repository.ts` | `http-user.repository.ts` |
| Use Case | `.use-case.ts` | `user-creator.use-case.ts` |
| Props | `.props.ts` | `create-user.props.ts`, `find-by-email.props.ts` |
| Domain Exception | `.exception.ts` | `user-not-found.exception.ts` |
| Infra mapper | `.mapper.ts` | `user.mapper.ts` |
| Screen Mapper | `<screen-kebab>.mapper.ts` | `user-profile-screen.mapper.ts` |
| Presentation Model | `.model.ts` | `user-summary.model.ts` |
| Form Model | `.form-model.ts` | `create-user.form-model.ts` |
| Form Mapper | `.form-mapper.ts` | `create-user.form-mapper.ts` |
| Store | `.store.ts` | `user.store.ts` |
| Routes | `.routes.ts` | `user.routes.ts` |
| Test | `.spec.ts` | `user-creator.use-case.spec.ts` |

All file names use **kebab-case**, except Vue SFCs (`.vue` screens, layouts and
components), which are **PascalCase**. Never use camelCase for file names.

## Rules — Digest

Full rules and canonical code templates live in `references/vue-conventions.md`.
Summary of the non-negotiables:

- **Error handling (Result pattern)** — `domain/` and `application/` return `Result<T>`
  (`Result.ok` / `Result.err`, error side always a `DomainException` subclass) and **never
  throw**. Adapters `try/catch` the HTTP client and convert **every** failure into
  `Result.err` (mapping known statuses to domain exceptions; `HttpServiceException` is the
  generic fallback). The **ViewModel** unwraps the `Result` with `isErr()` and maps the
  error `code` to UI state — it never throws and `try/catch` is not its error channel.
- **Domain base classes** (`src/base/lib/domain/`) — `Command`, `Query`, `Props` (made
  nominal via a private `_brand` field) and `DomainException extends Error` (abstract
  stable `code`, message via `super`). Every input and error extends its base. Generate
  these files from the templates in the reference — never improvise their shape. There is
  **no** `Output` base and **no** `Event` base in the SPA.
- **`Paginated<T>`** (`src/base/lib/domain/paginated.ts`) — canonical return for
  collections: `items / total / page / limit`. No `totalPages` — it is derived in the
  presentation layer.
- **Domain entities** — `id`, `createdAt`, `updatedAt` are required and non-nullable; no
  shared `Entity` base class. `new <Entity>(...)` is allowed only in infrastructure→domain
  mappers (canonical), in test builders, or, exceptionally, in a use case when every field
  already exists in memory. Never build an entity for data that doesn't exist yet — pass
  the Props to the adapter; it persists and returns the entity.
- **Value objects** — validated single-value VO = class `<Entity><Field>ValueObject`;
  structural data VO = plain interface without suffix. Business VOs shared by ≥2 modules go
  in `modules/shared/domain/value-objects/`; aggregate-owned VOs in the module. Never in
  `src/base/` (technical machinery only).
- **Port file isolation** — a port file (`domain/ports/<entity>.repository.ts`) declares
  **only** the port interface(s). Every supporting type it references — argument shapes and
  return/result wrappers — is extracted to its own file under `domain/props/`, never
  declared inline in the port file.
- **Port parameters** — ports receive domain classes and always return
  `Promise<Result<T>>`: reuse the use case's Props when the params match exactly; otherwise
  a dedicated `<FunctionName>Props` in `domain/props/` (suffix `.props.ts`) containing
  **only the fields that port method needs**.
- **UseCase base** — every use case `extends UseCase<I, O>` (never `implements`) and
  returns **domain data wrapped in a `Result`** — never a Presentation Model, a raw HTTP
  DTO, or an unwrapped value.
- **Form Model & Form Mapper** — forms use an interface FormModel in
  `presentation/models/` plus a static FormMapper in `presentation/mappers/` that converts
  it to a domain `Props`, stripping UI-only fields (e.g. `passwordConfirmation`).
- **Screen Mapper & Presentation Models** — one mapper per screen (`<Screen>Mapper`),
  methods named `<source>To<target>` (never `toModel` / `fromModel`), models are interfaces
  with only the fields the UI renders. This is the SPA's equivalent of a backend Output.
- **MVVM (mandatory)** — the Screen (View) is passive: it consumes only its ViewModel's
  return and never imports `domain/` / `application/`, the DI container, stores, or
  mappers. The ViewModel (`use<Screen>ViewModel`) receives route params as arguments,
  self-initializes (`onMounted` inside the ViewModel), resolves use cases from the Awilix
  container once at the top, builds Props classes, unwraps the `Result`, and returns
  **only** Presentation Models + UI state (`isLoading`, `error`) + action functions.
- **DI with Awilix** — the typed container lives in `src/base/config/di/container.ts`
  (there is no framework-level DI). Ports are registered bound to their adapter; only
  ViewModels resolve from the container.
- **Pinia store** — holds **cross-screen** state only. ViewModels read/write it; Views
  never import it. Tokens never live in the store.
- **Testing** — domain: pure unit tests; application: unit tests with mocked ports;
  infrastructure: unit tests with a mocked HTTP client; presentation: ViewModel tests
  (override container registrations with `asValue(mock)`) and component tests with
  `@vue/test-utils`. Tests live in `tests/` mirroring `src/`. **Generate tests from the
  canonical templates** in the reference — mock only at the port / use-case seam; entity
  fixtures come from builders in `tests/modules/<module>/builders/`.
- **Dependency rule** (enforced with `eslint-plugin-boundaries`) — a module never imports
  another module's internals; shared *business* code goes to `modules/shared/`; `domain/`
  depends on nothing; `application/` → `domain/`; `infrastructure/` and `presentation/` →
  `application/` + `domain/`.
- **`src/base/`** — technical cross-cutting only: `config/` (env, http, auth, logger, di,
  router), `constants/index.ts` (UPPER_SNAKE_CASE), `lib/` (base classes + pure `utils/`
  grouped by concern). The Criteria pattern lives in `base/lib/**/criteria/` (see the
  `criteria-pattern` skill).
- **Env vars** — the agent creates `.env.example` (committed); the **`VITE_` prefix is
  mandatory** (only `VITE_*` reaches `import.meta.env`); only `src/base/config/env/` reads
  `import.meta.env`. Never put a secret in a `VITE_` variable — everything ships to the browser.
- **Auth & session** — tokens are technical machinery in `src/base/config/auth/`
  (`TokenStorage` port + `localStorage` adapter); the base HTTP client injects
  `Authorization` + stable identity headers and runs a **single-flight refresh on 401**
  through a bare client; session state lives in the auth module's store (tokens never enter
  the store); login/register are a regular `modules/auth/` module. The HTTP client never
  navigates — navigation is presentation's job.
- **Code quality** — `husky` + `lint-staged` pre-commit hooks running lint and format.

---

## Workflow — Creating a New Vue Project

1. **Confirm it is a Vue SPA** with the user if not obvious.
2. **Verify the official documentation** — before scaffolding, fetch
   https://vuejs.org/guide/quick-start to confirm the recommended way to create a project
   and the current version. CLI commands and flags change between versions; never rely
   solely on the commands in the reference file.
3. **Read `references/vue-conventions.md` and `references/frontend-spa-vue.md`.**
4. **Scaffold** using the verified CLI command (TypeScript, Vue Router, Pinia, ESLint,
   Prettier).
5. **Install dependencies** listed in the Vue reference (Tailwind, Axios, Awilix, zod,
   Vitest, `@vue/test-utils`, `eslint-plugin-boundaries`, husky, lint-staged).
6. **Create `src/base/`** with `config/` and `lib/` as described in the references.
7. **Configure boundary enforcement** (ESLint) and the Husky + lint-staged hooks.
8. **Create the first module** (if the user specified a domain) following the folder
   structure and the 4 layers.
9. **Register DI** — wire ports to adapters in the Awilix container.
10. **Register routes** — add the module's routes to `src/base/config/router/`.

## Workflow — Adding a Module or Feature

1. **Confirm the project is a Vue SPA** from `package.json` / existing structure.
2. **Read `references/vue-conventions.md` and `references/frontend-spa-vue.md`.**
3. **Create the module folder** inside `src/modules/<domain>/` with its 4 layers.
4. **Generate files** following the naming conventions (class names *and* file suffixes).
5. **Respect the dependency rule** — `domain/` imports nothing from outer layers.
6. **Apply MVVM** — every new screen gets a ViewModel; the View stays passive.
7. **Register in DI** — add the port/adapter and the use case to the Awilix container.
8. **Add routes** to the module's `*.routes.ts` and register them in the router.
9. **Generate tests** in `tests/` mirroring `src/`: `src/.../foo.ts` →
   `tests/.../foo.spec.ts`.

Do not skip steps 6-9 unless the user explicitly says so.

---

## Reference Files

- **Canonical rules and code templates** → read `references/vue-conventions.md`
- **Vue setup, structure, DI, MVVM, router, layouts, full module example** → read
  `references/frontend-spa-vue.md`

## Related Skills

- `hexagonal-architecture` — the multi-stack version (NestJS, React, React Native,
  Next.js, and Vue). Use it when the task is **not** a Vue SPA.
- `criteria-pattern` — dynamic filters + order + pagination behind a single
  `matching(criteria)` repository method.

---

## Maintaining This Skill

This SKILL.md is deliberately thin: it loads in full on every trigger, so every line here
costs context in every coding task. The full rules and code templates live in
`references/vue-conventions.md`, which is read on demand. Keep that contract:

- **If a rule keeps getting violated in practice** (the agent skips the reference and
  generates something wrong), the fix is to **promote that specific rule to the digest
  above** — one bullet, no code blocks. Do NOT paste the full section back into this file.
- **New conventions** go into `references/vue-conventions.md` (or
  `references/frontend-spa-vue.md` if framework-specific), plus at most one digest bullet
  here.
- Never duplicate code templates between this file and the references — single source of
  truth is the reference; drift is worse than verbosity.
- This skill was derived from `hexagonal-architecture`. When a **shared** rule changes in
  one of them (Result, base classes, entities, VOs, ports, testing, boundaries), review the
  other so the two do not diverge silently.
- Any edit to this file must be mirrored in Spanish in `README.es.md` (see the
  `create-skill` skill, which automates this).
