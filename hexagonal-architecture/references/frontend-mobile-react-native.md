# Frontend Mobile — React Native (Expo) Reference

## Table of Contents
1. [Project Setup](#setup)
2. [Project Structure](#structure)
3. [src/base Structure](#base)
4. [HTTP Client](#http-client)
5. [Presentation Layer](#presentation)
6. [DI with Awilix](#di)
7. [Props Convention](#props)
8. [Form Model & Form Mapper](#form-mapper)
9. [ViewModel Convention](#viewmodel)
10. [Navigation Convention](#navigation)
11. [Layouts Convention](#layouts)
12. [Module Example](#module-example)

---

## Project Setup {#setup}

React Native projects **always use Expo**. Do not use bare React Native CLI.

### CLI Scaffold
```bash
npx create-expo-app <project-name> --template blank-typescript
```

### Required Dependencies
```bash
npm install awilix              # dependency injection
npm install zod                 # env and DTO validation
npm install axios               # HTTP client
npm install zustand             # state management
npm install expo-constants      # environment variables
npm install expo-secure-store   # secure token storage
npm install expo-splash-screen  # splash screen control
npm install @react-navigation/native @react-navigation/native-stack
npm install react-native-screens react-native-safe-area-context

# Formatting
npm install -D prettier

# Testing
npm install -D @testing-library/react-native @testing-library/jest-dom

# Code quality
npm install -D husky lint-staged
npm install -D eslint-plugin-boundaries  # layer boundary enforcement
npx husky init
```

### Post-scaffold Steps
1. Create `src/base/` structure with config and lib folders
2. Create `src/modules/shared/` and first domain module
3. Set up Awilix container in `src/base/config/di/container.ts`
4. Set up Navigation in `src/base/config/navigation/index.tsx`
5. Set up env validation with zod in `src/base/config/env/`
6. Configure path alias `@/` → `src/` in `tsconfig.json`
7. Configure `eslint-plugin-boundaries` in ESLint config (see below)
8. Configure `husky` + `lint-staged` for pre-commit hooks (see below)

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

### Boundary Enforcement (`eslint-plugin-boundaries`)

Add the plugin and boundary rules to your ESLint flat config. This enforces the hexagonal dependency rule at lint time — violations fail the pre-commit hook.

**Frontend layers (4):** `domain` → `application` → `infrastructure` / `presentation`. Infrastructure and presentation are siblings — neither can import from the other.

```typescript
// eslint.config.ts
import boundaries from 'eslint-plugin-boundaries'

export default [
  // ... your existing React Native/TypeScript config ...
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
      'boundaries/ignore': ['**/*.spec.ts', '**/*.spec.tsx'],
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
│   │           └── private/
│   │               ├── PrivateNavigator.tsx
│   │               └── usePrivateLayoutViewModel.ts
│   └── user/
│       ├── domain/          ← entities, value-objects, exceptions, props/, ports
│       ├── application/     ← use-cases/
│       ├── infrastructure/
│       └── presentation/
└── App.tsx
```

---

## src/base Structure {#base}

```
src/base/
├── config/
│   ├── env/
│   │   └── env.config.ts          ← uses expo-constants
│   ├── http/
│   │   ├── axios.http-client.ts
│   │   ├── http-response.ts
│   │   └── exception/
│   │       └── http-service.exception.ts
│   ├── logger/
│   │   └── logger.ts
│   ├── di/
│   │   └── container.ts           ← Awilix container setup
│   └── navigation/                ← NOT router/
│       └── index.tsx              ← NavigationContainer + module screens
├── constants/
│   └── index.ts                   ← project-wide constants (export const VARIABLE_NAME = value)
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

### App.tsx entry point
```typescript
// App.tsx
import { AppNavigator } from './base/config/navigation'

export default function App() {
  return <AppNavigator />
}
```

---

## Environment Variables {#env}

Expo only exposes variables prefixed with `EXPO_PUBLIC_` to the app bundle. **Never put secrets in `EXPO_PUBLIC_*` variables** — they are bundled into the app and visible in the binary. Secrets must stay server-side.

Variables are defined in `.env` and accessed via `expo-constants` (which reads `process.env` at build time via Expo's config plugin).

**Generate all three files exactly as shown** when scaffolding a new project. Adjust variable names to match the actual project requirements.

### `.env.example` (committed to repo)
```bash
# API
EXPO_PUBLIC_API_BASE_URL=http://localhost:3000
```

### `src/base/config/env/env.schema.ts`
```typescript
import { z } from 'zod'

export const envSchema = z.object({
  EXPO_PUBLIC_API_BASE_URL: z.string().url(),
})

export type Env = z.infer<typeof envSchema>
```

### `src/base/config/env/env.config.ts`
```typescript
import Constants from 'expo-constants'
import { envSchema } from './env.schema'

// expo-constants exposes EXPO_PUBLIC_* vars via expoConfig.extra or process.env
const raw = {
  EXPO_PUBLIC_API_BASE_URL: process.env.EXPO_PUBLIC_API_BASE_URL,
}

const result = envSchema.safeParse(raw)

if (!result.success) {
  throw new Error(`Invalid environment variables:\n${result.error.toString()}`)
}

export const env = result.data
```

Usage anywhere in the app:
```typescript
import { env } from '@/base/config/env/env.config'

// env.EXPO_PUBLIC_API_BASE_URL — fully typed, validated at startup
```

Import in root entry point (`src/_layout.tsx` or `App.tsx`):
```typescript
import '@/base/config/env/env.config'   // must be first — crashes early if vars are missing
```

Rules:
- **Never** read `process.env.EXPO_PUBLIC_X` directly outside `env.config.ts`. All other code imports from `env`.
- Validation runs once at app startup and crashes clearly if any variable is missing.
- Add new variables to **all three files** at the same time: `.env.example`, `env.schema.ts`, and `env.config.ts`.
- Use `.env.local` for local overrides (not committed). Expo loads it automatically.

---

## HTTP Client {#http-client}

**Generate these files exactly as shown.** Same architecture as React SPA but uses `expo-secure-store` for token storage instead of `localStorage`.

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
import * as SecureStore from 'expo-secure-store'
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

    // Auth interceptor — attach token from SecureStore
    this.client.interceptors.request.use(async (config) => {
      const token = await SecureStore.getItemAsync('auth-token')
      if (token) {
        config.headers.Authorization = `Bearer ${token}`
      }
      return config
    })

    // Response interceptor — on 401, clear token (navigation handled by PrivateNavigator)
    this.client.interceptors.response.use(
      (response) => response,
      async (error) => {
        if (axios.isAxiosError(error) && error.response?.status === 401) {
          await SecureStore.deleteItemAsync('auth-token')
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
│   │   ├── UserProfileScreen.tsx
│   │   └── useUserProfileViewModel.ts
│   └── change-password/
│       ├── ChangePasswordScreen.tsx
│       └── useChangePasswordViewModel.ts
├── components/               ← components scoped to this module
│   └── UserCard.tsx
├── routes/
│   └── user.routes.tsx
├── store/                    ← Zustand stores for this module
│   └── user.store.ts
├── hooks/                    ← reusable hooks (not ViewModels)
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
import { HttpUserRepository } from '@/modules/user/infrastructure/repositories/http-user.repository'
import { UserCreator } from '@/modules/user/application/use-cases/user-creator/user-creator.use-case'

export const container = createContainer({ injectionMode: InjectionMode.CLASSIC })

container.register({
  // Base infrastructure
  httpClient: asClass(AxiosHttpClient).singleton(),

  // Adapters (receive httpClient via constructor injection)
  userRepository: asClass(HttpUserRepository).singleton(),

  // Use cases (receive adapters via constructor injection)
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

The use case receives the Props class and extends `UseCase`:
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

## ViewModel Convention {#viewmodel}

- Named: `use<Screen>ViewModel.ts`
- Lives alongside its screen
- Consumes use cases via Awilix container
- Returns state + actions
- Uses `try/catch` (not Result pattern)
- When submitting forms, uses a **Form Mapper** to convert the Form Model into domain Props

```typescript
// useUserProfileViewModel.ts
import { useState } from 'react'
import { container } from '@/base/config/di/container'
import type { UserSummaryModel } from '../models/user-summary.model'
import type { UserPermissionsModel } from '../models/user-permissions.model'
import type { CreateUserFormModel } from '../models/create-user.form-model'
import { UserProfileScreenMapper } from '../mappers/user-profile-screen.mapper'
import { CreateUserFormMapper } from '../mappers/create-user.form-mapper'

export function useUserProfileViewModel() {
  const [summary, setSummary] = useState<UserSummaryModel | null>(null)
  const [permissions, setPermissions] = useState<UserPermissionsModel | null>(null)
  const [isLoading, setIsLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)

  async function loadUser(id: string) {
    setIsLoading(true)
    try {
      const userFinder = container.resolve('userFinder')
      const user = await userFinder.execute({ id })
      setSummary(UserProfileScreenMapper.userDomainToUserSummaryModel(user))
      setPermissions(UserProfileScreenMapper.userDomainToUserPermissionsModel(user))
    } catch {
      setError('Failed to load user')
    } finally {
      setIsLoading(false)
    }
  }

  async function createUser(formData: CreateUserFormModel) {
    setIsLoading(true)
    try {
      const userCreator = container.resolve('userCreator')
      const props = CreateUserFormMapper.toProps(formData)
      await userCreator.execute(props)
    } catch {
      setError('Failed to create user')
    } finally {
      setIsLoading(false)
    }
  }

  return { summary, permissions, isLoading, error, loadUser, createUser }
}
```

---

## Navigation Convention {#navigation}

React Native uses React Navigation instead of React Router. Navigation config lives in `src/base/config/navigation/`.

### Navigation setup
```typescript
// src/base/config/navigation/index.tsx
import { NavigationContainer } from '@react-navigation/native'
import { createNativeStackNavigator } from '@react-navigation/native-stack'
import { userScreens } from '@/modules/user/presentation/routes/user.routes'

const Stack = createNativeStackNavigator()

export function AppNavigator() {
  return (
    <NavigationContainer>
      <Stack.Navigator>
        {userScreens.map(screen => (
          <Stack.Screen key={screen.name} {...screen} />
        ))}
      </Stack.Navigator>
    </NavigationContainer>
  )
}
```

### Module route exports
```typescript
// modules/user/presentation/routes/user.routes.tsx
import { UserProfileScreen } from '../screens/user-profile/UserProfileScreen'
import { ChangePasswordScreen } from '../screens/change-password/ChangePasswordScreen'

export const userScreens = [
  { name: 'UserProfile', component: UserProfileScreen },
  { name: 'ChangePassword', component: ChangePasswordScreen },
]
```

---

## Layouts Convention {#layouts}

In React Native, layouts work as **wrapper navigators** instead of route layouts.

- There is no `PublicLayout` — public screens are rendered directly.
- **PrivateNavigator** wraps authenticated screens. Uses `usePrivateLayoutViewModel` to validate the user is authenticated. If not authenticated, navigation is handled by the auth state.

### PrivateNavigator + ViewModel
```typescript
// modules/shared/presentation/layouts/private/usePrivateLayoutViewModel.ts
import { useState, useEffect } from 'react'
import { container } from '@/base/config/di/container'

export function usePrivateLayoutViewModel() {
  const [isAuthenticated, setIsAuthenticated] = useState(false)
  const [isLoading, setIsLoading] = useState(true)

  useEffect(() => {
    async function validateAuth() {
      try {
        const authValidator = container.resolve('authValidator')
        const result = await authValidator.execute()

        if (result.isOk()) {
          setIsAuthenticated(true)
        }
      } catch {
        // not authenticated
      } finally {
        setIsLoading(false)
      }
    }

    validateAuth()
  }, [])

  return { isAuthenticated, isLoading }
}
```

```typescript
// modules/shared/presentation/layouts/private/PrivateNavigator.tsx
import { usePrivateLayoutViewModel } from './usePrivateLayoutViewModel'

export function PrivateNavigator({ children }: { children: React.ReactNode }) {
  const { isAuthenticated, isLoading } = usePrivateLayoutViewModel()

  if (isLoading) return <LoadingScreen />
  if (!isAuthenticated) return null

  return <>{children}</>
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
    │       ├── UserProfileScreen.tsx
    │       └── useUserProfileViewModel.ts
    ├── components/
    │   └── UserCard.tsx
    ├── routes/
    │   └── user.routes.tsx
    ├── store/
    │   └── user.store.ts
    ├── hooks/
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
