# Frontend — Next.js Reference

## Table of Contents
1. [Project Setup](#setup)
2. [Key Differences vs SPAs](#differences)
3. [Project Structure](#structure)
4. [src/base Structure](#base)
5. [HTTP Client](#http-client)
6. [Presentation Layer](#presentation)
7. [DI with Awilix](#di)
8. [Props Convention](#props)
9. [Server Actions Convention](#server-actions)
10. [Route Protection](#route-protection)
11. [app/ Directory Convention](#app-dir)
12. [Module Example](#module-example)

---

## Project Setup {#setup}

### CLI Scaffold
```bash
npx create-next-app@latest <project-name> --typescript --app --src-dir --eslint --tailwind
```

### Required Dependencies
```bash
npm install awilix           # dependency injection (server-side only)
npm install zod              # env and DTO validation
npm install axios            # HTTP client (for external API calls)

# Formatting
npm install -D prettier

# Testing
npm install -D vitest @testing-library/react @testing-library/jest-dom jsdom

# Code quality
npm install -D husky lint-staged
npm install -D eslint-plugin-boundaries  # layer boundary enforcement
npx husky init
```

Note: Tailwind CSS is included by default via `--tailwind` flag in the scaffold command.

### Post-scaffold Steps
1. Create `src/base/` structure with config and lib folders (no `router/` — Next.js uses file system routing)
2. Create `src/modules/shared/` and first domain module
3. Set up Awilix container in `src/base/config/di/container.ts` (server-side only)
4. Set up env validation with zod in `src/base/config/env/`
5. Clean up default `app/page.tsx` — make it delegate to a module's Page component
6. Configure path alias `@/` → `src/` in `tsconfig.json` (Next.js does this by default with `--src-dir`)
7. Configure `vitest` with `jsdom` environment and `@/` alias (see below)
8. Configure `tsconfig` to include `tests/**/*` (see below)
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
    "*.{ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ]
  }
}
```

### Vitest + TypeScript Configuration

Tests live in `tests/` at the root (mirroring `src/`). Configure these files so that TypeScript resolves the `@/` alias and vitest finds the test files:

**`tsconfig.json`** — add `tests/**/*` to `include`:
```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src/**/*", "tests/**/*"]
}
```

**`vitest.config.ts`** — create a separate vitest config (Next.js uses `next.config.ts` for its own build):
```typescript
import { fileURLToPath, URL } from 'node:url'
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    root: '.',
    include: ['tests/**/*.spec.ts', 'tests/**/*.spec.tsx'],
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

**Frontend layers (4):** `domain` → `application` → `infrastructure` / `presentation`. Infrastructure and presentation are siblings — neither can import from the other. Note: the `app/` directory (Next.js routing wrappers) is excluded from boundary checks — it only delegates to module Page components.

```typescript
// eslint.config.mjs
import boundaries from 'eslint-plugin-boundaries'

export default [
  // ... your existing Next.js/TypeScript config ...
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
      'boundaries/ignore': ['**/*.spec.ts', '**/*.spec.tsx', 'src/app/**'],
    },
    rules: {
      'boundaries/element-types': [2, {
        default: 'disallow',
        rules: [
          {
            from: ['domain'],
            allow: [
              ['domain', { module: '${from.module}' }],
              'shared-domain',
              'base',
            ],
          },
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
          { from: ['shared-domain'], allow: ['shared-domain', 'base'] },
          { from: ['shared-app'],    allow: ['shared-domain', 'shared-app', 'base'] },
          { from: ['shared-infra'],  allow: ['shared-domain', 'shared-app', 'shared-infra', 'base'] },
          { from: ['shared-pres'],   allow: ['shared-domain', 'shared-app', 'shared-pres', 'base'] },
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
- ✅ All layers can import from `base/` and from `shared/` (respecting shared's own layer hierarchy)
- ✅ `app/` directory is excluded — it only contains thin routing wrappers

---

## Key Differences vs SPAs {#differences}

Next.js is treated as its own category — **do NOT apply MVVM or ViewModels here**.

| Concept | SPAs (Vue/React/RN) | Next.js |
|---|---|---|
| Presentation pattern | MVVM with ViewModels | Server/Client Components + Server Actions |
| Router | Vue Router / React Router / React Navigation | App Router (file system based) |
| `src/base/config/router/` | ✅ exists | ❌ does not exist — routing is file system |
| ViewModels | ✅ `useXxxViewModel.ts` | ❌ does not apply |
| Server Actions | ❌ | ✅ inside `presentation/screens/<screen>/actions/` |
| `app/` directory | ❌ | ✅ entry point only — delegates to modules |

---

## Project Structure {#structure}

```
app/                           ← Next.js App Router entry points ONLY
│                                 Each page.tsx imports from src/modules
├── (auth)/
│   └── login/
│       └── page.tsx           ← import { LoginPage } from '@/modules/auth/presentation/...'
├── users/
│   └── [id]/
│       └── page.tsx           ← import { UserProfilePage } from '@/modules/user/presentation/...'
└── layout.tsx

src/
├── base/
├── modules/
│   ├── shared/
│   │   ├── domain/
│   │   ├── application/
│   │   ├── infrastructure/
│   │   └── presentation/
│   │       ├── components/
│   │       └── design-tokens/
│   └── user/
│       ├── domain/          ← entities, value-objects, exceptions, props/, ports
│       ├── application/     ← use-cases/
│       ├── infrastructure/
│       └── presentation/
```

---

## src/base Structure {#base}

```
src/base/
├── config/
│   ├── env/
│   │   ├── env.config.ts      ← imported in layout.tsx or instrumentation.ts
│   │   └── env.schema.ts      ← zod schema
│   ├── http/
│   │   ├── axios.http-client.ts
│   │   ├── http-response.ts
│   │   └── exception/
│   │       └── http-service.exception.ts
│   ├── logger/
│   │   └── logger.ts
│   └── di/
│       └── container.ts       ← Awilix container (used in Server Components / Actions)
├── constants/
│   └── index.ts               ← project-wide constants (export const VARIABLE_NAME = value)
└── lib/
    ├── domain/
    │   ├── result.ts              ← manual Result<T> class
    │   ├── props.base.ts          ← abstract class Props
    │   ├── domain-exception.base.ts   ← abstract class DomainException
    │   └── value-object.base.ts
    ├── application/
    │   └── use-case.base.ts
    └── utils/                    ← pure functions grouped by concern
        ├── index.ts              ← re-exports all utils
        ├── date.utils.ts
        ├── string.utils.ts
        └── ...
```

Note: **no `router/` in `src/base/config/`** — Next.js routing is file system based via `app/`.

---

## Environment Variables {#env}

Next.js has two categories of env vars:
- **`NEXT_PUBLIC_*`** — bundled into the client. Safe for public config (API URLs, feature flags). **Never secrets.**
- **No prefix** — server-only. Only available in Server Components, Server Actions, Route Handlers, and `instrumentation.ts`. Never sent to the browser.

**Generate all three files exactly as shown** when scaffolding a new project. Adjust variable names to match the actual project requirements.

### `.env.example` (committed to repo)
```bash
# Public (client + server)
NEXT_PUBLIC_API_BASE_URL=http://localhost:3000

# Server-only (never exposed to client)
API_SECRET_KEY=
DATABASE_URL=postgresql://localhost:5432/myapp
JWT_SECRET=
```

### `src/base/config/env/env.schema.ts`
```typescript
import { z } from 'zod'

// Client-accessible vars (NEXT_PUBLIC_*)
export const clientEnvSchema = z.object({
  NEXT_PUBLIC_API_BASE_URL: z.string().url(),
})

// Server-only vars — only validated and used server-side
export const serverEnvSchema = z.object({
  API_SECRET_KEY: z.string().min(1),
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
})

export const envSchema = clientEnvSchema.merge(serverEnvSchema)

export type ClientEnv = z.infer<typeof clientEnvSchema>
export type ServerEnv = z.infer<typeof serverEnvSchema>
export type Env = z.infer<typeof envSchema>
```

### `src/base/config/env/env.config.ts`
```typescript
import { envSchema, clientEnvSchema } from './env.schema'

// Server-side validation (full schema — runs in Node.js)
function validateServerEnv() {
  const result = envSchema.safeParse(process.env)
  if (!result.success) {
    throw new Error(`Invalid environment variables:\n${result.error.toString()}`)
  }
  return result.data
}

// Client-side validation (public vars only — runs in browser via NEXT_PUBLIC_*)
function validateClientEnv() {
  const result = clientEnvSchema.safeParse({
    NEXT_PUBLIC_API_BASE_URL: process.env.NEXT_PUBLIC_API_BASE_URL,
  })
  if (!result.success) {
    throw new Error(`Invalid public environment variables:\n${result.error.toString()}`)
  }
  return result.data
}

export const env = typeof window === 'undefined'
  ? validateServerEnv()
  : validateClientEnv()
```

Usage anywhere in the app:
```typescript
import { env } from '@/base/config/env/env.config'

// Server Component / Server Action:
// env.DATABASE_URL, env.JWT_SECRET, env.NEXT_PUBLIC_API_BASE_URL

// Client Component:
// env.NEXT_PUBLIC_API_BASE_URL  ← only public vars available
```

Register in `src/instrumentation.ts` (runs once at server startup in Next.js 15+):
```typescript
// src/instrumentation.ts (root of src/)
export async function register() {
  const { env } = await import('@/base/config/env/env.config')
  // Validation already ran via the import — crashes at startup if vars are missing
}
```

Rules:
- **Never** read `process.env.X` directly outside `env.config.ts`. All other code imports from `env`.
- Server-only vars must **never** be passed as props to Client Components — pass only what the client needs.
- `instrumentation.ts` ensures server-side validation runs before any request is handled.
- Add new variables to **all three files** at the same time: `.env.example`, `env.schema.ts`, and `env.config.ts`.

---

## HTTP Client {#http-client}

**Generate these files exactly as shown.** Next.js HTTP client is server-side. It is an injectable class registered in the Awilix container. The auth token is passed at construction time from cookies.

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
  constructor(
    public readonly statusCode: number,
    public readonly message: string,
    public readonly code?: string,
  ) {
    super()
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

  constructor(authToken?: string) {
    this.client = axios.create({
      baseURL: API_BASE_URL,
      timeout: HTTP_TIMEOUT_MS,
    })

    if (authToken) {
      this.client.defaults.headers.common.Authorization = `Bearer ${authToken}`
    }
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
In Next.js the container is **scoped per request** — the auth token is read from cookies and passed to the `AxiosHttpClient` constructor:

```typescript
// src/base/config/di/container.ts
import { createContainer, asClass, asValue, InjectionMode } from 'awilix'
import { AxiosHttpClient } from '@/base/config/http/axios.http-client'

export function createRequestContainer(authToken?: string) {
  const container = createContainer({ injectionMode: InjectionMode.CLASSIC })

  container.register({
    httpClient: asValue(new AxiosHttpClient(authToken)),
  })

  return container
}
```

### Usage in Server Components / Server Actions
```typescript
import { cookies } from 'next/headers'
import { createRequestContainer } from '@/base/config/di/container'

const token = cookies().get('auth-token')?.value
const container = createRequestContainer(token)
const userFinder = container.resolve('userFinder')
const result = await userFinder.execute(new FindUserProps(userId))
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
      return Result.err(new UserAlreadyExistsException(httpError.message))
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
│   │   ├── UserProfilePage.tsx         ← Server Component (default)
│   │   ├── UserProfileClient.tsx       ← Client Component ('use client') if needed
│   │   └── actions/
│   │       └── update-user.action.ts   ← Server Actions
│   └── change-password/
│       ├── ChangePasswordPage.tsx
│       ├── ChangePasswordClient.tsx
│       └── actions/
│           └── change-password.action.ts
├── components/                         ← components scoped to this module
│   └── UserCard.tsx
├── mappers/                            ← one mapper per page, domain ↔ presentation models
│   ├── user-profile-page.mapper.ts
│   ├── change-password-page.mapper.ts
│   └── create-user.form-mapper.ts
└── models/                             ← presentation-specific interfaces (free name + Model suffix)
    ├── user-summary.model.ts
    ├── user-permissions.model.ts
    └── create-user.form-model.ts
```

Notes:
- No `routes/` folder — routing lives in `app/`
- No `store/` folder — Server Components fetch data directly; Client Components use React state or context
- No `hooks/` folder — ViewModels do not apply; hooks only inside Client Components when needed

---

## DI with Awilix {#di}

In Next.js, Awilix is used in **Server Components and Server Actions** (server-side only):

```typescript
// src/base/config/di/container.ts
import { createContainer, asClass, InjectionMode } from 'awilix'
import { HttpUserRepository } from '@/modules/user/infrastructure/repositories/http-user.repository'
import { UserCreator } from '@/modules/user/application/use-cases/user-creator/user-creator.use-case'

export const container = createContainer({ injectionMode: InjectionMode.CLASSIC })

container.register({
  userRepository: asClass(HttpUserRepository).singleton(),
  userCreator: asClass(UserCreator).singleton(),
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

## Page Mapper & Presentation Model {#page-mapper}

In Next.js the Server Component (the `<Page>` component) is the equivalent of a "screen" in SPAs. Every page that consumes domain entities defines a **Page Mapper** that converts those entities into one or more **Presentation Models** tailored to that page's UI.

### Naming
- **Mapper file:** named after the page in kebab-case → `<page-kebab>.mapper.ts` (e.g. `user-profile-page.mapper.ts`).
- **Mapper class:** `<Page>Mapper` (e.g. `UserProfilePageMapper`).
- **One mapper per page**, even if the page consumes several entities — a single mapper holds multiple methods.
- **Model files:** free base name + `.model.ts` suffix (e.g. `user-summary.model.ts`, `user-permissions.model.ts`). One page often needs several models — let context decide each name.
- **Model class/interface:** ends in `Model` (e.g. `UserSummaryModel`).

### Method naming
Each mapping method uses the explicit `<Source>To<Target>` pattern with the real class names — never generic `toModel` / `fromModel`:
- `<Entity>DomainTo<Name>Model` — domain entity → presentation model
- `<Name>ModelTo<Entity>Domain` — presentation model → domain entity (when needed)

### Example
```typescript
// presentation/models/user-summary.model.ts
export interface UserSummaryModel {
  id: string
  fullName: string
  initials: string
  joinedLabel: string
}

// presentation/models/user-permissions.model.ts
export interface UserPermissionsModel {
  canEdit: boolean
  canDelete: boolean
}

// presentation/mappers/user-profile-page.mapper.ts
import type { User } from '@/modules/user/domain/user.entity'
import type { UserSummaryModel } from '../models/user-summary.model'
import type { UserPermissionsModel } from '../models/user-permissions.model'

export class UserProfilePageMapper {
  static userDomainToUserSummaryModel(user: User): UserSummaryModel {
    return {
      id: user.id,
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
- **Only map fields the UI actually uses.** Never spread an entity or copy fields the page does not render.
- One mapper per page. The file name matches the page.
- Method names follow `<Source>To<Target>` using the real class names.
- Models are interfaces (not classes) unless they need behavior.
- The Server Component (`<Page>`) calls the mapper before passing data to its Client Component.

---

## Server Actions Convention {#server-actions}

- Server Actions live in `presentation/screens/<screen>/actions/<action>.action.ts`
- Marked with `'use server'`
- Resolve use cases from Awilix container
- Use `try/catch` — do NOT use Result pattern here
- When receiving form data, use a **Form Mapper** to convert it into domain Props

```typescript
// modules/user/presentation/screens/user-profile/actions/create-user.action.ts
'use server'

import { container } from '@/base/config/di/container'
import { revalidatePath } from 'next/cache'
import { CreateUserFormMapper } from '../../mappers/create-user.form-mapper'
import type { CreateUserFormModel } from '../../models/create-user.form-model'

export async function createUserAction(formData: CreateUserFormModel) {
  try {
    const userCreator = container.resolve('userCreator')
    const props = CreateUserFormMapper.toProps(formData)
    await userCreator.execute(props)
    revalidatePath('/users')
  } catch (error) {
    return { error: 'Failed to create user' }
  }
}
```

---

## Route Protection {#route-protection}

Next.js does **not** use PublicLayout/PrivateLayout like SPAs. Instead, use **Next.js Middleware** to protect routes:

```typescript
// middleware.ts (root of the project, next to app/)
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

const publicPaths = ['/login', '/register', '/forgot-password']

export function middleware(request: NextRequest) {
  const token = request.cookies.get('auth-token')?.value
  const isPublicPath = publicPaths.some(path => request.nextUrl.pathname.startsWith(path))

  if (!token && !isPublicPath) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  if (token && isPublicPath) {
    return NextResponse.redirect(new URL('/', request.url))
  }

  return NextResponse.next()
}

export const config = {
  matcher: ['/((?!api|_next/static|_next/image|favicon.ico).*)'],
}
```

Notes:
- No `layouts/` folder in `shared/presentation/` — middleware handles auth at the edge
- Middleware runs **before** the page renders, so there is no loading/flash state
- The `matcher` excludes API routes and static assets

---

## app/ Directory Convention {#app-dir}

The `app/` directory contains **only routing wrappers** — all logic lives in `src/modules/`:

```typescript
// app/users/[id]/page.tsx
import { UserProfilePage } from '@/modules/user/presentation/screens/user-profile/UserProfilePage'

export default function Page({ params }: { params: { id: string } }) {
  return <UserProfilePage userId={params.id} />
}
```

```typescript
// modules/user/presentation/screens/user-profile/UserProfilePage.tsx
import { container } from '@/base/config/di/container'
import { UserProfilePageMapper } from '../../mappers/user-profile-page.mapper'
import { UserProfileClient } from './UserProfileClient'

export async function UserProfilePage({ userId }: { userId: string }) {
  const userFinder = container.resolve('userFinder')
  const user = await userFinder.execute({ id: userId })

  const summary = UserProfilePageMapper.userDomainToUserSummaryModel(user)
  const permissions = UserProfilePageMapper.userDomainToUserPermissionsModel(user)

  return <UserProfileClient summary={summary} permissions={permissions} />
}
```

```typescript
// modules/user/presentation/screens/user-profile/UserProfileClient.tsx
'use client'

import { updateUserAction } from './actions/update-user.action'
import type { UserSummaryModel } from '../../models/user-summary.model'
import type { UserPermissionsModel } from '../../models/user-permissions.model'

export function UserProfileClient({
  summary,
  permissions,
}: {
  summary: UserSummaryModel
  permissions: UserPermissionsModel
}) {
  return (
    <form action={updateUserAction}>
      <input name="id" type="hidden" value={summary.id} />
      <input name="fullName" defaultValue={summary.fullName} />
      {permissions.canDelete && <button type="button">Delete</button>}
      <button type="submit">Save</button>
    </form>
  )
}
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
    │       ├── UserProfilePage.tsx
    │       ├── UserProfileClient.tsx
    │       └── actions/
    │           └── update-user.action.ts
    ├── components/
    │   └── UserCard.tsx
    ├── mappers/
    │   ├── user-profile-page.mapper.ts
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
