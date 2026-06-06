> **Nota:** este archivo es la traducción en español de `SKILL.md` para lectura humana.
> No es una skill registrada — Claude Code solo carga `SKILL.md`. No lo uses como
> contexto operativo.

# Skill: Arquitectura Hexagonal

Esta skill impone una arquitectura hexagonal (Ports & Adapters) consistente en todos los
proyectos TypeScript. Antes de generar cualquier archivo o carpeta, lee el archivo de
referencia apropiado para el tipo de proyecto.

## Detección del tipo de proyecto

Identifica el tipo de proyecto desde el contexto (package.json, estructura existente, o
descripción del usuario):

| Tipo | Indicadores | Archivo de referencia |
|---|---|---|
| Monolito NestJS | NestJS + múltiples dominios/módulos | `references/backend-nestjs.md` |
| Microservicio NestJS | NestJS + un solo dominio | `references/backend-nestjs.md` |
| SPA Vue.js | Vue 3 + Vue Router | `references/frontend-spa-vue.md` |
| SPA React | React + React Router (sin Next.js) | `references/frontend-spa-react.md` |
| React Native | Expo + React Navigation | `references/frontend-mobile-react-native.md` |
| Next.js | Next.js + App Router | `references/frontend-nextjs.md` |

**Siempre lee el archivo de referencia antes de generar cualquier estructura.**

> Si no puedes determinar el tipo de proyecto desde `package.json`, estructura de
> carpetas, o descripción del usuario, **pregunta al usuario antes de generar cualquier
> cosa**. Nunca adivines el tipo de proyecto.

---

## Reglas universales (aplican a TODOS los tipos de proyecto)

### Capas
Cada proyecto tiene exactamente 3 capas (backend) o 4 capas (frontend):
1. **domain/** — entidades, value objects, ports (interfaces), errores de dominio, eventos de dominio
2. **application/** — use cases, DTOs, errores de aplicación
3. **infrastructure/** — adapters, clients, models, schemas, controllers
4. **presentation/** — pantallas, view models, componentes, rutas, store *(solo frontend)*

**La infraestructura del backend se divide en `driving/` y `driven/`:**
- `infrastructure/driving/` — puntos de entrada que **invocan** la aplicación (controllers HTTP, consumers RPC, consumers de mensajes, handlers CLI, cron jobs). Organizado por **protocolo** y luego **slice vertical por acción**.
- `infrastructure/driven/` — adapters salientes que la aplicación **invoca** (repositorios, publishers de mensajes, clientes HTTP a otros servicios, emisores de email, etc.). Organizado por preocupación técnica (`persistence/`, `messaging/`, `http/`, `email/`).

Esta división hace explícita la dirección hexagonal: los adapters driving empujan hacia el hexágono, los adapters driven son empujados por el hexágono.

### Convenciones de nomenclatura — Clases e interfaces
| Concepto | Convención | Ejemplo |
|---|---|---|
| Port (interface) | `<Entity><Role>` | `UserRepository` |
| Adapter (impl) | `<Infra><Entity><Role>` | `MongoUserRepository`, `HttpUserRepository` |
| Use case | `<Entity><Action>` | `UserCreator`, `UserFinder`, `UserUpdater` |
| Carpeta del use case | `<entity>-<action>/` | `user-creator/` |
| Output del use case (backend) | `<UseCase>Output` | `SpotsFinderOutput`, `SpotCreatorOutput` |
| Mapper del use case (backend) | `<UseCase>Mapper` | `SpotsFinderMapper`, `SpotCreatorMapper` |
| Contenedor Paginated (dominio) | `Paginated<T>` | `Paginated<SpotsFinderOutput>` |
| Excepción de dominio | `<Entity><Reason>Exception` | `UserNotFoundException`, `UserAlreadyExistsException` |
| Excepción HTTP (backend) | `DownstreamServiceErrorException` | `DownstreamServiceErrorException` |
| Excepción HTTP (frontend) | `HttpServiceException` | `HttpServiceException` |
| Props (frontend) | `<Action><Entity>Props` | `CreateUserProps`, `FindUserProps` |
| Command (backend) | `<Action><Entity>Command` | `CreateUserCommand`, `UpdateUserCommand` |
| Query (backend) | `<Action><Entity>Query` | `FindUserQuery`, `ListUsersQuery` |
| Event (backend) | `<Entity><Action>Event` | `UserCreatedEvent`, `OrderCancelledEvent` |
| Props de función port | `<FunctionName>Props` | `FindByEmailProps`, `SaveUserProps` |
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
| Wrapper Single Response | `SingleResponse<T>` | `SingleResponse<SpotCreatorOutput>` |
| Wrapper Paginated Response | `PaginatedResponse<T>` | `PaginatedResponse<SpotsFinderOutput>` |
| Layout (SPA) | `<Scope>Layout.tsx\|vue` | `PublicLayout.tsx`, `PrivateLayout.tsx` |
| Layout ViewModel (SPA) | `use<Scope>LayoutViewModel.ts` | `usePrivateLayoutViewModel.ts` |

### Convenciones de nomenclatura — Sufijos de archivos
| Concepto | Sufijo | Ejemplo |
|---|---|---|
| Entity | `.entity.ts` | `user.entity.ts` |
| Value Object | `.value-object.ts` | `user-email.value-object.ts` |
| Port (interface) | `.repository.ts` / `.port.ts` | `user.repository.ts` |
| Adapter | `.repository.ts` | `mongo-user.repository.ts`, `http-user.repository.ts` |
| Use Case | `.use-case.ts` | `user-creator.use-case.ts` |
| Output del use case (backend) | `.output.ts` | `spots-finder.output.ts` |
| Mapper del use case (backend) | `.mapper.ts` (en la carpeta `mapper/` del use case) | `spots-finder.mapper.ts` |
| Props (frontend) | `.props.ts` | `create-user.props.ts` |
| Command (backend) | `.command.ts` | `create-user.command.ts` |
| Query (backend) | `.query.ts` | `find-user.query.ts` |
| Event (backend) | `.event.ts` | `user-created.event.ts` |
| Props de función port | `.props.ts` | `find-by-email.props.ts` |
| Request DTO (backend) | `.request.dto.ts` | `create-user.request.dto.ts` |
| Driving Mapper (backend) | `.mapper.ts` | `find-all-users.mapper.ts` |
| HTTP Controller (backend) | `<action>.http.controller.ts` | `create-user.http.controller.ts` |
| RPC Controller (backend) | `<action>.rpc.controller.ts` | `product-created.rpc.controller.ts` |
| Messaging Controller (backend) | `<action>.messaging.controller.ts` | `order-placed.messaging.controller.ts` |
| Excepción de dominio | `.exception.ts` | `user-not-found.exception.ts` |
| Schema (DB) | `.schema.ts` | `user.schema.ts` |
| Tokens DI | `.di-tokens.ts` | `user.di-tokens.ts` |
| Test | `.spec.ts` | `user-creator.use-case.spec.ts` |
| Screen Mapper (frontend) | `<screen-kebab>.mapper.ts` | `login-screen.mapper.ts`, `user-profile-screen.mapper.ts` |
| Presentation Model (frontend) | `.model.ts` (nombre libre + sufijo `Model`) | `user-summary.model.ts`, `user-permissions.model.ts` |
| Form Model (frontend) | `.form-model.ts` | `create-user.form-model.ts` |
| Form Mapper (frontend) | `.form-mapper.ts` | `create-user.form-mapper.ts` |

Todos los nombres de archivos usan **kebab-case**. Nunca uses camelCase o PascalCase para nombres de archivos.

### Patrón Result (manual)
Cada proyecto incluye una clase `Result<T>` manual en `src/base/lib/domain/result.ts`. El tipo de error es **siempre `DomainException`** — no es un parámetro genérico:

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

Reglas:
- Usar `Result.ok(value)` y `Result.err(error)` en las capas de **domain** y **application** de TODOS los proyectos
- El lado del error es siempre `DomainException` — cualquier excepción concreta (ej. `UserNotFoundException`) funciona porque `extends DomainException`
- En **backend**: convertir resultados de error en `HttpException` en la frontera de infraestructura vía un interceptor global
- En **frontend**: usar `try/catch` en las capas de infraestructura y presentación
- Nunca lances errores de dominio — siempre retorna `Result.err(new DomainException())`

### Clases base del dominio
Cada proyecto incluye estas clases base abstractas en `src/base/lib/domain/`. **Genera cada archivo exactamente como se muestra — no renombres clases:**

```typescript
// src/base/lib/domain/command.base.ts
export abstract class Command {}

// src/base/lib/domain/query.base.ts
export abstract class Query {}

// src/base/lib/domain/props.base.ts
export abstract class Props {}

// src/base/lib/domain/domain-exception.base.ts
export abstract class DomainException {}

// src/base/lib/domain/output.base.ts (solo backend — base de los Outputs de use case)
export abstract class Output {}

// src/base/lib/domain/event.base.ts (solo backend)
export interface EventMetadata {
  eventId: string
  occurredAt: Date
}

export abstract class Event implements EventMetadata {
  readonly eventId: string = crypto.randomUUID()
  readonly occurredAt: Date = new Date()
}
```

Todos los Commands, Queries, Props, Events, errores de dominio y Outputs de use case deben extender estas clases base:
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

// domain/events/user-created.event.ts (solo backend)
import { Event } from '@/base/lib/domain/event.base'
export class UserCreatedEvent extends Event {
  constructor(public readonly userId: string, public readonly email: string) { super() }
}

// application/use-cases/spots-finder/spots-finder.output.ts (solo backend)
import { Output } from '@/base/lib/domain/output.base'
export class SpotsFinderOutput extends Output { ... }
```

### Contenedor Paginated (dominio — universal)
Cada proyecto incluye una clase genérica `Paginated<T>` en `src/base/lib/domain/paginated.ts`.
Es la forma de retorno canónica para cualquier use case que devuelva una colección.
**Genera este archivo exactamente como se muestra:**

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

Reglas:
- Un use case que devuelve una colección retorna `Paginated<Output>` (backend) o
  `Paginated<Entity>` (frontend), nunca un array crudo.
- **Sin campo `totalPages`.** `totalPages` es un valor de presentación derivado que se calcula
  en el borde de infraestructura (el controller, al envolver en `PaginatedResponse`), no en la
  capa de aplicación.
- `Paginated<T>` se instancia directamente (`new Paginated(items, total, page, limit)`), por eso
  su archivo es `paginated.ts` (no `*.base.ts`, reservado para bases abstractas que se extienden).

### Reglas de entidades de dominio
Las entidades de dominio (ej. `User`, `Product`, `Spot`) representan objetos de negocio persistidos y siguen reglas estrictas de construcción.

**1. Atributos requeridos — toda entidad de dominio DEBE tener:**
- `id` — no nullable, asignado por la capa de persistencia (o generado aguas arriba cuando esté justificado)
- `createdAt: Date` — no nullable, seteado cuando la entidad se persiste por primera vez
- `updatedAt: Date` — no nullable, actualizado en cada mutación

Estos tres campos **nunca son opcionales ni nullables**. Un objeto al que le falte alguno no es una entidad de dominio válida.

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

**No** hay una clase base `Entity` compartida. Cada entidad declara sus propios campos explícitamente.

**2. Reglas de instanciación — `new <Entity>(...)` solo está permitido en dos lugares:**

   **a) Dentro de un mapper infraestructura → dominio** (el caso canónico). Los adapters reciben datos raw de una fuente de datos (fila DB, respuesta HTTP, modelo ORM) y un mapper reconstruye la entidad:
   ```typescript
   // infrastructure/mappers/user.mapper.ts
   export class UserMapper {
     static toDomain(raw: UserRecord): User {
       return new User(raw.id, raw.email, raw.name, raw.created_at, raw.updated_at)
     }
   }
   ```

   **b) Dentro de un use case, solo cuando TODO lo siguiente aplica:**
   - Cada campo requerido (`id`, `createdAt`, `updatedAt`, más todos los campos de negocio) ya está disponible en memoria — sin nullables, sin placeholders, sin `new Date()` como filler para `createdAt` de algo que aún no se ha persistido.
   - Hay un **propósito justificable** para construirla en el use case en vez de delegar a un adapter (ej. ensamblar una entidad desde piezas ya obtenidas, proyección en memoria; los fixtures de test dentro del use case **no** son razón válida).
   - Si hay duda, delega al adapter y deja que el mapper la construya.

**3. Prohibido:**
- ❌ Instanciar una entidad en presentation, orquestación de application, o cualquier otro lugar.
- ❌ Instanciar una entidad para representar datos que **aún no existen** en la fuente (ej. "construir un `User` para pasarlo a `userRepository.create(user)`"). Para flujos de creación/modificación, pasa un `Command`, `Query` o `Props` al adapter — el adapter persiste y retorna la entidad completa.
- ❌ Hacer `id`, `createdAt`, o `updatedAt` opcionales, nullables, o con default en el constructor.

**4. Flujo de datos para crear/actualizar:**
```
Presentation (FormModel)
  → FormMapper.toProps()
  → UseCase.execute(props)
  → Adapter.create(props)         ← el adapter persiste, recibe id/timestamps de la DB
  → Mapper.toDomain(record)       ← entidad instanciada AQUÍ
  → retornada hacia arriba como User
```
El use case **nunca** llama a `new User(...)` para pasárselo al adapter en creación.

### Convención de parámetros de Port
Las funciones de port (métodos de repositorio, métodos de servicio) reciben sus parámetros como clases de dominio. Hay 3 casos:

**Caso 1 — Mismos params que el use case:** reusa el Input directamente (Command/Query/Props)
```typescript
// El port recibe el mismo Command que recibió el use case
export interface UserRepository {
  save(command: CreateUserCommand): Promise<Result<User>>
}
```

**Caso 2 — Publicar un evento (solo backend):** crea una clase Event en `domain/events/`
```typescript
// El port publica un evento a una cola de mensajes
export interface EventBus {
  publish(event: UserCreatedEvent): Promise<Result<void>>
}
```

**Caso 3 — Params distintos:** crea una clase `<FunctionName>Props` en `domain/props/`
```typescript
// domain/props/find-by-email.props.ts
import { Props } from '@/base/lib/domain/props.base'
export class FindByEmailProps extends Props {
  constructor(public readonly email: string) { super() }
}

// El port recibe una clase Props específica
export interface UserRepository {
  findByEmail(props: FindByEmailProps): Promise<Result<User>>
}
```

### Clase base UseCase
Cada proyecto incluye una clase abstracta `UseCase` en `src/base/lib/application/use-case.base.ts`. **Genera este archivo exactamente como se muestra — no renombres la clase, los genéricos, ni el método:**

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

Todos los use cases **extienden** esta clase (nunca `implements`). **Los use cases de backend
devuelven un `Output` (o `Paginated<Output>`) — nunca una entidad de dominio ni un DTO de
infraestructura.** Los use cases de frontend devuelven datos de dominio (la capa de presentation
los adapta vía Screen Mapper / Presentation Model).
```typescript
// Backend — resultado simple
export class SpotCreator extends UseCase<CreateSpotCommand, SpotCreatorOutput> {
  execute(command: CreateSpotCommand): Promise<Result<SpotCreatorOutput>> {
    // ...
  }
}

// Backend — colección paginada
export class SpotsFinder extends UseCase<FindSpotsQuery, Paginated<SpotsFinderOutput>> {
  execute(query: FindSpotsQuery): Promise<Result<Paginated<SpotsFinderOutput>>> {
    // ...
  }
}

// Frontend — devuelve datos de dominio
export class UserCreator extends UseCase<CreateUserProps, User> {
  execute(props: CreateUserProps): Promise<Result<User>> {
    // ...
  }
}
```

### Convención Output & Mapper de Use Case (solo backend)

Un use case de backend nunca devuelve una entidad de dominio ni un DTO de infraestructura.
Devuelve una clase **Output** dedicada, y un **Mapper** colocado junto a él convierte el
resultado del port (normalmente una entidad de dominio) en ese Output. Así la capa de
aplicación queda autocontenida: posee su propio contrato de retorno y nunca toca
`infrastructure/`.

**Estructura de carpeta** — toda carpeta de use case contiene tres cosas:
```
application/use-cases/spots-finder/
├── spots-finder.use-case.ts      ← SpotsFinder (orquestación)
├── spots-finder.output.ts        ← SpotsFinderOutput extends Output (contrato de retorno)
└── mapper/
    └── spots-finder.mapper.ts     ← SpotsFinderMapper (entidad de dominio → Output)
```

**1. Output** (`<use-case>.output.ts`, clase `<UseCase>Output extends Output`):
- Una selección curada de los campos que la API realmente expone — descarta arrays pesados,
  flags internos y cualquier dato sensible.
- Las fechas se serializan a strings ISO aquí (el Output es lo que el controller entrega al
  response wrapper, así que lleva la forma final).
- Puede reutilizar tipos de value-object del dominio (ej. `Location`) ya que `application → domain`
  está permitido.

**2. Mapper** (`mapper/<use-case>.mapper.ts`, clase estática `<UseCase>Mapper`):
- Un método estático, `toOutput(entity): <UseCase>Output`, que mapea la entidad de dominio del
  port al Output. El nombre se mantiene corto (`toOutput`) — la clase ya nombra el use case.

```typescript
// application/use-cases/spots-finder/mapper/spots-finder.mapper.ts
import { type Spot } from '@/modules/spot/domain/entities/spot.entity'
import { SpotsFinderOutput } from '../spots-finder.output'

export class SpotsFinderMapper {
  static toOutput(spot: Spot): SpotsFinderOutput {
    return new SpotsFinderOutput({ id: spot.id, /* …solo los campos que la API expone */ })
  }
}
```

**3. El use case** solo orquesta — llama al port, mapea entidades con el Mapper, y envuelve la
colección en `Paginated`. Sin ensamblado campo a campo, sin `totalPages`, sin imports de infra:

```typescript
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

**4. El controller** envuelve el Output directamente (sin response DTO aparte) y deriva
`totalPages` en el borde:
```typescript
async find(@Query() dto: FindSpotsQueryDto): Promise<PaginatedResponse<SpotsFinderOutput>> {
  const result = await this.spotsFinder.execute(FindSpotsHttpMapper.toQuery(dto))
  if (result.isErr()) throw result.getError()

  const { items, total, page, limit } = result.getValue()
  const totalPages = total === 0 ? 0 : Math.ceil(total / limit)
  return new PaginatedResponse(items, { page, limit, total, totalPages })
}
```

> Un use case de resultado simple devuelve el Output directamente
> (`UseCase<CreateSpotCommand, SpotCreatorOutput>`) y su controller lo envuelve en
> `SingleResponse<SpotCreatorOutput>`.

### Convención Form Model & Form Mapper (solo frontend)

Cuando una pantalla tiene un formulario, la capa de presentation define:

1. **Form Model** — una `interface` en `presentation/models/` que representa los campos del formulario tal como los ve la UI
2. **Form Mapper** — una clase estática en `presentation/mappers/` que convierte el Form Model en la clase `Props` de dominio antes de llamar al use case

Esto mantiene la capa de presentation desacoplada del dominio: el formulario trabaja con su propio modelo y el mapper hace la traducción.

```typescript
// presentation/models/create-user.form-model.ts
export interface CreateUserFormModel {
  name: string
  email: string
  passwordConfirmation: string  // campo solo-UI, no parte de Props de dominio
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

El ViewModel (SPAs) o Server Action (Next.js) usa el mapper:
```typescript
// dentro del ViewModel
const props = CreateUserFormMapper.toProps(formData)
const result = await userCreator.execute(props)
```

Reglas:
- Los Form Models son **interfaces** (no clases) — representan el estado puro del form
- Los Form Mappers son **clases estáticas** — no requieren instanciación
- El mapper recibe el Form Model y retorna una instancia de `Props` de dominio
- Los campos que existen solo en la UI (ej. `passwordConfirmation`) los elimina el mapper
- Nombrado: `<action>-<entity>.form-model.ts` y `<action>-<entity>.form-mapper.ts`

### Convención Screen Mapper & Presentation Model (solo frontend)

Cada pantalla que consume entidades de dominio define un **Screen Mapper** que convierte esas entidades en uno o más **Presentation Models** a la medida de lo que la UI realmente renderiza. Este es el único lugar del frontend que traduce dominio → UI.

**Nombrado de archivo y clase:**
- El archivo del mapper se nombra según la **pantalla** en kebab-case: `login-screen.mapper.ts`, `user-profile-screen.mapper.ts`. La clase es `<Screen>Mapper`.
- Un mapper por pantalla, incluso si la pantalla consume múltiples entidades. Un solo mapper puede mapear varias entidades distintas — por eso es a nivel de pantalla, no de entidad.
- Los archivos de Presentation Model usan el sufijo `.model.ts` y la clase/interface termina en `Model`. El nombre base es **libre** (que el contexto decida): una pantalla suele necesitar más de un modelo (`UserSummaryModel`, `UserPermissionsModel`, `RecentActivityModel`), y forzar el nombre de la pantalla en cada modelo sería engañoso.

**Nombrado de métodos dentro del mapper:**
Cada método de mapeo se nombra según su fuente y destino con el patrón `<source>To<target>`, donde fuente/destino usan los nombres reales de clases (entity, model, dto):
- `<Entity>DomainTo<Name>Model` — entidad de dominio → presentation model
- `<Name>ModelTo<Entity>Domain` — presentation model → entidad de dominio (cuando se necesita)

Esto hace explícita la dirección y evita nombres ambiguos `toModel` / `fromModel` cuando un mapper maneja múltiples tipos.

```typescript
// presentation/models/user-summary.model.ts
export interface UserSummaryModel {
  fullName: string
  initials: string
  joinedLabel: string  // pre-formateado para la UI, ej. "Joined 2 months ago"
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

Reglas:
- **Mapea solo los campos que la UI realmente usa**. Nunca hagas spread de una entidad ni copies campos que la pantalla no renderiza.
- Un mapper por pantalla. El archivo del mapper se nombra según la pantalla.
- Los nombres de métodos siguen `<Source>To<Target>` usando nombres reales de clases — nunca genéricos `toModel` / `fromModel`.
- Los Presentation Models son interfaces (no clases) salvo que se necesite comportamiento.
- Los models viven en `presentation/models/` con nombres base libres + sufijo `Model`.

### Convención de Slice Vertical Driving (solo backend)

`infrastructure/driving/` está organizado primero por **protocolo** (`http/`, `rpc/`, `messaging/`, `cli/`) y dentro de cada protocolo por **carpeta de acción** — una carpeta por endpoint / consumer / handler. Cada carpeta de acción contiene todo lo que ese endpoint necesita, colocado junto:

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
    │   │   │   └── create-user.mapper.ts   ← request DTO → Command
    │   │   └── create-user.http.controller.ts
    │   └── find-all-users/
    │       ├── dto/
    │       │   └── find-all-users.request.dto.ts   ← params de paginación/filtro
    │       ├── mapper/
    │       │   └── find-all-users.mapper.ts         ← request DTO → Query
    │       └── find-all-users.http.controller.ts
    └── rpc/
        └── product-created/
            ├── dto/
            │   └── product-created.request.dto.ts
            ├── mapper/
            │   └── product-created.mapper.ts
            └── product-created.rpc.controller.ts
```

Reglas:
- **Una carpeta por acción.** El nombre de la carpeta en kebab-case coincide con la acción (ej. `create-user/`, `find-all-users/`, `product-created/`).
- **`dto/` en singular** — contiene solo el lado **request**: `<action>.request.dto.ts` (la forma de entrada HTTP/RPC). **No hay response DTO**: el **Output** del use case es el contrato de respuesta (ver *Convención Output & Mapper de Use Case*). Omite `dto/` cuando la acción no recibe input.
- **`mapper/` en singular** — contiene `<action>.mapper.ts`, que mapea solo **request DTO → Command/Query**. Omite la carpeta si la acción no recibe input. La clase es `<Action>Mapper`. (Distinto del `<UseCase>Mapper` de la capa de aplicación, que mapea entidad de dominio → Output.)
- **Archivo del controller** es `<action>.<protocol>.controller.ts`. La clase es `<Action><Protocol>Controller` (ej. `CreateUserHttpController`, `ProductCreatedRpcController`).
- El controller es lo **único** que conoce el protocolo. El mapper, DTOs y use case son agnósticos al protocolo en forma (los nombres de DTO viven junto al protocolo porque son con forma de protocolo).

**Nombrado de métodos del mapper** — el driving mapper maneja solo el lado **request** (el
shaping de la respuesta vive en el `<UseCase>Mapper` de aplicación → Output, ver *Convención
Output & Mapper de Use Case*):
- `<Action>RequestDtoTo<Action><Command|Query>` — DTO de request → input de application

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

### Convención de Response Wrapper (solo backend)

Cada respuesta de controller se envuelve en uno de dos wrappers genéricos definidos en `src/base/lib/infrastructure/`:

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

Reglas:
- Los endpoints de un solo registro retornan `SingleResponse<<UseCase>Output>`.
- Los endpoints de lista/paginados retornan `PaginatedResponse<<UseCase>Output>`, construido a partir del `Paginated<Output>` del use case.
- El controller **deriva `totalPages` en el borde** (`total === 0 ? 0 : Math.ceil(total / limit)`); el `Paginated<T>` de aplicación solo lleva `items / total / page / limit`.
- **Nunca** retornes una entidad raw, un array raw, o un DTO desnudo desde un controller.
- El **Output** (construido por el `<UseCase>Mapper`) ya incluye solo los campos que la API expone — los campos internos, derivados y sensibles se eliminan ahí, no en el controller.

### Convenciones de testing
| Capa | Tipo de test | Estrategia de mock |
|---|---|---|
| `domain/` | Tests unitarios — puros, sin mocks | Ninguna |
| `application/` | Tests unitarios | Mockear ports (interfaces) |
| `infrastructure/` | Tests unitarios | Mockear clientes HTTP, ORM, etc. |
| `presentation/` | Tests de componentes | Mockear view models / composables |

Frameworks de testing por tipo de proyecto:
| Proyecto | Framework | Testing de componentes |
|---|---|---|
| NestJS | Jest (incluido en NestJS) | — |
| Vue.js | Vitest | `@vue/test-utils` |
| SPA React | Vitest | `@testing-library/react` |
| React Native | Jest (incluido en Expo) | `@testing-library/react-native` |
| Next.js | Vitest | `@testing-library/react` |

Los archivos de test viven en una carpeta `tests/` al nivel raíz, espejando la estructura de `src/`:
```
src/modules/user/application/use-cases/user-creator/user-creator.use-case.ts
tests/modules/user/application/use-cases/user-creator/user-creator.use-case.spec.ts
```

### Calidad de código
Todos los proyectos usan `husky` + `lint-staged` para pre-commit hooks que corren linting y formateo automáticamente.

### Regla de dependencias (impuesta vía `eslint-plugin-boundaries`)
- Los módulos **nunca** importan de otros módulos
- Si algo se necesita en más de un módulo → muévelo a `shared/` (monolito/frontend)
- `domain/` tiene cero dependencias externas
- `application/` depende solo de `domain/`
- `infrastructure/` depende de `application/` y `domain/`
- `presentation/` depende de `application/` y `domain/`

Estas reglas se **imponen en tiempo de lint** usando `eslint-plugin-boundaries`. Cada proyecto lo instala como dev dependency y configura ESLint para definir elementos de frontera (uno por capa) con `default: 'disallow'` y reglas `allow` explícitas coincidiendo con la lista de arriba. Las violaciones fallan el lint y por tanto fallan el pre-commit hook de `lint-staged`.

Ve cada archivo de referencia para la configuración ESLint específica del proyecto — backend y frontend difieren en cantidad de capas y estructura de carpetas.

### Estructura de src/base
Cada proyecto tiene una carpeta `src/base/` con preocupaciones técnicas transversales:

```
src/base/
├── config/          ← infraestructura técnica (env, http, logger, di, etc.)
├── constants/
│   └── index.ts     ← constantes de todo el proyecto (export const VARIABLE_NAME = value)
└── lib/             ← clases base abstractas extendidas por módulos
    ├── domain/         ← result.ts, command.base.ts, query.base.ts, props.base.ts, domain-exception.base.ts, event.base.ts, value-object.base.ts
    ├── application/    ← use-case.base.ts
    ├── infrastructure/ ← single-response.ts, paginated-response.ts (solo backend)
    └── utils/          ← funciones utilitarias puras agrupadas por concern
```

### Convención de variables de entorno

Cada proyecto sigue estas reglas para variables de entorno:

**Quién crea cada archivo:**
- `.env.example` — **creado por el agente** al scaffoldear el proyecto. Contiene todas las variables requeridas con valores vacíos o placeholders. **Committeado al repo.**
- `.env` / `.env.local` — **creado por el desarrollador** con valores reales. **Nunca committeado** (va a `.gitignore`).
- `.env.development` / `.env.production` — opcionales, creados por el desarrollador por ambiente.

**Prefijo de variable por tipo de proyecto** (impuesto por la plataforma, no opcional):
- **Backend NestJS**: sin prefijo — se accede vía `process.env.VAR_NAME`, leído a través de `EnvVarsService`
- **Vite (Vue, React)**: prefijo `VITE_` — solo las variables `VITE_*` se exponen al navegador vía `import.meta.env`
- **Next.js**: prefijo `NEXT_PUBLIC_` para variables accesibles desde cliente; sin prefijo para variables solo-servidor
- **Expo (React Native)**: prefijo `EXPO_PUBLIC_` — solo las `EXPO_PUBLIC_*` se empaquetan en la app

**Patrón de acceso**: nunca leas `process.env` o `import.meta.env` directamente fuera de `src/base/config/env/`. Todo el código que necesite una env var importa desde la config centralizada. Esto significa:
- Un solo lugar para validar todas las env vars al arranque
- Un solo lugar para arreglar una variable faltante
- Acceso type-safe en todos lados

Ve cada archivo de referencia para la implementación completa (schema, service/config, `.env.example`).

### Convención de constantes
Todas las constantes del proyecto viven en `src/base/constants/index.ts`. Usa `UPPER_SNAKE_CASE` para los nombres:
```typescript
// src/base/constants/index.ts
export const API_BASE_URL = 'https://api.example.com'
export const MAX_RETRY_ATTEMPTS = 3
export const DEFAULT_PAGE_SIZE = 20
```

### Convención de utils
Las funciones utilitarias viven en `src/base/lib/utils/`, agrupadas por concern — un archivo por tema, múltiples funciones por archivo:
```
src/base/lib/utils/
├── index.ts              ← re-exporta todos los utils para imports limpios
├── date.utils.ts         ← formatDate, parseDate, diffInDays, ...
├── string.utils.ts       ← capitalize, slugify, truncate, ...
├── number.utils.ts       ← formatCurrency, roundTo, clamp, ...
├── array.utils.ts        ← chunk, unique, groupBy, ...
├── object.utils.ts       ← deepClone, pick, omit, ...
└── validation.utils.ts   ← isEmail, isUrl, isEmpty, ...
```

Reglas:
- Todas las funciones deben ser **puras** — sin dependencias externas, sin efectos secundarios
- Convención de nombre de archivo: `<concern>.utils.ts`
- Si un archivo crece más allá de ~30 funciones, separa por sub-concern (ej. `date-format.utils.ts`, `date-parse.utils.ts`)
- `index.ts` re-exporta todo para que los consumidores importen desde un solo lugar: `import { formatDate, slugify } from '@/base/lib/utils'`

El backend también incluye:
```
src/base/
├── health/          ← controller y módulo de healthcheck
└── config/
    ├── context/     ← correlation ID, request context
    ├── messaging/   ← cliente del message broker (RabbitMQ por defecto)
    └── nestjs/      ← filtro global de excepciones, interceptor de logger
```

---

## Referencia rápida: estructura de archivos de Use Case

Cada use case vive en su propia carpeta. La clase de input (Props, Command, o Query) vive en `domain/props/`:

**Frontend:**
```
src/.../domain/props/
└── create-user.props.ts          ← clase CreateUserProps

src/.../application/use-cases/user-creator/
└── user-creator.use-case.ts      ← recibe CreateUserProps
```

**Backend:** la carpeta del use case contiene el use case, su Output y su Mapper:
```
src/.../domain/props/
├── create-user.command.ts        ← clase CreateUserCommand (operaciones de escritura)
└── find-user.query.ts            ← clase FindUserQuery (operaciones de lectura)

src/.../application/use-cases/user-creator/
├── user-creator.use-case.ts      ← recibe CreateUserCommand, devuelve UserCreatorOutput
├── user-creator.output.ts        ← UserCreatorOutput extends Output
└── mapper/
    └── user-creator.mapper.ts     ← UserCreatorMapper (entidad de dominio → Output)

src/.../application/use-cases/user-finder/
├── user-finder.use-case.ts       ← recibe FindUserQuery, devuelve Paginated<UserFinderOutput>
├── user-finder.output.ts         ← UserFinderOutput extends Output
└── mapper/
    └── user-finder.mapper.ts      ← UserFinderMapper (entidad de dominio → Output)
```

**Tests:**
```
tests/.../application/use-cases/user-creator/
├── user-creator.use-case.spec.ts
└── mapper/
    └── user-creator.mapper.spec.ts
```

---

## Flujo — Crear un proyecto nuevo

Cuando el usuario pida crear un proyecto desde cero, sigue estos pasos:

1. **Confirma el tipo de proyecto** con el usuario si no es obvio
2. **Verifica la documentación oficial** — Antes de scaffoldear, obtén la página oficial de "getting started" del framework para confirmar la manera recomendada de crear un proyecto nuevo y asegurarte de usar la última versión. Usa estas URLs:

   | Framework | Docs oficiales |
   |---|---|
   | NestJS | https://docs.nestjs.com |
   | Vue.js | https://vuejs.org/guide/quick-start |
   | React (Vite) | https://vite.dev/guide |
   | React Native | https://docs.expo.dev/get-started/create-a-project |
   | Next.js | https://nextjs.org/docs/getting-started/installation |

   Este paso es crítico — los comandos CLI y flags recomendados cambian entre versiones. Nunca dependas solamente de los comandos en los archivos de referencia; siempre cruza con las docs oficiales.

3. **Lee el archivo de referencia** para ese tipo de proyecto (ver tabla de arriba)
4. **Scaffoldea el proyecto** usando el comando CLI verificado desde las docs oficiales
5. **Instala dependencias** listadas en el archivo de referencia para ese tipo de proyecto
6. **Crea `src/base/`** con las carpetas config y lib como se describe en la referencia
7. **Crea el primer módulo** (si el usuario especificó un dominio) siguiendo la estructura de carpetas
8. **Registra DI** — cablea ports y adapters en el contenedor de DI (módulo NestJS o Awilix)
9. **Registra rutas** — agrega rutas del módulo a la config de router/navegación (excepto Next.js, que usa file system)

---

## Flujo — Agregar un módulo o feature

Cuando el usuario pida agregar un módulo, feature, use case, o archivos nuevos:

1. **Detecta el tipo de proyecto** desde `package.json`, estructura de carpetas existente, o descripción del usuario
2. **Lee el archivo de referencia** para ese tipo de proyecto
3. **Crea la estructura de carpetas del módulo** dentro de `src/modules/<domain>/` (monolito/frontend) o `src/app/` (microservicio)
4. **Genera archivos** siguiendo convenciones de nomenclatura (tanto nombres de clase como sufijos de archivo)
5. **Respeta la regla de dependencias** — domain no importa de capas externas
6. **Registra en DI** — agrega providers al módulo NestJS o regístralo en el contenedor Awilix
7. **Agrega rutas** si el módulo tiene endpoints HTTP (backend) o pantallas (frontend)
8. **Genera tests** en `tests/` espejando la ruta de `src/`: `src/.../foo.ts` → `tests/.../foo.spec.ts`

No te saltes los pasos 6-8 a menos que el usuario lo diga explícitamente.

---

## Archivos de referencia

Para estructuras de carpetas completas, patrones de DI, convenciones específicas del framework, comandos de setup del proyecto y ejemplos de código, lee la referencia apropiada:

- **NestJS (monolito o microservicio)** → leer `references/backend-nestjs.md`
- **SPA Vue.js** → leer `references/frontend-spa-vue.md`
- **SPA React** → leer `references/frontend-spa-react.md`
- **React Native (Expo)** → leer `references/frontend-mobile-react-native.md`
- **Next.js** → leer `references/frontend-nextjs.md`
