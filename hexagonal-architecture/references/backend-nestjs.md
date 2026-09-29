# NestJS Reference — Hexagonal Architecture

## Table of Contents

1. [Project Setup](#setup)
2. [Monolith Structure](#monolith)
3. [Microservice Structure](#microservice)
4. [src/base Structure](#base)
5. [HTTP Client](#http-client)
6. [Naming & DI Tokens](#naming)
7. [Domain Exception Filter](#domain-exception-filter)
8. [Module Example](#module-example)

---

## Project Setup {#setup}

### CLI Scaffold

```bash
# Monolith
npx @nestjs/cli new <project-name> --strict --package-manager npm

# Microservice (same CLI, single domain structure)
npx @nestjs/cli new <project-name> --strict --package-manager npm
```

### Required Dependencies

```bash
npm install @nestjs/config zod         # env validation
npm install winston nest-winston       # logging
npm install @nestjs/mongoose mongoose  # if using MongoDB
npm install @nestjs/terminus           # healthcheck
npm install amqplib @nestjs/microservices  # messaging (RabbitMQ by default)

# Production essentials
npm install @nestjs/swagger swagger-ui-express  # API documentation
npm install helmet                     # security headers
npm install @nestjs/throttler          # rate limiting
npm install class-validator class-transformer   # DTO validation at controller level
npm install compression                # response compression

# Code quality
npm install -D husky lint-staged
npm install -D eslint-plugin-boundaries  # layer boundary enforcement
npx husky init
```

### Post-scaffold Steps

1. Remove the default `src/app.controller.ts`, `src/app.service.ts`, and `src/app.module.ts` (for monolith)
2. Create `src/base/` structure (see below)
3. Create `src/modules/` (monolith) or `src/app/` (microservice)
4. Register `DomainExceptionFilter` globally in `main.ts`
5. Register `helmet`, `compression`, and `ThrottlerGuard` globally in `main.ts`
6. Set up Swagger in `main.ts`
7. Set up env validation with zod in `src/base/config/env/`
8. Configure `eslint-plugin-boundaries` in ESLint config (see below)
9. Configure `husky` + `lint-staged` for pre-commit hooks (see below)

### Husky + lint-staged Configuration

After `npx husky init`, replace `.husky/pre-commit` contents:

```bash
npx lint-staged
```

Add to `package.json`:

```json
{
  "lint-staged": {
    "*.ts": ["eslint --fix", "prettier --write"]
  }
}
```

### Boundary Enforcement (`eslint-plugin-boundaries`)

Add the plugin and boundary rules to your ESLint flat config. This enforces the hexagonal dependency rule at lint time — violations fail the pre-commit hook.

**Backend layers (3):** `domain` → `application` → `infrastructure` (with `driving/` and `driven/`). The infrastructure layer is split into two sub-zones so that driving and driven adapters cannot import from each other.

```typescript
// eslint.config.mjs
import boundaries from "eslint-plugin-boundaries";

export default [
  // ... your existing NestJS/TypeScript config ...
  {
    plugins: { boundaries },
    settings: {
      "boundaries/elements": [
        // Module layers (monolith: src/modules/<module>/<layer>, microservice: src/app/<layer>)
        {
          type: "domain",
          pattern: ["src/modules/*/domain/**", "src/app/domain/**"],
          capture: ["module"],
        },
        {
          type: "application",
          pattern: ["src/modules/*/application/**", "src/app/application/**"],
          capture: ["module"],
        },
        {
          type: "driving",
          pattern: [
            "src/modules/*/infrastructure/driving/**",
            "src/app/infrastructure/driving/**",
          ],
          capture: ["module"],
        },
        {
          type: "driven",
          pattern: [
            "src/modules/*/infrastructure/driven/**",
            "src/app/infrastructure/driven/**",
          ],
          capture: ["module"],
        },
        // Cross-cutting
        { type: "shared-domain", pattern: ["src/modules/shared/domain/**"] },
        { type: "shared-app", pattern: ["src/modules/shared/application/**"] },
        {
          type: "shared-infra",
          pattern: ["src/modules/shared/infrastructure/**"],
        },
        { type: "base", pattern: ["src/base/**"] },
      ],
      "boundaries/ignore": ["**/*.spec.ts", "**/*.e2e-spec.ts"],
    },
    rules: {
      "boundaries/element-types": [
        2,
        {
          default: "disallow",
          rules: [
            // domain → only same-module domain, shared-domain, and base
            {
              from: ["domain"],
              allow: [
                ["domain", { module: "${from.module}" }],
                "shared-domain",
                "base",
              ],
            },
            // application → domain + application (same module), shared-domain, shared-app, base
            {
              from: ["application"],
              allow: [
                ["domain", { module: "${from.module}" }],
                ["application", { module: "${from.module}" }],
                "shared-domain",
                "shared-app",
                "base",
              ],
            },
            // driving (controllers) → domain + application (same module), shared-*, base
            {
              from: ["driving"],
              allow: [
                ["domain", { module: "${from.module}" }],
                ["application", { module: "${from.module}" }],
                ["driving", { module: "${from.module}" }],
                "shared-domain",
                "shared-app",
                "shared-infra",
                "base",
              ],
            },
            // driven (repositories, publishers) → domain + application (same module), shared-*, base
            {
              from: ["driven"],
              allow: [
                ["domain", { module: "${from.module}" }],
                ["application", { module: "${from.module}" }],
                ["driven", { module: "${from.module}" }],
                "shared-domain",
                "shared-app",
                "shared-infra",
                "base",
              ],
            },
            // shared layers follow the same inward rule
            { from: ["shared-domain"], allow: ["shared-domain", "base"] },
            {
              from: ["shared-app"],
              allow: ["shared-domain", "shared-app", "base"],
            },
            {
              from: ["shared-infra"],
              allow: ["shared-domain", "shared-app", "shared-infra", "base"],
            },
            // base can import from base only
            { from: ["base"], allow: ["base"] },
          ],
        },
      ],
      "boundaries/no-unknown": [2],
    },
  },
];
```

What this enforces:

- ❌ `domain/` importing from `application/`, `infrastructure/`, or another module
- ❌ `application/` importing from `infrastructure/` or another module
- ❌ `driving/` importing from `driven/` (and vice-versa)
- ❌ Cross-module imports (e.g. `modules/user/` importing from `modules/order/`)
- ✅ All layers can import from `base/`
- ✅ All layers can import from `shared/` (respecting shared's own layer hierarchy)

---

## Monolith Structure {#monolith}

```
src/
├── base/                          ← see src/base section below
├── modules/
│   ├── shared/
│   │   ├── domain/
│   │   │   ├── value-objects/
│   │   │   ├── exceptions/
│   │   │   │   └── domain.exception.ts
│   │   │   └── ports/
│   │   │       └── event-bus.port.ts
│   │   ├── application/
│   │   └── infrastructure/
│   │       └── config/
│   └── user/
│       ├── domain/
│       │   ├── user.entity.ts
│       │   ├── value-objects/
│       │   │   └── user-email.value-object.ts
│       │   ├── exceptions/
│       │   │   ├── user-not-found.exception.ts
│       │   │   └── user-already-exists.exception.ts
│       │   ├── props/
│       │   │   ├── create-user.command.ts
│       │   │   ├── find-user.query.ts
│       │   │   └── find-by-email.props.ts
│       │   ├── events/
│       │   │   └── user-created.event.ts
│       │   └── ports/
│       │       ├── user.repository.ts
│       │       └── event-bus.port.ts
│       ├── application/
│       │   ├── use-cases/
│       │   │   └── user-creator/
│       │   │       ├── user-creator.use-case.ts
│       │   │       ├── user-creator.output.ts      ← UserCreatorOutput extends Output
│       │   │       └── mapper/
│       │   │           └── user-creator.mapper.ts   ← domain entity → Output
│       │   └── user.module.ts
│       └── infrastructure/
│           ├── driven/
│           │   └── persistence/
│           │       ├── mongo-user.repository.ts
│           │       └── schemas/
│           │           └── user.schema.ts
│           └── driving/
│               └── http/
│                   ├── create-user/
│                   │   ├── dto/
│                   │   │   └── create-user.request.dto.ts
│                   │   ├── mapper/
│                   │   │   └── create-user.mapper.ts   ← request DTO → Command
│                   │   └── create-user.http.controller.ts
│                   └── find-all-users/
│                       ├── dto/
│                       │   └── find-all-users.request.dto.ts
│                       ├── mapper/
│                       │   └── find-all-users.mapper.ts
│                       └── find-all-users.http.controller.ts
└── main.ts
```

---

## Microservice Structure {#microservice}

Single domain — no `modules/` wrapper, no `shared/`:

```
src/
├── base/
├── app/
│   ├── domain/
│   │   ├── order.entity.ts
│   │   ├── value-objects/
│   │   ├── exceptions/
│   │   ├── props/
│   │   │   └── create-order.command.ts
│   │   └── ports/
│   │       └── order.repository.ts
│   ├── application/
│   │   ├── use-cases/
│   │   │   └── order-creator/
│   │   │       ├── order-creator.use-case.ts
│   │   │       ├── order-creator.output.ts      ← OrderCreatorOutput extends Output
│   │   │       └── mapper/
│   │   │           └── order-creator.mapper.ts   ← domain entity → Output
│   │   └── app.module.ts
│   └── infrastructure/
│       ├── driven/
│       │   ├── persistence/
│       │   │   ├── mongo-order.repository.ts
│       │   │   └── schemas/
│       │   │       └── order.schema.ts
│       │   └── messaging/
│       │       └── publishers/
│       │           └── rabbit-event-bus.publisher.ts
│       └── driving/
│           ├── http/
│           │   └── create-order/
│           │       ├── dto/
│           │       │   └── create-order.request.dto.ts
│           │       ├── mapper/
│           │       │   └── create-order.mapper.ts   ← request DTO → Command
│           │       └── create-order.http.controller.ts
│           └── rpc/
│               └── product-created/
│                   ├── dto/
│                   │   └── product-created.request.dto.ts
│                   ├── mapper/
│                   │   └── product-created.mapper.ts
│                   └── product-created.rpc.controller.ts
└── main.ts
```

---

## src/base Structure {#base}

```
src/base/
├── config/
│   ├── context/
│   │   ├── correlation-id.middleware.ts
│   │   └── request-context.ts
│   ├── env/
│   │   ├── env-vars.module.ts
│   │   ├── env-vars.schema.ts
│   │   └── env-vars.service.ts
│   ├── http/
│   │   ├── axios.http-client.ts
│   │   ├── http-client.module.ts
│   │   ├── http-response.ts
│   │   └── exception/
│   │       └── downstream-service-error.exception.ts
│   ├── logger/
│   │   ├── logger.di-tokens.ts
│   │   ├── logger.module.ts
│   │   ├── winston.config.ts
│   │   └── winston.logger.adapter.ts
│   ├── messaging/
│   │   ├── message-deduplication.store.ts
│   │   ├── messaging.di-tokens.ts
│   │   ├── messaging.module.ts
│   │   ├── rabbitmq.constants.ts
│   │   ├── rabbitmq.messaging-client.ts
│   │   └── rabbitmq.server.ts
│   ├── nestjs/
│   │   ├── custom.exception-filter.ts
│   │   ├── domain-exception.filter.ts
│   │   └── logger.interceptor.ts
│   └── routes/
│       └── app.routes.ts
├── constants/
│   └── index.ts               ← project-wide constants (export const VARIABLE_NAME = value)
├── health/
│   ├── external-service.health.ts
│   ├── health.controller.ts
│   └── health.module.ts
└── lib/
    ├── domain/
    │   ├── command.base.ts        ← abstract class Command
    │   ├── query.base.ts          ← abstract class Query
    │   ├── props.base.ts          ← abstract class Props
    │   ├── domain-exception.base.ts   ← abstract class DomainException extends Error (abstract `code`)
    │   ├── event.base.ts         ← abstract class Event + EventMetadata interface
    │   ├── output.base.ts        ← abstract class Output (use-case return base)
    │   ├── value-object.base.ts
    │   ├── logger.port.ts
    │   ├── message.base.ts
    │   └── paginated.ts   ← class Paginated<T>
    ├── application/
    │   └── use-case.base.ts
    ├── infrastructure/
    │   ├── single-response.ts        ← class SingleResponse<T>
    │   └── paginated-response.ts     ← class PaginatedResponse<T> + PaginationMeta
    └── utils/                    ← pure functions grouped by concern
        ├── index.ts              ← re-exports all utils
        ├── date.utils.ts
        ├── string.utils.ts
        ├── security.utils.ts     ← obfuscatePrivateBodyProperties, etc.
        └── ...
```

---

## Environment Variables {#env}

**Generate all three files exactly as shown** when scaffolding a new project. Adjust variable names to match the actual project requirements.

### `.env.example` (committed to repo)

```bash
# Application
NODE_ENV=development
PORT=3000

# Database
DATABASE_URL=mongodb://localhost:27017/myapp

# External services
EXTERNAL_SERVICE_API_URL=https://api.example.com
EXTERNAL_SERVICE_API_KEY=

# HTTP client
HTTP_TIMEOUT_MS=5000
HTTP_RETRY_ATTEMPTS=3

# Auth
JWT_SECRET=
JWT_EXPIRES_IN=7d

# Messaging (RabbitMQ)
RABBITMQ_URL=amqp://localhost:5672
```

### `src/base/config/env/env-vars.schema.ts`

```typescript
import { z } from "zod";

export const envVarsSchema = z.object({
  NODE_ENV: z
    .enum(["development", "production", "test"])
    .default("development"),
  PORT: z.coerce.number().default(3000),

  DATABASE_URL: z.string().url(),

  EXTERNAL_SERVICE_API_URL: z.string().url(),
  EXTERNAL_SERVICE_API_KEY: z.string().min(1),

  HTTP_TIMEOUT_MS: z.coerce.number().default(5000),
  HTTP_RETRY_ATTEMPTS: z.coerce.number().default(3),

  JWT_SECRET: z.string().min(32),
  JWT_EXPIRES_IN: z.string().default("7d"),

  RABBITMQ_URL: z.string().url(),
});

export type EnvVars = z.infer<typeof envVarsSchema>;
```

### `src/base/config/env/env-vars.service.ts`

```typescript
import { Injectable } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";
import type { EnvVars } from "./env-vars.schema";

@Injectable()
export class EnvVarsService {
  constructor(private readonly config: ConfigService<EnvVars, true>) {}

  get nodeEnv() {
    return this.config.get("NODE_ENV", { infer: true });
  }
  get port() {
    return this.config.get("PORT", { infer: true });
  }
  get databaseUrl() {
    return this.config.get("DATABASE_URL", { infer: true });
  }
  get externalServiceApiUrl() {
    return this.config.get("EXTERNAL_SERVICE_API_URL", { infer: true });
  }
  get externalServiceApiKey() {
    return this.config.get("EXTERNAL_SERVICE_API_KEY", { infer: true });
  }
  get httpTimeoutMs() {
    return this.config.get("HTTP_TIMEOUT_MS", { infer: true });
  }
  get httpRetryAttempts() {
    return this.config.get("HTTP_RETRY_ATTEMPTS", { infer: true });
  }
  get jwtSecret() {
    return this.config.get("JWT_SECRET", { infer: true });
  }
  get jwtExpiresIn() {
    return this.config.get("JWT_EXPIRES_IN", { infer: true });
  }
  get rabbitmqUrl() {
    return this.config.get("RABBITMQ_URL", { infer: true });
  }
}
```

### `src/base/config/env/env-vars.module.ts`

```typescript
import { Global, Module } from "@nestjs/common";
import { ConfigModule } from "@nestjs/config";
import { envVarsSchema } from "./env-vars.schema";
import { EnvVarsService } from "./env-vars.service";

@Global()
@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      validate: (config) => {
        const result = envVarsSchema.safeParse(config);
        if (!result.success) {
          throw new Error(
            `Invalid environment variables:\n${result.error.toString()}`,
          );
        }
        return result.data;
      },
    }),
  ],
  providers: [EnvVarsService],
  exports: [EnvVarsService],
})
export class EnvVarsModule {}
```

Register in `AppModule` (or the root module):

```typescript
import { EnvVarsModule } from './base/config/env/env-vars.module'

@Module({
  imports: [EnvVarsModule, ...],
})
export class AppModule {}
```

Rules:

- **Never** read `process.env.VAR` directly outside of `EnvVarsService`. All other code injects `EnvVarsService`.
- If validation fails at startup, the app crashes with a clear message listing the missing/invalid vars.
- `@Global()` + `exports: [EnvVarsService]` makes it available everywhere without importing `EnvVarsModule` per module.
- Add new variables to **all three files** at the same time: `env-vars.schema.ts`, `env-vars.service.ts`, and `.env.example`.

---

## HTTP Client {#http-client}

**Generate these files exactly as shown.** This is the production-grade HTTP client used across all NestJS projects.

### `src/base/config/http/http-response.ts`

```typescript
export type HttpHeaders = Record<string, string>;

export interface HttpResponse<T> {
  data: T;
  status: number;
  headers: HttpHeaders;
}
```

### `src/base/config/http/exception/downstream-service-error.exception.ts`

```typescript
import { DomainException } from "@/base/lib/domain/domain-exception.base";

export class DownstreamServiceErrorException extends DomainException {
  readonly code = "DOWNSTREAM_SERVICE_ERROR";

  constructor(
    public readonly statusCode: number,
    message: string,
    public readonly downstreamCode?: string,
  ) {
    super(message);
  }

  static fromAxiosError(error: unknown): DownstreamServiceErrorException {
    if (axios.isAxiosError(error)) {
      return new DownstreamServiceErrorException(
        error.response?.status ?? 500,
        error.response?.data?.message ?? error.message,
        error.code,
      );
    }
    return new DownstreamServiceErrorException(500, "Unexpected error");
  }
}
```

### `src/base/config/http/axios.http-client.ts`

```typescript
import type { AxiosInstance, AxiosRequestConfig, AxiosError } from "axios";
import axios from "axios";
import { Inject, Injectable } from "@nestjs/common";

import { EnvVarsService } from "../env/env-vars.service";
import { obfuscatePrivateBodyProperties } from "@/base/lib/utils";
import { PRIVATE_BODY_PROPERTIES } from "@/base/constants";
import type { HttpResponse, HttpHeaders } from "./http-response";
import { DownstreamServiceErrorException } from "./exception/downstream-service-error.exception";
import type { LoggerPort } from "@/base/lib/domain/logger.port";
import { getRequestContext } from "../context/request-context";
import { LOGGER_ADAPTER } from "../logger/logger.di-tokens";

@Injectable()
export class AxiosHttpClient {
  private readonly http: AxiosInstance;
  private readonly retryAttempts: number;

  constructor(
    envVarsService: EnvVarsService,
    @Inject(LOGGER_ADAPTER) private readonly logger: LoggerPort,
  ) {
    this.logger.setContext(AxiosHttpClient.name);
    this.retryAttempts = envVarsService.httpRetryAttempts;
    this.http = axios.create({
      baseURL: envVarsService.externalServiceApiUrl,
      timeout: envVarsService.httpTimeoutMs,
    });
  }

  async get<T>(
    url: string,
    config?: AxiosRequestConfig<T>,
  ): Promise<HttpResponse<T>> {
    return this.request<T>({ method: "GET", url, ...config });
  }

  async post<T>(
    url: string,
    config?: AxiosRequestConfig<T>,
  ): Promise<HttpResponse<T>> {
    return this.request<T>({ method: "POST", url, ...config });
  }

  async put<T>(
    url: string,
    config?: AxiosRequestConfig<T>,
  ): Promise<HttpResponse<T>> {
    return this.request<T>({ method: "PUT", url, ...config });
  }

  async patch<T>(
    url: string,
    config?: AxiosRequestConfig<T>,
  ): Promise<HttpResponse<T>> {
    return this.request<T>({ method: "PATCH", url, ...config });
  }

  async delete<T>(
    url: string,
    config?: AxiosRequestConfig<T>,
  ): Promise<HttpResponse<T>> {
    return this.request<T>({ method: "DELETE", url, ...config });
  }

  private async request<T>(
    axiosConfig: AxiosRequestConfig<T>,
  ): Promise<HttpResponse<T>> {
    const correlationId = getRequestContext()?.correlationId;
    const requestAt = Date.now();
    const requestData = {
      baseURL: axiosConfig.baseURL,
      url: axiosConfig.url,
      method: axiosConfig.method,
      headers: { ...axiosConfig.headers, "x-correlation-id": correlationId },
      params: axiosConfig.params,
      body: obfuscatePrivateBodyProperties(
        axiosConfig.data,
        PRIVATE_BODY_PROPERTIES,
      ),
      requestAt,
    };

    let lastError: unknown;

    for (let attempt = 0; attempt <= this.retryAttempts; attempt++) {
      try {
        if (attempt > 0) {
          const delay = Math.min(1000 * Math.pow(2, attempt - 1), 10000);
          this.logger.warn(
            `Retrying HTTP ${axiosConfig.method} ${axiosConfig.url} (attempt ${attempt}/${this.retryAttempts}) after ${delay}ms`,
          );
          await new Promise((resolve) => setTimeout(resolve, delay));
        }

        const response = await this.http.request<T>({
          ...axiosConfig,
          headers: {
            ...axiosConfig.headers,
            "x-correlation-id": correlationId,
          },
        });
        const responseAt = Date.now();

        this.logger.log(
          `HTTP ${axiosConfig.method} ${axiosConfig.url} successful`,
          {
            requestData,
            responseData: { statusCode: response.status, responseAt },
            duration: responseAt - requestAt,
            success: true,
          },
        );

        return {
          data: response.data,
          status: response.status,
          headers: response.headers as HttpHeaders,
        };
      } catch (error: unknown) {
        lastError = error;
        if (!this.isRetryable(error) || attempt === this.retryAttempts) break;
      }
    }

    const exception = DownstreamServiceErrorException.fromAxiosError(lastError);
    const responseAt = Date.now();

    this.logger.error(
      `HTTP ${axiosConfig.method} ${axiosConfig.url} failed`,
      undefined,
      {
        requestData,
        responseData: {
          statusCode: exception.statusCode,
          message: exception.message,
          responseAt,
        },
        duration: responseAt - requestAt,
        stack: lastError instanceof Error ? lastError.stack : undefined,
        success: false,
      },
    );

    throw exception;
  }

  private isRetryable(error: unknown): boolean {
    if (!axios.isAxiosError(error)) return false;
    const axiosError = error as AxiosError;
    if (!axiosError.response) return true; // network errors, timeouts
    return axiosError.response.status >= 500;
  }
}
```

### `src/base/config/http/http-client.module.ts`

```typescript
import { Module, Global } from "@nestjs/common";
import { AxiosHttpClient } from "./axios.http-client";

export const HTTP_CLIENT = Symbol("HttpClient");

@Global()
@Module({
  providers: [
    {
      provide: HTTP_CLIENT,
      useClass: AxiosHttpClient,
    },
  ],
  exports: [HTTP_CLIENT],
})
export class HttpClientModule {}
```

### `src/base/constants/index.ts` (add these)

```typescript
export const PRIVATE_BODY_PROPERTIES = [
  "password",
  "token",
  "secret",
  "authorization",
  "creditCard",
];
```

### `src/base/lib/utils/security.utils.ts`

```typescript
export function obfuscatePrivateBodyProperties(
  body: unknown,
  privateProperties: string[],
): unknown {
  if (!body || typeof body !== "object") return body;

  const obfuscated = { ...(body as Record<string, unknown>) };
  for (const key of Object.keys(obfuscated)) {
    if (privateProperties.includes(key.toLowerCase())) {
      obfuscated[key] = "***";
    }
  }
  return obfuscated;
}
```

---

## Naming & DI Tokens {#naming}

### DI Token Convention

Flat `Symbol` tokens, one file per module at the module root (`<module>.di-tokens.ts`). A token is
needed **only for a class injected through an interface** — i.e. a **port** bound to its adapter
(repositories, gateways, event buses). A use case implements no interface, so it needs **no
token**.

```typescript
// user.di-tokens.ts
export const USER_REPOSITORY = Symbol("UserRepository");
```

### Module Registration

Bind ports (interface → adapter) by token; register use cases as **plain providers** (no token):

```typescript
// user.module.ts
@Module({
  providers: [
    { provide: USER_REPOSITORY, useClass: MongoUserRepository }, // port → adapter (token)
    UserCreator, // use case → plain provider (concrete class, no token)
    UserFinder,
  ],
})
export class UserModule {}
```

### Port (Interface)

Port functions receive domain classes as parameters (see 3 cases in SKILL.md):

Ports return the value directly and **throw** a `DomainException` on the error path — there is
no `Result` wrapper in the backend:

```typescript
// domain/ports/user.repository.ts
import { CreateUserCommand } from "../props/create-user.command";
import { FindByEmailProps } from "../props/find-by-email.props";

export interface UserRepository {
  // Case 1: same params as use case → reuse Command/Query
  save(command: CreateUserCommand): Promise<User>;
  // Case 3: different params → create <FunctionName>Props
  findByEmail(props: FindByEmailProps): Promise<User>;
}

// domain/ports/event-bus.port.ts
import { UserCreatedEvent } from "../events/user-created.event";

export interface EventBus {
  // Case 2: publishing an event
  publish(event: UserCreatedEvent): Promise<void>;
}
```

#### Port file isolation (only the interface)

A port file declares **only** the port interface(s) and nothing else. Every
supporting type it references — argument shapes **and** return/result wrappers —
is extracted to its own file under `domain/props/`. Never declare a helper `type`
or `interface` inline in the port file.

- **Argument shapes** — when a method's argument is **not identical** to an
  existing use-case input, create a `<action>-<entity>.props.ts` file in
  `domain/props/` holding an interface/class with **only the fields that method
  needs** (not the whole command). That type is the method's argument type
  (this is "Case 3 / `<FunctionName>Props`" above).
- **Return/result wrappers** — a wrapper the port returns (e.g. a paginated
  `MatchingSpots = { items: Spot[]; total: number }`) is likewise its own file in
  `domain/props/` (e.g. `matching-spots.props.ts`).

```typescript
// ❌ Wrong — result and argument shapes declared inline in the port file
// domain/ports/spot.repository.ts
export interface MatchingSpots { items: Spot[]; total: number }      // ⟵ move out
export interface CreateSpotParams { /* … */ }                        // ⟵ move out
export interface SpotRepository {
  matching(criteria: Criteria): Promise<MatchingSpots>;
  create(params: CreateSpotParams): Promise<Spot>;
}
```

```typescript
// ✅ Right — the port file holds only the interface
// domain/props/matching-spots.props.ts
import { type Spot } from "../entities/spot.entity";
export interface MatchingSpots {
  items: Spot[];
  total: number;
}

// domain/props/create-spot.props.ts   (only the fields the port needs)
import { type Location } from "@/modules/shared/domain/value-objects/location";
export interface CreateSpotProps {
  name: string;
  location: Location;
  // …only what create() persists — not the whole CreateSpotCommand
}

// domain/ports/spot.repository.ts   (only the interface)
import { type Criteria } from "@/base/lib/domain/criteria/criteria";
import { type Spot } from "../entities/spot.entity";
import { type MatchingSpots } from "../props/matching-spots.props";
import { type CreateSpotProps } from "../props/create-spot.props";

export interface SpotRepository {
  matching(criteria: Criteria): Promise<MatchingSpots>;
  findById(id: string): Promise<Spot | null>;
  create(props: CreateSpotProps): Promise<Spot>;
}
```

### Adapter (Implementation)

**Rule — wrap every technical operation in `try/catch`.** Each call an adapter makes to its
underlying technology (DB query, HTTP request, message publish, S3 call, …) is wrapped in a
`try/catch`. In the `catch`, the adapter **logs** the error (with the stack and the operation
name) and **throws a `DomainException` associated with that adapter** — an *infrastructure*
exception such as `DatabaseErrorException` (DB), `DownstreamServiceErrorException` (HTTP),
`MessagingErrorException` (RabbitMQ). The original error never leaks past the adapter; the
global `DomainExceptionFilter` maps the thrown exception to its `httpStatus` (typically `500`).

Distinguish the two kinds of throw inside an adapter:
- **Infrastructure failure** (the operation itself threw) → caught, logged, rethrown as the
  adapter's infra `DomainException` (`DatabaseErrorException`, …).
- **Business invariant** derived from a *successful* result (not found, duplicate) → thrown
  directly from the result, **not** from a `catch` (`UserNotFoundException`, `UserAlreadyExistsException`).

The infra exception is a plain `DomainException` with `httpStatus = 500`:

```typescript
// base/lib/domain/exceptions/database-error.exception.ts
import { HttpStatus } from "@nestjs/common";
import { DomainException } from "@/base/lib/domain/domain-exception.base";

export class DatabaseErrorException extends DomainException {
  override readonly httpStatus = HttpStatus.INTERNAL_SERVER_ERROR; // 500

  constructor(message = "A database error occurred") {
    super(message);
  }
}
```

```typescript
// infrastructure/driven/persistence/mongo-user.repository.ts
import { Inject, Injectable } from "@nestjs/common";
import { InjectModel } from "@nestjs/mongoose";
import { type Model } from "mongoose";
import { type LoggerPort } from "@/base/lib/domain/logger.port";
import { LOGGER_ADAPTER } from "@/base/config/logger/logger.di-tokens";
import { DatabaseErrorException } from "@/base/lib/domain/exceptions/database-error.exception";
import { CreateUserCommand } from "../../domain/props/create-user.command";
import { FindByEmailProps } from "../../domain/props/find-by-email.props";

@Injectable()
export class MongoUserRepository implements UserRepository {
  constructor(
    @InjectModel(UserDocument.name) private readonly model: Model<UserDocument>,
    @Inject(LOGGER_ADAPTER) private readonly logger: LoggerPort,
  ) {
    this.logger.setContext(MongoUserRepository.name);
  }

  async save(command: CreateUserCommand): Promise<User> {
    let existing: UserLean | null;
    try {
      existing = await this.model.findOne({ email: command.email }).lean<UserLean>();
    } catch (error: unknown) {
      this.logger.error(
        "Failed to query user by email",
        error instanceof Error ? error.stack : undefined,
        { operation: "save" },
      );
      throw new DatabaseErrorException();
    }

    // Business invariant — derived from the successful query result, not a failure:
    if (existing) throw new UserAlreadyExistsException(command.email);

    try {
      const doc = await this.model.create(command);
      return UserMapper.toDomain(doc);
    } catch (error: unknown) {
      this.logger.error(
        "Failed to persist user",
        error instanceof Error ? error.stack : undefined,
        { operation: "save" },
      );
      throw new DatabaseErrorException();
    }
  }

  async findByEmail(props: FindByEmailProps): Promise<User> {
    let doc: UserLean | null;
    try {
      doc = await this.model.findOne({ email: props.email }).lean<UserLean>();
    } catch (error: unknown) {
      this.logger.error(
        "Failed to query user by email",
        error instanceof Error ? error.stack : undefined,
        { operation: "findByEmail" },
      );
      throw new DatabaseErrorException();
    }

    if (!doc) throw new UserNotFoundException(props.email); // business invariant
    return UserMapper.toDomain(doc);
  }
}
```

### Persistence — Schema, Lean Reads & Mapper

The adapter persists through a Mongoose schema and rebuilds domain entities with a mapper. The
schema `extends Document`, enables `timestamps`, and owns its indexes; reads use `.lean()` with an
explicit lean type, and the mapper's `toDomain` rebuilds the entity from that plain object.

```typescript
// infrastructure/driven/persistence/user.schema.ts
import { Prop, Schema, SchemaFactory } from "@nestjs/mongoose";
import { Document } from "mongoose";

@Schema({ timestamps: true, collection: "users" })
export class UserDocument extends Document {
  @Prop({ required: true })
  name: string;

  @Prop({ required: true })
  email: string;

  // `timestamps: true` populates these; declare them so the mapper reads them typed.
  createdAt: Date;
  updatedAt: Date;
}

export const UserSchema = SchemaFactory.createForClass(UserDocument);

// Indexes are an intrinsic property of the schema — declare them here.
UserSchema.index({ email: 1 }, { unique: true });
```

```typescript
// infrastructure/driven/persistence/user.mapper.ts
import { type FlattenMaps, type Types } from "mongoose";
import { User } from "@/modules/user/domain/entities/user.entity";
import { type UserDocument } from "./user.schema";

// Shape returned by `.lean()` — a plain object, not a hydrated document.
export type UserLean = FlattenMaps<UserDocument> & { _id: Types.ObjectId };

export class UserMapper {
  static toDomain(raw: UserLean): User {
    return new User(
      String(raw._id),
      raw.name,
      raw.email,
      raw.createdAt,
      raw.updatedAt,
    );
  }
}
```

Rules:

- The schema class **extends `Document`**, lives in `infrastructure/driven/persistence/`, and
  declares `createdAt` / `updatedAt` (populated by `timestamps: true`) so the mapper reads them typed.
- Indexes are intrinsic to the schema — declare them with `Schema.index(...)` next to it.
- Reads use `.lean<XxxLean[]>()` where `XxxLean = FlattenMaps<XxxDocument> & { _id: Types.ObjectId }`;
  the mapper method is **`toDomain`** and converts the id with `String(raw._id)`.
- Mappers never leak Mongoose types upward — they return domain entities only.

### Criteria-based Repositories

When a repository needs dynamic filtering + ordering + pagination, it exposes a single
`matching(criteria: Criteria)` method instead of many `findByX`. The `Criteria` value objects live
in `src/base/lib/domain/criteria/` and the Mongo converter (`CriteriaToMongoConverter`, turning a
`Criteria` into `{ filter, sort, skip, limit }`) in `src/base/lib/infrastructure/criteria/` — both
are technical machinery, never inside a module or `modules/shared/`.

```typescript
// domain/ports/user.repository.ts
export interface MatchingUsers {
  items: User[];
  total: number;
}

export interface UserRepository {
  matching(criteria: Criteria): Promise<MatchingUsers>;
}
```

```typescript
// infrastructure/driven/persistence/mongo-user.repository.ts
async matching(criteria: Criteria): Promise<MatchingUsers> {
  const { filter, sort, skip, limit } = this.converter.convert(
    criteria,
    FILTERABLE_FIELDS, // per-repository whitelist: field → 'string' | 'number' | 'boolean' | 'date'
  );

  let docs: UserLean[];
  let total: number;
  try {
    [docs, total] = await Promise.all([
      this.model.find(filter).sort(sort).skip(skip).limit(limit).lean<UserLean[]>().exec(),
      this.model.countDocuments(filter).exec(),
    ]);
  } catch (error: unknown) {
    this.logger.error(
      "Failed to query users from database",
      error instanceof Error ? error.stack : undefined,
      { operation: "matching" },
    );
    throw new DatabaseErrorException();
  }

  return { items: docs.map((doc) => UserMapper.toDomain(doc)), total };
}
```

`FILTERABLE_FIELDS` doubles as the whitelist (a field absent from it is rejected) and the type map
the converter uses to coerce raw string values. For the full Criteria pattern (value objects,
operators, converter implementation), see the **`criteria-pattern`** skill.

### Command / Query (use case input)

```typescript
// domain/props/create-user.command.ts
import { Command } from "@/base/lib/domain/command.base";

export class CreateUserCommand extends Command {
  constructor(
    public readonly name: string,
    public readonly email: string,
  ) {
    super();
  }
}

// domain/props/find-user.query.ts
import { Query } from "@/base/lib/domain/query.base";

export class FindUserQuery extends Query {
  constructor(public readonly id: string) {
    super();
  }
}
```

### Use Case (Output + Mapper)

A use case returns a dedicated **Output** (or `Paginated<Output>`), never a domain entity. The
Output and its Mapper live in the use-case folder; the Mapper turns the port's domain entity
into the Output.

```typescript
// application/use-cases/user-creator/user-creator.output.ts
import { Output } from "@/base/lib/domain/output.base";

export class UserCreatorOutput extends Output {
  readonly id: string;
  readonly name: string;
  readonly email: string;
  readonly createdAt: string; // ISO 8601
  constructor(props: UserCreatorOutputProps) {
    super();
    // assign each field from props
  }
}
```

```typescript
// application/use-cases/user-creator/mapper/user-creator.mapper.ts
import { type User } from "../../../domain/user.entity";
import { UserCreatorOutput } from "../user-creator.output";

export class UserCreatorMapper {
  static toOutput(user: User): UserCreatorOutput {
    return new UserCreatorOutput({
      id: user.id,
      name: user.name,
      email: user.email,
      createdAt: user.createdAt.toISOString(),
    });
  }
}
```

```typescript
// application/use-cases/user-creator/user-creator.use-case.ts
import { UseCase } from "@/base/lib/application/use-case.base";
import { CreateUserCommand } from "../../domain/props/create-user.command";
import { UserCreatorMapper } from "./mapper/user-creator.mapper";
import { type UserCreatorOutput } from "./user-creator.output";

@Injectable()
export class UserCreator extends UseCase<CreateUserCommand, UserCreatorOutput> {
  constructor(
    @Inject(USER_REPOSITORY) private readonly userRepository: UserRepository,
  ) {
    super(); // mandatory: `UseCase` is a base class, so a derived constructor must call it
  }

  async execute(command: CreateUserCommand): Promise<UserCreatorOutput> {
    // The port throws (e.g. UserAlreadyExistsException) on the error path; let it
    // propagate to the global DomainExceptionFilter.
    const user = await this.userRepository.save(command);
    return UserCreatorMapper.toOutput(user);
  }
}
```

For a collection use case, return `Paginated<Output>`:

```typescript
// application/use-cases/user-finder/user-finder.use-case.ts
export class UserFinder extends UseCase<FindUsersQuery, Paginated<UserFinderOutput>> {
  async execute(query: FindUsersQuery): Promise<Paginated<UserFinderOutput>> {
    const { items, total } = await this.userRepository.matching(/* criteria */);
    return new Paginated(
      items.map(UserFinderMapper.toOutput),
      total,
      query.page,
      query.limit,
    );
  }
}
```

---

## Domain Exception Filter {#domain-exception-filter}

Domain and application code **throw** a `DomainException` on the error path; use cases and
controllers never catch it. The thrown `DomainException` (which extends `Error`) is caught by
this global filter and mapped to the proper `HttpException` **by its `code`** — never by class
name (`constructor.name` breaks under minification and renames silently).

Lives in `src/base/config/nestjs/domain-exception.filter.ts`:

```typescript
import {
  type ArgumentsHost,
  Catch,
  ConflictException,
  type ExceptionFilter,
  HttpException,
  InternalServerErrorException,
  NotFoundException,
} from "@nestjs/common";
import type { Response } from "express";
import { DomainException } from "@/base/lib/domain/domain-exception.base";

@Catch(DomainException)
export class DomainExceptionFilter implements ExceptionFilter {
  catch(exception: DomainException, host: ArgumentsHost) {
    const response = host.switchToHttp().getResponse<Response>();
    const httpException = this.mapToHttpException(exception);
    response.status(httpException.getStatus()).json(httpException.getResponse());
  }

  private mapToHttpException(error: DomainException): HttpException {
    const map: Record<string, HttpException> = {
      USER_NOT_FOUND: new NotFoundException(error.message),
      USER_ALREADY_EXISTS: new ConflictException(error.message),
    };
    return map[error.code] ?? new InternalServerErrorException();
  }
}
```

Register globally in `main.ts`:

```typescript
app.useGlobalFilters(new DomainExceptionFilter());
```

This filter is the **only** domain-error-to-HTTP mechanism. Controllers return wrapper
instances (`SingleResponse` / `PaginatedResponse`) directly and let any `DomainException`
propagate to the filter — there is no `Result` type and no Result-unwrapping interceptor.

---

## Module Example — Full User Module {#module-example}

### Source (`src/modules/user/`)

```
modules/user/
├── domain/
│   ├── user.entity.ts
│   ├── user.di-tokens.ts
│   ├── value-objects/
│   │   └── user-email.value-object.ts
│   ├── exceptions/
│   │   ├── user-not-found.exception.ts
│   │   └── user-already-exists.exception.ts
│   ├── props/
│   │   ├── create-user.command.ts
│   │   ├── find-user.query.ts
│   │   └── find-by-email.props.ts
│   ├── events/
│   │   └── user-created.event.ts
│   └── ports/
│       ├── user.repository.ts
│       └── event-bus.port.ts
├── application/
│   ├── use-cases/
│   │   ├── user-creator/
│   │   │   ├── user-creator.use-case.ts
│   │   │   ├── user-creator.output.ts
│   │   │   └── mapper/
│   │   │       └── user-creator.mapper.ts
│   │   └── user-finder/
│   │       ├── user-finder.use-case.ts
│   │       ├── user-finder.output.ts
│   │       └── mapper/
│   │           └── user-finder.mapper.ts
│   └── user.module.ts
└── infrastructure/
    ├── driven/
    │   └── persistence/
    │       ├── mongo-user.repository.ts
    │       └── schemas/
    │           └── user.schema.ts
    └── driving/
        └── http/
            ├── create-user/
            │   ├── dto/
            │   │   └── create-user.request.dto.ts
            │   ├── mapper/
            │   │   └── create-user.mapper.ts
            │   └── create-user.http.controller.ts
            └── find-user/
                └── find-user.http.controller.ts
```

### Tests (`tests/modules/user/`)

```
tests/modules/user/
├── application/
│   └── use-cases/
│       ├── user-creator/
│       │   ├── user-creator.use-case.spec.ts
│       │   └── mapper/
│       │       └── user-creator.mapper.spec.ts
│       └── user-finder/
│           ├── user-finder.use-case.spec.ts
│           └── mapper/
│               └── user-finder.mapper.spec.ts
└── infrastructure/
    ├── driven/
    │   └── persistence/
    │       └── mongo-user.repository.spec.ts
    └── driving/
        └── http/
            ├── create-user/
            │   └── create-user.http.controller.spec.ts
            └── find-user/
                └── find-user.http.controller.spec.ts
```

---

## Driving Vertical Slice {#driving-slice}

`infrastructure/driving/` is organized first by **protocol** (`http/`, `rpc/`, `messaging/`, `cli/`) and then by **action folder** — one folder per endpoint. Each action folder is a self-contained vertical slice with its DTOs, mapper, and controller colocated.

### Action folder layout

```
<action-name>/
├── dto/
│   └── <action-name>.request.dto.ts    ← optional (request/input side only)
├── mapper/
│   └── <action-name>.mapper.ts         ← optional (request DTO → Command/Query)
└── <action-name>.<protocol>.controller.ts
```

Rules:

- **One folder per action.** Folder name in kebab-case (`create-user/`, `find-all-users/`, `product-created/`).
- `dto/` and `mapper/` are **singular folder names**. Omit either if the action does not need it.
- There is **no response DTO**. The controller wraps the use case's **Output** (or `Paginated<Output>`) directly — response shaping happens in the application use-case Mapper, not here.
- Controller file: `<action>.<protocol>.controller.ts`. Class: `<Action><Protocol>Controller` (e.g. `CreateUserHttpController`).
- The controller class is the **only** thing that imports protocol decorators (`@Controller`, `@MessagePattern`, etc.).

### Request DTO

Plain class with `class-validator` decorators for input validation. Lives in the action's `dto/` folder.

```typescript
// infrastructure/driving/http/create-user/dto/create-user.request.dto.ts
import { IsEmail, IsString, MinLength } from "class-validator";

export class CreateUserRequestDto {
  @IsString()
  @MinLength(2)
  name: string;

  @IsEmail()
  email: string;
}
```

### Response contract — the use-case Output (no response DTO)

There is **no** infrastructure response DTO. The use case returns an **Output** (see
*Use Case (Output + Mapper)* above) that already describes exactly the shape returned to the
client — only the fields the API exposes, dates as ISO strings. The controller wraps that
Output directly, so the response contract stays in the application layer, out of
`infrastructure/`.

### Mapper

Static class mapping the **request DTO → Command/Query**. One mapper per action, request side
only — response shaping is the application use-case Mapper's job (domain entity → Output).

```typescript
// infrastructure/driving/http/create-user/mapper/create-user.mapper.ts
import { CreateUserCommand } from "@/modules/user/domain/props/create-user.command";
import type { CreateUserRequestDto } from "../dto/create-user.request.dto";

export class CreateUserMapper {
  static createUserRequestDtoToCreateUserCommand(
    dto: CreateUserRequestDto,
  ): CreateUserCommand {
    return new CreateUserCommand(dto.name, dto.email);
  }
}
```

Rules:

- Method names follow `<Source>To<Target>` using the actual class names — never generic `toDto` / `toDomain`.
- Mapper methods are **static** — never instantiate the mapper.

### Response Wrappers

Every controller response is wrapped in one of two envelopes from `src/base/lib/infrastructure/`:

```typescript
// src/base/lib/infrastructure/single-response.ts
export class SingleResponse<T> {
  constructor(public readonly data: T) {}
}

// src/base/lib/infrastructure/paginated-response.ts
export interface PaginationMeta {
  page: number;
  limit: number;
  total: number;
  totalPages: number;
}

export class PaginatedResponse<T> {
  constructor(
    public readonly data: T[],
    public readonly pagination: PaginationMeta,
  ) {}
}
```

- Single-record endpoints return `SingleResponse<<UseCase>Output>`.
- List/paginated endpoints return `PaginatedResponse<<UseCase>Output>`, built from the use case's `Paginated<Output>`.
- The controller **derives `totalPages` at the boundary** (`total === 0 ? 0 : Math.ceil(total / limit)`); the application `Paginated<T>` carries only `items / total / page / limit`.
- **Never** return a raw entity, raw array, or naked DTO from a controller.

### HTTP Controller example

```typescript
// infrastructure/driving/http/create-user/create-user.http.controller.ts
import { Body, Controller, Post } from "@nestjs/common";
import { SingleResponse } from "@/base/lib/infrastructure/single-response";
import type { UserCreatorOutput } from "@/modules/user/application/use-cases/user-creator/user-creator.output";
import { UserCreator } from "@/modules/user/application/use-cases/user-creator/user-creator.use-case";
import { CreateUserRequestDto } from "./dto/create-user.request.dto";
import { CreateUserMapper } from "./mapper/create-user.mapper";

@Controller("users")
export class CreateUserHttpController {
  // Use case injected by class — it implements no interface, so no token.
  constructor(private readonly userCreator: UserCreator) {}

  @Post()
  async handle(
    @Body() dto: CreateUserRequestDto,
  ): Promise<SingleResponse<UserCreatorOutput>> {
    const command =
      CreateUserMapper.createUserRequestDtoToCreateUserCommand(dto);
    // The use case returns the Output directly and throws a DomainException on
    // the error path; the global DomainExceptionFilter maps it to HTTP.
    const output = await this.userCreator.execute(command);

    // Return the wrapper instance directly so the metadata interceptor can
    // enrich it; the use case already returns the Output, ready to wrap.
    return new SingleResponse(output);
  }
}
```

### Paginated example

The use case returns `Paginated<UserFinderOutput>` (see *Use Case (Output + Mapper)*). The
controller has no response mapper — it wraps the Output list and derives `totalPages`:

```typescript
// infrastructure/driving/http/find-all-users/find-all-users.http.controller.ts
@Controller("users")
export class FindAllUsersHttpController {
  constructor(private readonly userFinder: UserFinder) {}

  @Get()
  async handle(
    @Query() query: FindAllUsersRequestDto,
  ): Promise<PaginatedResponse<UserFinderOutput>> {
    const { items, total, page, limit } = await this.userFinder.execute(
      FindAllUsersMapper.toQuery(query),
    );
    const totalPages = total === 0 ? 0 : Math.ceil(total / limit);
    return new PaginatedResponse(items, { page, limit, total, totalPages });
  }
}
```

### RPC / Messaging controller example

```typescript
// infrastructure/driving/rpc/product-created/product-created.rpc.controller.ts
import { Controller } from "@nestjs/common";
import { EventPattern, Payload } from "@nestjs/microservices";
import { ProductProjector } from "@/modules/product/application/use-cases/product-projector/product-projector.use-case";
import { ProductCreatedRequestDto } from "./dto/product-created.request.dto";
import { ProductCreatedMapper } from "./mapper/product-created.mapper";

@Controller()
export class ProductCreatedRpcController {
  constructor(private readonly productProjector: ProductProjector) {}

  @EventPattern("product.created")
  async handle(@Payload() dto: ProductCreatedRequestDto): Promise<void> {
    const command =
      ProductCreatedMapper.productCreatedRequestDtoToProjectProductCommand(dto);
    await this.productProjector.execute(command);
  }
}
```

---

## Base Client Pattern {#base-client-pattern}

Every external service integration (HTTP APIs, message brokers, cloud storage, email providers, payment gateways, etc.) follows the same pattern in `src/base/`. This ensures consistent configuration, logging, error handling, and DI across all integrations.

### Pattern structure

```
src/base/config/<concern>/
├── <library>.<concern>.client.ts       ← wrapper class
├── <concern>.client.module.ts          ← @Global() module with DI token
└── exception/
    └── <concern>-error.exception.ts    ← optional: typed error for this integration
```

### 4 required pieces

| #   | Piece                    | Location                                                                      | Purpose                                                                                                                                                                          |
| --- | ------------------------ | ----------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | **Client class**         | `base/config/<concern>/<library>.<concern>.client.ts`                         | Wraps the third-party SDK. Injects `EnvVarsService` for config and `LoggerPort` for structured logging. Encapsulates retry logic, error mapping, and correlation-id propagation. |
| 2   | **Global module**        | `base/config/<concern>/<concern>.client.module.ts`                            | `@Global() @Module` that provides the client via a `Symbol` DI token and exports it. Registered once in `AppModule`.                                                             |
| 3   | **Env vars**             | `base/config/env/env-vars.schema.ts` + `env-vars.service.ts` + `.env.example` | All three files updated simultaneously with the new variables.                                                                                                                   |
| 4   | **Exception** (optional) | `base/config/<concern>/exception/<concern>-error.exception.ts`                | Typed error class extending `DomainException` (stable `code`, message via `super`) with a static factory method `fromSdkError(error)` for consistent error mapping.                                                   |

### Existing implementations

| Concern     | Client class              | DI Token           | Env vars                                                             |
| ----------- | ------------------------- | ------------------ | -------------------------------------------------------------------- |
| `http`      | `AxiosHttpClient`         | `HTTP_CLIENT`      | `EXTERNAL_SERVICE_API_URL`, `HTTP_TIMEOUT_MS`, `HTTP_RETRY_ATTEMPTS` |
| `messaging` | `RabbitMQMessagingClient` | `MESSAGING_CLIENT` | `RABBITMQ_URL`                                                       |

### Client class template

```typescript
// src/base/config/<concern>/<library>.<concern>.client.ts
import { Inject, Injectable } from '@nestjs/common'
import { EnvVarsService } from '../env/env-vars.service'
import type { LoggerPort } from '@/base/lib/domain/logger.port'
import { LOGGER_ADAPTER } from '../logger/logger.di-tokens'

@Injectable()
export class <Library><Concern>Client {
  private readonly client: <SdkType>

  constructor(
    private readonly envVarsService: EnvVarsService,
    @Inject(LOGGER_ADAPTER) private readonly logger: LoggerPort,
  ) {
    this.logger.setContext(<Library><Concern>Client.name)
    this.client = this.initializeClient()
  }

  // Public methods: domain-agnostic operations
  // Example: upload(), download(), send(), publish()
  // Each method:
  //   1. Logs the operation start with correlation-id
  //   2. Delegates to the SDK
  //   3. Logs success/failure with duration
  //   4. Returns a typed result or throws a typed exception

  private initializeClient(): <SdkType> {
    // Initialize the SDK with env vars
    // Configure timeouts, regions, credentials, etc.
  }
}
```

### Global module template

```typescript
// src/base/config/<concern>/<concern>.client.module.ts
import { Module, Global } from '@nestjs/common'
import { <Library><Concern>Client } from './<library>.<concern>.client'

export const <CONCERN>_CLIENT = Symbol('<Concern>Client')

@Global()
@Module({
  providers: [
    {
      provide: <CONCERN>_CLIENT,
      useClass: <Library><Concern>Client,
    },
  ],
  exports: [<CONCERN>_CLIENT],
})
export class <Concern>ClientModule {}
```

### How modules consume it (two-layer separation)

The base client is **infrastructure cross-cutting** — it knows the SDK but not the domain. Each module adds its own hexagonal layer on top:

```
base/config/storage/          ← generic S3 client (upload bytes, delete key)
    ↑ injected via STORAGE_CLIENT
modules/spot/domain/ports/
    image-storage.port.ts     ← domain port (uploadSpotImage, deleteSpotImage)
modules/spot/infrastructure/driven/storage/
    s3-image-storage.adapter.ts  ← adapter: implements port, uses STORAGE_CLIENT
```

1. **Port** (domain) — defines operations in domain language: `uploadSpotImage(spotId, file)`.
2. **Adapter** (driven infrastructure) — injects the base client via `@Inject(<CONCERN>_CLIENT)`, translates domain calls to SDK calls (builds S3 keys, sets content types, etc.).
3. **DI token** (domain) — module-level token: `IMAGE_STORAGE = Symbol('ImageStorage')`.
4. **Wiring** (module) — `{ provide: IMAGE_STORAGE, useClass: S3ImageStorage }`.

This means:

- The domain never knows about S3, Axios, or RabbitMQ.
- The base client never knows about spots, skaters, or business rules.
- Swapping the SDK (e.g. S3 → GCS) only changes the base client class. Ports and adapters stay the same.
- Swapping the provider within a module (e.g. S3 → local filesystem for tests) only changes the adapter binding in the module.

### Rules

- **One client per external concern.** Don't mix S3 and SES in the same client.
- **Client methods are domain-agnostic.** `upload(bucket, key, body)` not `uploadSpotImage(spotId, file)`. Domain translation happens in the module's adapter.
- **Always log with correlation-id.** Use `getRequestContext()?.correlationId` from `base/config/context/`.
- **Always add env vars to all three files simultaneously.** Schema, service, and `.env.example`.
- **Always `@Global()`.** Base clients are used across modules — avoid per-module imports.
- **Never import a base client directly in a use case.** The use case depends on the port, the adapter depends on the client. Two layers of indirection.
- **Error mapping in the client, not the adapter.** The client catches SDK errors and throws a typed `DomainException` subclass. The adapter receives clean errors.
