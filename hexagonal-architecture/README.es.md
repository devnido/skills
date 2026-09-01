> **Nota:** este archivo es la traducción en español de `SKILL.md` para lectura humana.
> No es una skill registrada — Claude Code solo carga `SKILL.md`. No lo uses como
> contexto operativo.

# Skill: Arquitectura Hexagonal

Esta skill impone una arquitectura hexagonal (Ports & Adapters) consistente en todos los
proyectos TypeScript. Este archivo es el router y el resumen de reglas. **Antes de generar
cualquier archivo o carpeta, lee DOS archivos de referencia:**

1. `references/universal-conventions.md` — las reglas universales completas y las plantillas
   de código canónicas (Result, clases base, Paginated, wrappers, reglas de
   entidades/VOs/ports, convenciones de env/utils). **Obligatorio para toda tarea** — el
   resumen de abajo lo sintetiza, pero la referencia es la fuente de la verdad.
2. La referencia del tipo de proyecto (tabla de abajo) — estructura, DI y setup específicos
   del framework.

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

**Siempre lee los archivos de referencia antes de generar cualquier estructura.**

> Si no puedes determinar el tipo de proyecto desde `package.json`, estructura de
> carpetas, o descripción del usuario, **pregunta al usuario antes de generar cualquier
> cosa**. Nunca adivines el tipo de proyecto.

### Monorepos (varios proyectos en un repo)

- El **`package.json` más cercano** subiendo desde el archivo que estás tocando define el
  proyecto y su tipo. Un `package.json` raíz con `workspaces` (o `pnpm-workspace.yaml`,
  `turbo.json`, `nx.json`) **no es un proyecto** — es orquestación; nunca detectes el tipo
  desde ahí.
- Toda ruta de esta skill (`src/`, `tests/`, `.env.example`, config de ESLint) es relativa
  a **la raíz de ese proyecto**, nunca a la raíz del repo.
- **Cada proyecto es su propio hexágono.** Nunca importes código de un proyecto hermano —
  los servicios se hablan solo por sus contratos públicos (API REST, broker de mensajes).
  Es la regla "los módulos nunca importan otros módulos", un nivel más arriba.
- La duplicación de `src/base/` entre proyectos hermanos es **deliberada** (es el precio
  de la autonomía de cada servicio). No la "deduplices" hacia una carpeta compartida del
  repo ni un paquete de workspace — extraer un paquete compartido es una decisión
  explícita del usuario, jamás iniciativa del agente.
- Una tarea que cruza dos proyectos son N sub-tareas: relee el reference correspondiente
  al entrar a cada proyecto. Si no está claro a qué proyecto pertenece un cambio,
  **pregunta**.

---

## Capas

Cada proyecto tiene exactamente 3 capas (backend) o 4 capas (frontend):
1. **domain/** — entidades, value objects, ports (interfaces), errores de dominio, eventos de dominio
2. **application/** — use cases, DTOs, errores de aplicación
3. **infrastructure/** — adapters, clientes, modelos, schemas, controllers
4. **presentation/** — pantallas, view models, componentes, rutas, store *(solo frontend)*

**La infraestructura de backend se divide en `driving/` y `driven/`:**
- `infrastructure/driving/` — puntos de entrada que **invocan** la aplicación (controllers HTTP, consumers RPC, consumers de mensajes, handlers CLI, cron jobs). Organizada por **protocolo** y luego **vertical slice por acción**.
- `infrastructure/driven/` — adapters de salida que la aplicación **invoca** (repositorios, publishers de mensajes, clientes HTTP hacia otros servicios, envío de emails, etc.). Organizada por concern técnico (`persistence/`, `messaging/`, `http/`, `email/`).

Esta división hace explícita la dirección hexagonal: los driving adapters empujan hacia adentro del hexágono, los driven adapters son empujados por el hexágono.

## Convenciones de nombres — Clases e interfaces

| Concepto | Convención | Ejemplo |
|---|---|---|
| Port (interface) | `<Entity><Role>` | `UserRepository` |
| Adapter (impl) | `<Infra><Entity><Role>` | `MongoUserRepository`, `HttpUserRepository` |
| Use case | `<Entity><Action>` | `UserCreator`, `UserFinder`, `UserUpdater` |
| Carpeta de use case | `<entity>-<action>/` | `user-creator/` |
| Output de use case (backend) | `<UseCase>Output` | `SpotsFinderOutput`, `SpotCreatorOutput` |
| Mapper de use case (backend) | `<UseCase>Mapper` | `SpotsFinderMapper`, `SpotCreatorMapper` |
| Contenedor paginado (dominio) | `Paginated<T>` | `Paginated<SpotsFinderOutput>` |
| Excepción de dominio | `<Entity><Reason>Exception` | `UserNotFoundException`, `UserAlreadyExistsException` |
| Excepción HTTP (backend) | `DownstreamServiceErrorException` | `DownstreamServiceErrorException` |
| Excepción HTTP (frontend) | `HttpServiceException` | `HttpServiceException` |
| Props (frontend) | `<Action><Entity>Props` | `CreateUserProps`, `FindUserProps` |
| Command (backend) | `<Action><Entity>Command` | `CreateUserCommand`, `UpdateUserCommand` |
| Query (backend) | `<Action><Entity>Query` | `FindUserQuery`, `ListUsersQuery` |
| Event (backend) | `<Entity><Action>Event` | `UserCreatedEvent`, `OrderCancelledEvent` |
| Props de función de port | `<FunctionName>Props` | `FindByEmailProps`, `SaveUserProps` |
| Value object | `<Entity><Field>ValueObject` | `UserEmailValueObject` |
| Pantalla (SPA) | `<Screen>Screen.tsx\|vue` | `UserProfileScreen.tsx` |
| ViewModel (SPA) | `use<Screen>ViewModel.ts` | `useUserProfileViewModel.ts` |
| Página (Next.js) | `<Screen>Page.tsx` | `UserProfilePage.tsx` |
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

## Convenciones de nombres — Sufijos de archivo

| Concepto | Sufijo | Ejemplo |
|---|---|---|
| Entidad | `.entity.ts` | `user.entity.ts` |
| Value Object | `.value-object.ts` | `user-email.value-object.ts` |
| Port (interface) | `.repository.ts` / `.port.ts` | `user.repository.ts` |
| Adapter | `.repository.ts` | `mongo-user.repository.ts`, `http-user.repository.ts` |
| Use Case | `.use-case.ts` | `user-creator.use-case.ts` |
| Output de use case (backend) | `.output.ts` | `spots-finder.output.ts` |
| Mapper de use case (backend) | `.mapper.ts` (en la carpeta `mapper/` del use case) | `spots-finder.mapper.ts` |
| Props (frontend) | `.props.ts` | `create-user.props.ts` |
| Command (backend) | `.command.ts` | `create-user.command.ts` |
| Query (backend) | `.query.ts` | `find-user.query.ts` |
| Event (backend) | `.event.ts` | `user-created.event.ts` |
| Props de función de port | `.props.ts` | `find-by-email.props.ts` |
| Request DTO (backend) | `.request.dto.ts` | `create-user.request.dto.ts` |
| Driving Mapper (backend) | `.mapper.ts` | `find-all-users.mapper.ts` |
| Controller HTTP (backend) | `<action>.http.controller.ts` | `create-user.http.controller.ts` |
| Controller RPC (backend) | `<action>.rpc.controller.ts` | `product-created.rpc.controller.ts` |
| Controller Messaging (backend) | `<action>.messaging.controller.ts` | `order-placed.messaging.controller.ts` |
| Excepción de dominio | `.exception.ts` | `user-not-found.exception.ts` |
| Schema (DB) | `.schema.ts` | `user.schema.ts` |
| Tokens DI | `.di-tokens.ts` | `user.di-tokens.ts` |
| Test | `.spec.ts` | `user-creator.use-case.spec.ts` |
| Screen Mapper (frontend) | `<screen-kebab>.mapper.ts` | `login-screen.mapper.ts`, `user-profile-screen.mapper.ts` |
| Presentation Model (frontend) | `.model.ts` (nombre libre + sufijo `Model`) | `user-summary.model.ts`, `user-permissions.model.ts` |
| Form Model (frontend) | `.form-model.ts` | `create-user.form-model.ts` |
| Form Mapper (frontend) | `.form-mapper.ts` | `create-user.form-mapper.ts` |

Todos los nombres de archivo usan **kebab-case**. Nunca uses camelCase ni PascalCase para nombres de archivo.

## Reglas universales — Resumen

Las reglas completas y las plantillas de código canónicas viven en
`references/universal-conventions.md`. Resumen de lo no negociable:

- **Manejo de errores** — **Frontend (patrón Result):** `domain/` y `application/` retornan `Result<T>` (`Result.ok` / `Result.err`, el lado del error siempre una subclase de `DomainException`); los adapters hacen `try/catch` del cliente HTTP y convierten todo fallo en `Result.err`; el ViewModel desenvuelve el `Result` y mapea el `code` del error a estado de UI — nunca lanza. **Backend (sin Result):** `domain/` y `application/` **lanzan** `DomainException` directamente y retornan valores planos; un `DomainExceptionFilter` global mapea el `code` de la excepción lanzada a una `HttpException` (el **único** mecanismo de mapeo de errores — sin `Result`, sin interceptor que desenvuelva). **Los adapters de backend envuelven cada operación técnica (consulta a BD, request HTTP, publicación en cola, S3) en un `try/catch`: ante un fallo loguean el error y lanzan una `DomainException` de infraestructura asociada a ese adapter (`DatabaseErrorException`, `DownstreamServiceErrorException`, …, `httpStatus 500`); las excepciones de negocio ("no encontrado"/"duplicado") se lanzan desde el resultado exitoso, no desde el `catch`.**
- **Clases base del dominio** (`src/base/lib/domain/`) — `Command`, `Query`, `Props`, `Output` (nominales vía un campo privado `_brand`) y `DomainException extends Error` (`code` estable abstracto, message vía `super`); `Event` con `eventId`/`occurredAt` (backend). Todo input, output, error y evento extiende su base. Genera estos archivos desde las plantillas de la referencia universal — nunca improvises su forma.
- **`Paginated<T>`** (`src/base/lib/domain/paginated.ts`) — retorno canónico para colecciones: `items / total / page / limit`. Sin `totalPages` — se deriva en el borde del controller.
- **Entidades de dominio** — `id`, `createdAt`, `updatedAt` son obligatorios y no-nullables; no hay clase base `Entity` compartida. `new <Entity>(...)` solo se permite en mappers infraestructura→dominio (caso canónico) o, excepcionalmente, en un use case cuando todos los campos ya existen en memoria. Nunca construyas una entidad para datos que aún no existen — pasa el Command/Props al adapter; él persiste y retorna la entidad.
- **Value objects** — VO validado de un solo valor = clase `<Entity><Field>ValueObject`; VO de datos estructural = interface plana sin sufijo. Los VOs de negocio compartidos por ≥2 módulos van en `modules/shared/domain/value-objects/`; los VOs propios de un agregado, en su módulo. Nunca en `src/base/` (solo maquinaria técnica).
- **Aislamiento del archivo del port** — un archivo de port (`domain/ports/<entity>.repository.ts`) declara **solo** la(s) interfaz(es) del port y nada más. Todo tipo auxiliar que referencie — formas de argumentos y wrappers de retorno/resultado — se extrae a su propio archivo en `domain/props/`, nunca declarado inline en el archivo del port. Ejemplo: `SpotRepository` contiene solo la interfaz; su resultado `MatchingSpots` y su argumento `CreateSpotProps` viven cada uno en `domain/props/`.
- **Parámetros de ports** — los ports reciben clases de dominio: reutiliza el Command/Query/Props del use case cuando los parámetros coinciden exactamente; un `Event` para publishers; en otro caso un `<FunctionName>Props` dedicado en `domain/props/` (sufijo `.props.ts`) que contenga **solo los campos que ese método del port necesita** (no el command completo). El wrapper de retorno/resultado de un port (p. ej. `MatchingSpots = { items; total }`) también es su propio archivo en `domain/props/`.
- **Base UseCase** — todo use case hace `extends UseCase<I, O>` (nunca `implements`). Los use cases de backend retornan un `Output` o `Paginated<Output>` — nunca una entidad de dominio ni un DTO de infra; los de frontend retornan datos de dominio. DI en backend: los use cases son providers planos inyectados por clase; los tokens DI se reservan para ports enlazados a adapters.
- **Output y Mapper (backend)** — cada carpeta de use case contiene `<uc>.use-case.ts`, `<uc>.output.ts` (campos curados de la API, fechas como strings ISO) y `mapper/<uc>.mapper.ts` con un `toOutput(entity)` estático.
- **Form Model y Form Mapper (frontend)** — los formularios usan una interface FormModel en `presentation/models/` más un FormMapper estático en `presentation/mappers/` que convierte a `Props` de dominio, eliminando los campos solo-UI.
- **Screen Mapper y Presentation Models (frontend)** — un mapper por pantalla (`<Screen>Mapper`), métodos nombrados `<source>To<target>` (nunca `toModel`/`fromModel`), los models son interfaces con solo los campos que la UI renderiza.
- **MVVM (solo SPAs y React Native — nunca Next.js)** — la Screen (View) es pasiva: consume solo lo que retorna su ViewModel y nunca importa `domain/`/`application/`, el contenedor DI, stores ni mappers. El ViewModel (`use<Screen>ViewModel`) recibe los params de ruta como argumentos, se auto-inicializa, construye clases Props, desenvuelve el `Result` del use case, y retorna solo Presentation Models + estado de UI + acciones. Next.js usa Server Components + Server Actions en su lugar — ahí el punto de unwrap es el Server Component (lecturas: `notFound()` o throw) y el Server Action (mutaciones: retornar un `{ error }` serializable).
- **Vertical slice driving (backend)** — `infrastructure/driving/<protocolo>/<acción>/` con `dto/` (solo el lado request — el Output del use case es el contrato de respuesta), `mapper/` (request DTO → Command/Query) y `<action>.<protocol>.controller.ts`. Solo el controller conoce el protocolo.
- **Response wrappers (backend)** — todo controller retorna `SingleResponse<Output>` o `PaginatedResponse<Output>` **directamente** (nunca un `Result`, nunca una entidad/array/DTO desnudos), derivando `totalPages` en el borde.
- **Testing** — domain: tests unitarios puros; application: unitarios con ports mockeados; infrastructure: unitarios con clientes mockeados; presentation: tests de componentes. Los tests viven en `tests/` espejando `src/`. **Genera los tests desde las plantillas canónicas** de la referencia universal (use case, mapper, adapter; la plantilla de ViewModel está en cada referencia SPA) — mockea solo en la costura del port / use case; los fixtures de entidades salen de builders en `tests/modules/<module>/builders/`.
- **Regla de dependencias** (impuesta con `eslint-plugin-boundaries`) — un módulo nunca importa los **internals** de otro módulo (entities, adapters, use cases); el código de negocio compartido va a `shared/`; `domain/` no depende de nada; `application/` → `domain/`; `infrastructure/` y `presentation/` → `application/` + `domain/`. **Excepción — ports cross-módulo (monolito):** un módulo PUEDE importar la **interfaz del port + su token DI** de otro módulo e inyectarla (el consumidor importa el módulo proveedor, que `exports` el token). Es la única importación inter-módulo permitida; sin ella todo contrato cross-módulo terminaría forzado en `shared/`. Un use case puede inyectar ese port (p. ej. `SpotCreator` inyectando el `SKATER_REPOSITORY` del módulo skater) — depende de la interfaz, nunca del adapter del otro módulo.
- **`src/base/`** — solo cross-cutting técnico: `config/`, `constants/index.ts` (UPPER_SNAKE_CASE), `lib/` (clases base + `utils/` puros agrupados por concern). El patrón Criteria vive en `base/lib/**/criteria/` (ver la skill `criteria-pattern`).
- **Variables de entorno** — el agente crea `.env.example` (commiteado); los prefijos por plataforma son obligatorios (`VITE_`, `NEXT_PUBLIC_`, `EXPO_PUBLIC_`, ninguno para NestJS); solo `src/base/config/env/` lee `process.env` / `import.meta.env`.
- **Auth y sesión (frontend)** — los tokens son maquinaria técnica en `src/base/config/auth/` (port `TokenStorage`; localStorage / SecureStore / cookies httpOnly según plataforma); el HTTP client base inyecta `Authorization` + headers estables de identidad (ej. `device-id`) y ejecuta un **refresh single-flight ante 401** a través de un cliente pelado; el estado de sesión vive en la store del módulo auth (aplican las reglas MVVM — los tokens nunca entran a la store); login/register son un módulo `modules/auth/` normal. El SSG de Next.js se autentica con un token guest/de servicio en build time desde env vars.
- **Calidad de código** — todo proyecto usa hooks pre-commit con `husky` + `lint-staged` que corren lint y formato.

---

## Workflow — Crear un proyecto nuevo

Cuando el usuario pida crear un proyecto desde cero, sigue estos pasos:

1. **Confirma el tipo de proyecto** con el usuario si no es obvio
2. **Verifica la documentación oficial** — Antes de hacer scaffold, consulta la página oficial de "getting started" del framework para confirmar la forma recomendada de crear un proyecto nuevo y asegurarte de usar la última versión. Usa estas URLs:

   | Framework | Docs oficiales |
   |---|---|
   | NestJS | https://docs.nestjs.com |
   | Vue.js | https://vuejs.org/guide/quick-start |
   | React (Vite) | https://vite.dev/guide |
   | React Native | https://docs.expo.dev/get-started/create-a-project |
   | Next.js | https://nextjs.org/docs/getting-started/installation |

   Este paso es crítico — los comandos de CLI y flags recomendados cambian entre versiones. Nunca confíes solo en los comandos de los archivos de referencia; siempre cruza con la documentación oficial.

3. **Lee `references/universal-conventions.md` y la referencia del tipo de proyecto** (ver tabla de arriba)
4. **Haz el scaffold del proyecto** usando el comando CLI verificado de la documentación oficial
5. **Instala las dependencias** listadas en el archivo de referencia para ese tipo de proyecto
6. **Crea `src/base/`** con las carpetas config y lib como describen las referencias
7. **Crea el primer módulo** (si el usuario especificó un dominio) siguiendo la estructura de carpetas
8. **Registra DI** — conecta ports y adapters en el contenedor de DI (módulo NestJS o Awilix)
9. **Registra rutas** — agrega las rutas del módulo a la configuración del router/navegación (excepto Next.js, que usa el filesystem)

---

## Workflow — Agregar un módulo o feature

Cuando el usuario pida agregar un módulo, feature, use case, o archivos nuevos:

1. **Detecta el tipo de proyecto** desde `package.json`, la estructura de carpetas existente, o la descripción del usuario
2. **Lee `references/universal-conventions.md` y la referencia del tipo de proyecto**
3. **Crea la estructura de carpetas del módulo** dentro de `src/modules/<domain>/` (monolito/frontend) o `src/app/` (microservicio)
4. **Genera los archivos** siguiendo las convenciones de nombres (tanto nombres de clases como sufijos de archivo)
5. **Respeta la regla de dependencias** — el dominio no importa de capas externas
6. **Registra en DI** — agrega los providers al módulo NestJS o regístralos en el contenedor Awilix
7. **Agrega rutas** si el módulo tiene endpoints HTTP (backend) o pantallas (frontend)
8. **Genera tests** en `tests/` espejando la ruta de `src/`: `src/.../foo.ts` → `tests/.../foo.spec.ts`

No te saltes los pasos 6-8 a menos que el usuario lo diga explícitamente.

---

## Archivos de referencia

- **Convenciones universales (todos los tipos de proyecto)** → lee `references/universal-conventions.md` — reglas completas y plantillas de código canónicas de todo lo resumido arriba
- **NestJS (monolito o microservicio)** → lee `references/backend-nestjs.md`
- **SPA Vue.js** → lee `references/frontend-spa-vue.md`
- **SPA React** → lee `references/frontend-spa-react.md`
- **React Native (Expo)** → lee `references/frontend-mobile-react-native.md`
- **Next.js** → lee `references/frontend-nextjs.md`

---

## Mantenimiento de esta skill

Este SKILL.md es deliberadamente delgado: se carga completo en cada trigger, así que cada
línea aquí cuesta contexto en cada tarea de código. Las reglas completas y las plantillas de
código viven en `references/universal-conventions.md`, que se lee bajo demanda. Mantén ese
contrato:

- **Si una regla se viola repetidamente en la práctica** (el agente se salta la referencia y
  genera algo mal), el fix es **subir esa regla específica al digest de arriba** — un bullet,
  sin bloques de código. NO pegues la sección completa de vuelta en este archivo.
- **Las convenciones nuevas** van en `references/universal-conventions.md` (o en la
  referencia del tipo de proyecto si son específicas del framework), más a lo sumo un bullet
  en el digest de aquí.
- Nunca dupliques plantillas de código entre este archivo y las referencias — la fuente de
  la verdad es la referencia; el drift es peor que la verbosidad.
- Toda edición a este archivo debe espejarse en español en `README.es.md` (ver la skill
  `create-skill`, que lo automatiza).
