# Vue.js SPA Reference — Structure, Setup & MVVM

Framework-specific reference for the `vuejs-hexagonal` skill. Read it together with
`vue-conventions.md` (the canonical rules and code templates) before generating any file.

## Table of Contents
1. [Project Setup](#setup)
2. [Project Structure](#structure)
3. [src/base Structure](#base)
4. [HTTP Client](#http-client)
5. [Presentation Layer](#presentation)
6. [DI with Awilix](#di)
7. [Props Convention](#props)
8. [MVVM Convention](#mvvm)
9. [Router Convention](#router)
10. [Layouts Convention](#layouts)
11. [Module Example](#module-example)

---

## Project Setup {#setup}

### CLI Scaffold
```bash
npm create vue@latest <project-name>
# Select: TypeScript, Vue Router, Pinia, ESLint, Prettier
```

### Required Dependencies
```bash
npm install awilix           # dependency injection
npm install zod              # env and DTO validation
npm install axios            # HTTP client
npm install @vueuse/core     # essential composables (useStorage, useDebounceFn, etc.)

# Tailwind CSS
npm install -D tailwindcss @tailwindcss/vite

# Testing
npm install -D vitest @vue/test-utils

# Code quality
npm install -D husky lint-staged
npm install -D eslint-plugin-boundaries  # layer boundary enforcement
npx husky init
```

### Post-scaffold Steps
1. Create `src/base/` structure with config and lib folders
2. Create `src/modules/shared/` and first domain module
3. Set up Awilix container in `src/base/config/di/container.ts`
4. Set up env validation with zod in `src/base/config/env/`
5. Configure path alias `@/` → `src/` in `tsconfig.json` and `vite.config.ts`
6. Configure Tailwind CSS in `vite.config.ts` and add `@import "tailwindcss"` to main CSS
7. Configure `vitest` in `vite.config.ts` with `@/` alias (see below)
8. Configure `tsconfig.vitest.json` to include `tests/**/*` (see below)
9. Configure `eslint-plugin-boundaries` in ESLint config (see below)
10. Configure `husky` + `lint-staged` for pre-commit hooks (see below)

### Husky + lint-staged Configuration

After `npx husky init`, replace `.husky/pre-commit` contents:
```bash
npx lint-staged
```

Add to `package.json`:
```json
{
  "lint-staged": {
    "*.{ts,vue}": [
      "eslint --fix",
      "prettier --write"
    ]
  }
}
```

### Vitest + TypeScript Configuration

Tests live in `tests/` at the root (mirroring `src/`). Two files must be configured so that TypeScript resolves the `@/` alias and vitest finds the test files:

**`tsconfig.vitest.json`** — add `tests/**/*` to `include`:
```json
{
  "extends": "./tsconfig.app.json",
  "include": ["src/**/__tests__/*", "tests/**/*", "env.d.ts"],
  "exclude": [],
  "compilerOptions": {
    "composite": true,
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.vitest.tsbuildinfo",
    "types": ["vitest/globals"]
  }
}
```

**`vite.config.ts`** — add the `test` section with the `@/` alias:
```typescript
import { fileURLToPath, URL } from 'node:url'
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import tailwindcss from '@tailwindcss/vite'

export default defineConfig({
  plugins: [vue(), tailwindcss()],
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url)),
    },
  },
  test: {
    environment: 'jsdom',
    root: '.',
    include: ['tests/**/*.spec.ts'],
    resolve: {
      alias: {
        '@': fileURLToPath(new URL('./src', import.meta.url)),
      },
    },
  },
})
```

Without these changes, tests in `tests/` will fail to resolve `@/` imports.

### Boundary Enforcement (`eslint-plugin-boundaries`)

Add the plugin and boundary rules to your ESLint flat config. This enforces the hexagonal dependency rule at lint time — violations fail the pre-commit hook.

**Frontend layers (4):** `domain` → `application` → `infrastructure` / `presentation`. Infrastructure and presentation are siblings — neither can import from the other.

```typescript
// eslint.config.ts
import boundaries from 'eslint-plugin-boundaries'

export default [
  // ... your existing Vue/TypeScript config ...
  {
    plugins: { boundaries },
    settings: {
      'boundaries/elements': [
        { type: 'domain',       pattern: ['src/modules/*/domain/**'],         capture: ['module'] },
        { type: 'application',  pattern: ['src/modules/*/application/**'],    capture: ['module'] },
        { type: 'infra',        pattern: ['src/modules/*/infrastructure/**'], capture: ['module'] },
        { type: 'presentation', pattern: ['src/modules/*/presentation/**'],   capture: ['module'] },
        // Cross-cutting
        { type: 'shared-domain', pattern: ['src/modules/shared/domain/**'] },
        { type: 'shared-app',    pattern: ['src/modules/shared/application/**'] },
        { type: 'shared-infra',  pattern: ['src/modules/shared/infrastructure/**'] },
        { type: 'shared-pres',   pattern: ['src/modules/shared/presentation/**'] },
        { type: 'base',          pattern: ['src/base/**'] },
      ],
      'boundaries/ignore': ['**/*.spec.ts'],
    },
    rules: {
      'boundaries/element-types': [2, {
        default: 'disallow',
        rules: [
          // domain → only same-module domain, shared-domain, base
          {
            from: ['domain'],
            allow: [
              ['domain', { module: '${from.module}' }],
              'shared-domain',
              'base',
            ],
          },
          // application → domain + application (same module), shared-domain, shared-app, base
          {
            from: ['application'],
            allow: [
              ['domain', { module: '${from.module}' }],
              ['application', { module: '${from.module}' }],
              'shared-domain',
              'shared-app',
              'base',
            ],
          },
          // infrastructure → domain + application + infra (same module), shared-*, base
          // ❌ Cannot import from presentation
          {
            from: ['infra'],
            allow: [
              ['domain', { module: '${from.module}' }],
              ['application', { module: '${from.module}' }],
              ['infra', { module: '${from.module}' }],
              'shared-domain',
              'shared-app',
              'shared-infra',
              'base',
            ],
          },
          // presentation → domain + application + presentation (same module), shared-*, base
          // ❌ Cannot import from infrastructure
          {
            from: ['presentation'],
            allow: [
              ['domain', { module: '${from.module}' }],
              ['application', { module: '${from.module}' }],
              ['presentation', { module: '${from.module}' }],
              'shared-domain',
              'shared-app',
              'shared-pres',
              'base',
            ],
          },
          // shared layers follow the same inward rule
          { from: ['shared-domain'], allow: ['shared-domain', 'base'] },
          { from: ['shared-app'],    allow: ['shared-domain', 'shared-app', 'base'] },
          { from: ['shared-infra'],  allow: ['shared-domain', 'shared-app', 'shared-infra', 'base'] },
          { from: ['shared-pres'],   allow: ['shared-domain', 'shared-app', 'shared-pres', 'base'] },
          // base can import from base only
          { from: ['base'], allow: ['base'] },
        ],
      }],
      'boundaries/no-unknown': [2],
    },
  },
]
```

What this enforces:
- ❌ `domain/` importing from `application/`, `infrastructure/`, `presentation/`, or another module
- ❌ `application/` importing from `infrastructure/`, `presentation/`, or another module
- ❌ `infrastructure/` importing from `presentation/` (and vice-versa)
- ❌ Cross-module imports (e.g. `modules/user/` importing from `modules/order/`)
- ✅ All layers can import from `base/`
- ✅ All layers can import from `shared/` (respecting shared's own layer hierarchy)

---

## Project Structure {#structure}

```
src/
├── base/
├── modules/
│   ├── shared/
│   │   ├── domain/
│   │   │   ├── value-objects/
│   │   │   └── exceptions/
│   │   │       └── domain.exception.ts
│   │   ├── application/
│   │   │   └── use-case.base.ts
│   │   ├── infrastructure/
│   │   │   └── http/
│   │   │       └── interceptors/
│   │   │           └── auth.interceptor.ts
│   │   └── presentation/
│   │       ├── components/
│   │       ├── design-tokens/
│   │       └── layouts/
│   │           ├── public/
│   │           │   └── PublicLayout.vue
│   │           └── private/
│   │               ├── PrivateLayout.vue
│   │               └── usePrivateLayoutViewModel.ts
│   └── user/
│       ├── domain/          ← entities, value-objects, exceptions, props/, ports
│       ├── application/     ← use-cases/
│       ├── infrastructure/
│       └── presentation/
├── App.vue
└── main.ts
```

---

## src/base Structure {#base}

```
src/base/
├── config/
│   ├── env/
│   │   ├── env.config.ts          ← imported first in main.ts for validation
│   │   └── env.schema.ts          ← zod schema
│   ├── http/
│   │   ├── axios.http-client.ts
│   │   ├── http-response.ts
│   │   └── exception/
│   │       └── http-service.exception.ts
│   ├── logger/
│   │   └── logger.ts
│   ├── di/
│   │   └── container.ts           ← Awilix container setup
│   └── router/
│       └── index.ts               ← createRouter, imports module routes
├── constants/
│   └── index.ts                   ← project-wide constants (export const VARIABLE_NAME = value)
└── lib/
    ├── domain/
    │   ├── result.ts              ← manual Result<T> class
    │   ├── props.base.ts          ← abstract class Props
    │   ├── domain-exception.base.ts   ← abstract class DomainException extends Error (abstract `code`)
    │   └── value-object.base.ts
    ├── application/
    │   └── use-case.base.ts
    └── utils/                    ← pure functions grouped by concern
        ├── index.ts              ← re-exports all utils
        ├── date.utils.ts
        ├── string.utils.ts
        └── ...
```

### main.ts pattern
```typescript
import './assets/main.css'
import './base/config/env/env.config'   // validate env vars first

import { createApp } from 'vue'
import { createPinia } from 'pinia'
import App from './App.vue'
import router from './base/config/router'

const app = createApp(App)
app.use(createPinia())
app.use(router)
app.mount('#app')
```

### App.vue pattern
```vue
<script setup lang="ts">
import { RouterView } from 'vue-router'
</script>

<template>
  <RouterView />
</template>
```

### Router pattern
```typescript
// src/base/config/router/index.ts
import { createRouter, createWebHistory } from 'vue-router'
import { authRoutes } from '@/modules/auth/presentation/routes/auth.routes'
import { userRoutes } from '@/modules/user/presentation/routes/user.routes'

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes: [
    {
      path: '/',
      component: () => import('@/modules/shared/presentation/layouts/public/PublicLayout.vue'),
      children: [
        ...authRoutes,
      ],
    },
    {
      path: '/',
      component: () => import('@/modules/shared/presentation/layouts/private/PrivateLayout.vue'),
      children: [
        ...userRoutes,
      ],
    },
  ],
})

export default router
```

---

## Environment Variables {#env}

Vite only exposes variables prefixed with `VITE_` to the browser. **Never put secrets in `VITE_*` variables** — they are bundled into the client bundle.

**Generate all three files exactly as shown** when scaffolding a new project. Adjust variable names to match the actual project requirements.

### `.env.example` (committed to repo)
```bash
# API
VITE_API_BASE_URL=http://localhost:3000
```

### `src/base/config/env/env.schema.ts`
```typescript
import { z } from 'zod'

export const envSchema = z.object({
  VITE_API_BASE_URL: z.string().url(),
})

export type Env = z.infer<typeof envSchema>
```

### `src/base/config/env/env.config.ts`
```typescript
import { envSchema } from './env.schema'

const result = envSchema.safeParse(import.meta.env)

if (!result.success) {
  throw new Error(`Invalid environment variables:\n${result.error.toString()}`)
}

export const env = result.data
```

Usage anywhere in the app:
```typescript
import { env } from '@/base/config/env/env.config'

// env.VITE_API_BASE_URL — fully typed, validated at startup
```

Import order in `src/main.ts`:
```typescript
import './base/config/env/env.config'   // must be first — crashes early if vars are missing
import { createApp } from 'vue'
// ...
```

Rules:
- **Never** read `import.meta.env.VITE_X` directly outside `env.config.ts`. All other code imports from `env`.
- `env.config.ts` is imported first in `main.ts` so the app crashes immediately with a clear error if any variable is missing or invalid.
- Add new variables to **all three files** at the same time: `.env.example`, `env.schema.ts`, and `env.config.ts`.

---

## HTTP Client {#http-client}

**Generate these files exactly as shown.** This is the production-grade HTTP client used across all Vue.js SPA projects. It is an injectable class registered in the Awilix container and injected into adapters.

### `src/base/config/http/http-response.ts`
```typescript
export type HttpHeaders = Record<string, string>

export interface HttpResponse<T> {
  data: T
  status: number
  headers: HttpHeaders
}
```

### `src/base/config/http/exception/http-service.exception.ts`
```typescript
import { DomainException } from '@/base/lib/domain/domain-exception.base'
import axios from 'axios'

export class HttpServiceException extends DomainException {
  readonly code = 'HTTP_SERVICE_ERROR'

  constructor(
    public readonly statusCode: number,
    message: string,
    public readonly downstreamCode?: string,
  ) {
    super(message)
  }

  static fromAxiosError(error: unknown): HttpServiceException {
    if (axios.isAxiosError(error)) {
      return new HttpServiceException(
        error.response?.status ?? 500,
        error.response?.data?.message ?? error.message,
        error.code,
      )
    }
    return new HttpServiceException(500, 'Unexpected error')
  }
}
```

### `src/base/config/http/axios.http-client.ts`
```typescript
import type { AxiosInstance, AxiosRequestConfig, AxiosError } from 'axios'
import axios from 'axios'
import type { HttpResponse, HttpHeaders } from './http-response'
import { HttpServiceException } from './exception/http-service.exception'
import { API_BASE_URL, HTTP_TIMEOUT_MS, HTTP_RETRY_ATTEMPTS } from '@/base/constants'

export class AxiosHttpClient {
  private readonly client: AxiosInstance
  private readonly retryAttempts: number = HTTP_RETRY_ATTEMPTS

  constructor() {
    this.client = axios.create({
      baseURL: API_BASE_URL,
      timeout: HTTP_TIMEOUT_MS,
    })

    // Auth interceptor — attach token from localStorage
    this.client.interceptors.request.use((config) => {
      const token = localStorage.getItem('auth-token')
      if (token) {
        config.headers.Authorization = `Bearer ${token}`
      }
      return config
    })

    // Response interceptor — redirect to login on 401
    this.client.interceptors.response.use(
      (response) => response,
      (error) => {
        if (axios.isAxiosError(error) && error.response?.status === 401) {
          localStorage.removeItem('auth-token')
          window.location.href = '/login'
        }
        return Promise.reject(error)
      },
    )
  }

  async get<T>(url: string, config?: AxiosRequestConfig): Promise<HttpResponse<T>> {
    return this.request<T>({ method: 'GET', url, ...config })
  }

  async post<T>(url: string, data?: unknown, config?: AxiosRequestConfig): Promise<HttpResponse<T>> {
    return this.request<T>({ method: 'POST', url, data, ...config })
  }

  async put<T>(url: string, data?: unknown, config?: AxiosRequestConfig): Promise<HttpResponse<T>> {
    return this.request<T>({ method: 'PUT', url, data, ...config })
  }

  async patch<T>(url: string, data?: unknown, config?: AxiosRequestConfig): Promise<HttpResponse<T>> {
    return this.request<T>({ method: 'PATCH', url, data, ...config })
  }

  async delete<T>(url: string, config?: AxiosRequestConfig): Promise<HttpResponse<T>> {
    return this.request<T>({ method: 'DELETE', url, ...config })
  }

  private async request<T>(axiosConfig: AxiosRequestConfig): Promise<HttpResponse<T>> {
    let lastError: unknown

    for (let attempt = 0; attempt <= this.retryAttempts; attempt++) {
      try {
        if (attempt > 0) {
          const delay = Math.min(1000 * Math.pow(2, attempt - 1), 10000)
          await new Promise((resolve) => setTimeout(resolve, delay))
        }

        const response = await this.client.request<T>(axiosConfig)

        return {
          data: response.data,
          status: response.status,
          headers: response.headers as HttpHeaders,
        }
      } catch (error: unknown) {
        lastError = error
        if (!this.isRetryable(error) || attempt === this.retryAttempts) break
      }
    }

    throw HttpServiceException.fromAxiosError(lastError)
  }

  private isRetryable(error: unknown): boolean {
    if (!axios.isAxiosError(error)) return false
    const axiosError = error as AxiosError
    if (!axiosError.response) return true // network errors, timeouts
    return axiosError.response.status >= 500
  }
}
```

### DI Registration
```typescript
// src/base/config/di/container.ts
import { createContainer, asClass, InjectionMode } from 'awilix'
import { AxiosHttpClient } from '@/base/config/http/axios.http-client'

export const container = createContainer({ injectionMode: InjectionMode.CLASSIC })

container.register({
  httpClient: asClass(AxiosHttpClient).singleton(),
})
```

### Adapter Usage
```typescript
// infrastructure/repositories/http-user.repository.ts
import { AxiosHttpClient } from '@/base/config/http/axios.http-client'
import type { HttpServiceException } from '@/base/config/http/exception/http-service.exception'
import type { UserRepository } from '../../domain/ports/user.repository'
import type { CreateUserProps } from '../../domain/props/create-user.props'
import type { UserResponseDto } from '../dtos/user-response.dto'
import { Result } from '@/base/lib/domain/result'

export class HttpUserRepository implements UserRepository {
  constructor(private readonly httpClient: AxiosHttpClient) {}

  async save(props: CreateUserProps): Promise<Result<User>> {
    try {
      const response = await this.httpClient.post<UserResponseDto>('/users', props)
      return Result.ok(UserMapper.toDomain(response.data))
    } catch (error) {
      const httpError = error as HttpServiceException
      if (httpError.statusCode === 409) return Result.err(new UserAlreadyExistsException(props.email))
      return Result.err(httpError) // HttpServiceException extends DomainException — generic fallback
    }
  }
}
```

---

## Presentation Layer {#presentation}

```
modules/user/presentation/
├── screens/
│   ├── user-profile/
│   │   ├── UserProfileScreen.vue
│   │   └── useUserProfileViewModel.ts
│   └── change-password/
│       ├── ChangePasswordScreen.vue
│       └── useChangePasswordViewModel.ts
├── components/               ← components scoped to this module
│   └── UserCard.vue
├── routes/
│   └── user.routes.ts
├── store/                    ← Pinia stores for this module
│   └── user.store.ts
├── composables/              ← reusable composables (not ViewModels)
│   └── useUserPermissions.ts
├── mappers/                  ← one mapper per screen, domain ↔ presentation models
│   ├── user-profile-screen.mapper.ts
│   ├── change-password-screen.mapper.ts
│   └── create-user.form-mapper.ts
└── models/                   ← presentation-specific interfaces (free name + Model suffix)
    ├── user-summary.model.ts
    ├── user-permissions.model.ts
    └── create-user.form-model.ts
```

---

## DI with Awilix {#di}

```typescript
// src/base/config/di/container.ts
import { createContainer, asClass, InjectionMode } from 'awilix'
import { AxiosHttpClient } from '@/base/config/http/axios.http-client'
import type { UserRepository } from '@/modules/user/domain/ports/user.repository'
import { HttpUserRepository } from '@/modules/user/infrastructure/repositories/http-user.repository'
import { UserCreator } from '@/modules/user/application/use-cases/user-creator/user-creator.use-case'
import { UserFinder } from '@/modules/user/application/use-cases/user-finder/user-finder.use-case'

// Typed cradle: container.resolve('name') returns the right type, and a typo
// in a registration name fails at compile time instead of at runtime.
export interface Cradle {
  httpClient: AxiosHttpClient
  userRepository: UserRepository
  userCreator: UserCreator
  userFinder: UserFinder
}

export const container = createContainer<Cradle>({ injectionMode: InjectionMode.CLASSIC })

container.register({
  // Base infrastructure
  httpClient: asClass(AxiosHttpClient).singleton(),

  // Adapters (receive httpClient via constructor injection)
  userRepository: asClass(HttpUserRepository).singleton(),

  // Use cases (receive adapters via constructor injection)
  userCreator: asClass(UserCreator).singleton(),
  userFinder: asClass(UserFinder).singleton(),
})
```

---

## Props Convention {#props}

Use case input parameters are defined as classes in `domain/props/`:

```typescript
// domain/props/create-user.props.ts
import { Props } from '@/base/lib/domain/props.base'

export class CreateUserProps extends Props {
  constructor(
    public readonly name: string,
    public readonly email: string,
  ) {
    super()
  }
}
```

The use case receives the Props class and implements `UseCase`:
```typescript
// application/use-cases/user-creator/user-creator.use-case.ts
import { Result } from '@/base/lib/domain/result'
import { UseCase } from '@/base/lib/application/use-case.base'
import { CreateUserProps } from '../../domain/props/create-user.props'

export class UserCreator extends UseCase<CreateUserProps, User> {
  constructor(private readonly userRepository: UserRepository) {}

  async execute(props: CreateUserProps): Promise<Result<User>> {
    // ...
  }
}
```

---

## Form Model & Form Mapper {#form-mapper}

When a screen has a form, define a **Form Model** (interface) and a **Form Mapper** (static class) in the presentation layer. The mapper converts the form data into domain `Props` before calling the use case.

### Form Model
```typescript
// presentation/models/create-user.form-model.ts
export interface CreateUserFormModel {
  name: string
  email: string
  passwordConfirmation: string  // UI-only field, not part of domain Props
}
```

### Form Mapper
```typescript
// presentation/mappers/create-user.form-mapper.ts
import type { CreateUserFormModel } from '../models/create-user.form-model'
import { CreateUserProps } from '../../domain/props/create-user.props'

export class CreateUserFormMapper {
  static toProps(form: CreateUserFormModel): CreateUserProps {
    return new CreateUserProps(form.name, form.email)
  }
}
```

---

## Screen Mapper & Presentation Model {#screen-mapper}

Every screen that consumes domain entities defines a **Screen Mapper** that converts those entities into one or more **Presentation Models** tailored to that screen's UI.

### Naming
- **Mapper file:** named after the screen in kebab-case → `<screen-kebab>.mapper.ts` (e.g. `user-profile-screen.mapper.ts`).
- **Mapper class:** `<Screen>Mapper` (e.g. `UserProfileScreenMapper`).
- **One mapper per screen**, even if the screen consumes several entities — a single mapper holds multiple methods.
- **Model files:** free base name + `.model.ts` suffix (e.g. `user-summary.model.ts`, `user-permissions.model.ts`). One screen often needs several models — let context decide each name.
- **Model class/interface:** ends in `Model` (e.g. `UserSummaryModel`).

### Method naming
Each mapping method uses the explicit `<Source>To<Target>` pattern with the real class names — never generic `toModel` / `fromModel`:
- `<Entity>DomainTo<Name>Model` — domain entity → presentation model
- `<Name>ModelTo<Entity>Domain` — presentation model → domain entity (when needed)

This avoids ambiguity when one mapper handles multiple types in both directions.

### Example
```typescript
// presentation/models/user-summary.model.ts
export interface UserSummaryModel {
  fullName: string
  initials: string
  joinedLabel: string
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
      joinedLabel: `Joined ${user.createdAt.toLocaleDateString()}`,
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

### Rules
- **Only map fields the UI actually uses.** Never spread an entity or copy fields the screen does not render.
- One mapper per screen. The file name matches the screen.
- Method names follow `<Source>To<Target>` using the real class names.
- Models are interfaces (not classes) unless they need behavior.

---

## MVVM Convention {#mvvm}

The presentation layer follows **MVVM**:
- **Model** — the Presentation Models / Form Models built by the screen's mappers, plus the module store for cross-screen state.
- **View** — the Screen component (`.vue` SFC). Passive: it renders state and forwards user intent to the ViewModel.
- **ViewModel** — the `use<Screen>ViewModel` composable. Owns UI state and orchestration; it is the **driving boundary** of the frontend hexagon (the SPA analog of a backend controller).

> MVVM is the mandatory presentation pattern for this project type. Every screen has a
> ViewModel; no screen calls a use case directly.

### ViewModel rules

- Named `use<Screen>ViewModel.ts`, lives alongside its screen. One ViewModel per screen — reusable logic goes to `composables/`, and a reusable composable must not call use cases.
- Receives route params as **function arguments** and self-initializes (`onMounted` inside the ViewModel) — the View never orchestrates loading.
- Resolves use cases from the typed Awilix container **once, at the top** of the composable.
- Builds use-case inputs as domain **Props classes** (via the Form Mapper for forms) — never object literals.
- **Unwrap point for `Result`**: use cases return `Result<T>` and never throw (adapters already converted every failure). The ViewModel checks `isErr()` and maps the `DomainException`'s `code` to a user-facing message. `try/catch` is NOT the error channel here — there is nothing to catch.
- Converts domain entities to Presentation Models via the Screen Mapper before exposing them.
- Returns **only**: reactive refs with Presentation Models, primitive UI state (`isLoading`, `error`), and action functions. Never domain entities, `Result`s, exceptions, or the container.
- The module store (Pinia) holds **cross-screen** state only; ViewModels read/write it, Views never import it.

### View (Screen) rules

- Consumes exactly what its ViewModel returns — nothing else.
- Never imports from `domain/` or `application/`, never touches the DI container, stores, or mappers.
- Local `ref`s are allowed only for pure view concerns (e.g. an accordion toggle that doesn't outlive the screen).

### Example — ViewModel

```typescript
// useUserProfileViewModel.ts
import { onMounted, ref } from 'vue'
import { container } from '@/base/config/di/container'
import type { DomainException } from '@/base/lib/domain/domain-exception.base'
import { FindUserProps } from '../../../domain/props/find-user.props'
import type { UserSummaryModel } from '../../models/user-summary.model'
import type { UserPermissionsModel } from '../../models/user-permissions.model'
import type { CreateUserFormModel } from '../../models/create-user.form-model'
import { UserProfileScreenMapper } from '../../mappers/user-profile-screen.mapper'
import { CreateUserFormMapper } from '../../mappers/create-user.form-mapper'

// Only the codes this screen can produce. Swap literals for i18n keys when needed.
const ERROR_MESSAGES: Record<string, string> = {
  USER_NOT_FOUND: 'User not found',
  USER_ALREADY_EXISTS: 'A user with this email already exists',
}

function toUiError(error: DomainException): string {
  return ERROR_MESSAGES[error.code] ?? 'Something went wrong, please try again'
}

export function useUserProfileViewModel(userId: string) {
  const userFinder = container.resolve('userFinder')
  const userCreator = container.resolve('userCreator')

  const summary = ref<UserSummaryModel | null>(null)
  const permissions = ref<UserPermissionsModel | null>(null)
  const isLoading = ref(false)
  const error = ref<string | null>(null)

  onMounted(() => loadUser(userId))

  async function loadUser(id: string) {
    isLoading.value = true
    error.value = null
    const result = await userFinder.execute(new FindUserProps(id))
    if (result.isErr()) {
      error.value = toUiError(result.getError())
    } else {
      const user = result.getValue()
      summary.value = UserProfileScreenMapper.userDomainToUserSummaryModel(user)
      permissions.value = UserProfileScreenMapper.userDomainToUserPermissionsModel(user)
    }
    isLoading.value = false
  }

  async function createUser(formData: CreateUserFormModel) {
    isLoading.value = true
    error.value = null
    const result = await userCreator.execute(CreateUserFormMapper.toProps(formData))
    if (result.isErr()) error.value = toUiError(result.getError())
    isLoading.value = false
  }

  return { summary, permissions, isLoading, error, createUser }
}
```

### Example — View

```vue
<!-- UserProfileScreen.vue -->
<script setup lang="ts">
import { useRoute } from 'vue-router'
import { useUserProfileViewModel } from './useUserProfileViewModel'

const route = useRoute()
const { summary, permissions, isLoading, error } =
  useUserProfileViewModel(route.params.id as string)
</script>

<template>
  <p v-if="isLoading">Loading…</p>
  <p v-else-if="error" role="alert">{{ error }}</p>
  <section v-else-if="summary">
    <h1>{{ summary.fullName }}</h1>
    <span>{{ summary.joinedLabel }}</span>
    <button v-if="permissions?.canEdit">Edit</button>
  </section>
</template>
```

### Testing the ViewModel

The composable uses `onMounted`, so it must run inside a component context. Use a tiny
`withSetup` helper, mock at the **use-case seam** by overriding the container
registrations with `asValue`, and assert only on what the ViewModel returns (the View's
contract) — never on internals, and never mocking axios/fetch:

```typescript
// tests/helpers/with-setup.ts
import { createApp, type App } from 'vue'

export function withSetup<T>(composable: () => T): { result: T; app: App } {
  let result!: T
  const app = createApp({
    setup() {
      result = composable()
      return () => null
    },
  })
  app.mount(document.createElement('div'))
  return { result, app }
}
```

```typescript
// tests/modules/user/presentation/screens/user-profile/useUserProfileViewModel.spec.ts
import { flushPromises } from '@vue/test-utils'
import { asValue } from 'awilix'
import { container } from '@/base/config/di/container'
import { Result } from '@/base/lib/domain/result'
import type { UserFinder } from '@/modules/user/application/use-cases/user-finder/user-finder.use-case'
import { UserNotFoundException } from '@/modules/user/domain/exceptions/user-not-found.exception'
import { useUserProfileViewModel } from '@/modules/user/presentation/screens/user-profile/useUserProfileViewModel'
import { buildUser } from '../../../builders/user.builder'
import { withSetup } from '../../../../../helpers/with-setup'

const execute = vi.fn()

beforeEach(() => {
  execute.mockReset()
  container.register({ userFinder: asValue({ execute } as unknown as UserFinder) })
})

it('exposes presentation models when the use case succeeds', async () => {
  execute.mockResolvedValue(Result.ok(buildUser()))

  const { result, app } = withSetup(() => useUserProfileViewModel('user-1'))
  await flushPromises()

  expect(result.summary.value).not.toBeNull()
  expect(result.error.value).toBeNull()
  app.unmount()
})

it('maps the DomainException code to a UI message on failure', async () => {
  execute.mockResolvedValue(Result.err(new UserNotFoundException('user-1')))

  const { result, app } = withSetup(() => useUserProfileViewModel('user-1'))
  await flushPromises()

  expect(result.error.value).toBe('User not found')
  expect(result.summary.value).toBeNull()
  app.unmount()
})
```

---

## Router Convention {#router}

Each module exports its own routes:

```typescript
// modules/user/presentation/routes/user.routes.ts
import type { RouteRecordRaw } from 'vue-router'

export const userRoutes: RouteRecordRaw = {
  path: '/users',
  children: [
    {
      path: 'profile',
      name: 'user-profile',
      component: () => import('../screens/user-profile/UserProfileScreen.vue'),
    },
  ],
}
```

---

## Layouts Convention {#layouts}

Layouts live in `modules/shared/presentation/layouts/` and wrap screens based on authentication state.

- **PublicLayout** — renders public screens (login, register, etc.). No auth check.
- **PrivateLayout** — renders private screens. Uses `usePrivateLayoutViewModel` to validate the user is authenticated before rendering. If not authenticated, redirects to login.

### PublicLayout
```vue
<!-- modules/shared/presentation/layouts/public/PublicLayout.vue -->
<script setup lang="ts">
import { RouterView } from 'vue-router'
</script>

<template>
  <div class="public-layout">
    <RouterView />
  </div>
</template>
```

### PrivateLayout + ViewModel
```typescript
// modules/shared/presentation/layouts/private/usePrivateLayoutViewModel.ts
import { ref, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { container } from '@/base/config/di/container'

export function usePrivateLayoutViewModel() {
  const router = useRouter()
  const isAuthenticated = ref(false)
  const isLoading = ref(true)

  onMounted(async () => {
    try {
      const authValidator = container.resolve('authValidator')
      const result = await authValidator.execute()

      if (result.isOk()) {  // uses manual Result class
        isAuthenticated.value = true
      } else {
        router.replace({ name: 'login' })
      }
    } catch {
      router.replace({ name: 'login' })
    } finally {
      isLoading.value = false
    }
  })

  return { isAuthenticated, isLoading }
}
```

```vue
<!-- modules/shared/presentation/layouts/private/PrivateLayout.vue -->
<script setup lang="ts">
import { RouterView } from 'vue-router'
import { usePrivateLayoutViewModel } from './usePrivateLayoutViewModel'

const { isAuthenticated, isLoading } = usePrivateLayoutViewModel()
</script>

<template>
  <div v-if="isLoading">Loading...</div>
  <div v-else-if="isAuthenticated" class="private-layout">
    <RouterView />
  </div>
</template>
```

---

## Module Example — Full User Module {#module-example}

### Source (`src/modules/user/`)
```
modules/user/
├── domain/
│   ├── user.entity.ts
│   ├── value-objects/
│   │   └── user-email.value-object.ts
│   ├── exceptions/
│   │   └── user-not-found.exception.ts
│   ├── props/
│   │   ├── create-user.props.ts
│   │   └── find-by-email.props.ts
│   └── ports/
│       └── user.repository.ts
├── application/
│   └── use-cases/
│       └── user-creator/
│           └── user-creator.use-case.ts
├── infrastructure/
│   ├── repositories/
│   │   └── http-user.repository.ts
│   └── dtos/
│       └── user-response.dto.ts
└── presentation/
    ├── screens/
    │   └── user-profile/
    │       ├── UserProfileScreen.vue
    │       └── useUserProfileViewModel.ts
    ├── components/
    │   └── UserCard.vue
    ├── routes/
    │   └── user.routes.ts
    ├── store/
    │   └── user.store.ts
    ├── composables/
    │   └── useUserPermissions.ts
    ├── mappers/
    │   ├── user-profile-screen.mapper.ts
    │   └── create-user.form-mapper.ts
    └── models/
        ├── user-summary.model.ts
        ├── user-permissions.model.ts
        └── create-user.form-model.ts
```

### Tests (`tests/modules/user/`)
```
tests/modules/user/
├── application/
│   └── use-cases/
│       └── user-creator/
│           └── user-creator.use-case.spec.ts
└── infrastructure/
    └── repositories/
        └── http-user.repository.spec.ts
```
