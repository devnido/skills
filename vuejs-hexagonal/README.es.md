> **Nota:** este archivo es la traducción en español de `SKILL.md` para lectura humana.
> No es una skill registrada — Claude Code solo carga `SKILL.md`. No lo uses como
> contexto operativo.

# Skill: Arquitectura Hexagonal en Vue.js

Esta skill impone una arquitectura hexagonal (Ports & Adapters) consistente, con vertical
slicing y MVVM, en **SPAs de Vue.js**. Este archivo es el router y el resumen de reglas.
**Antes de generar cualquier archivo o carpeta, lee AMBOS archivos de referencia:**

1. `references/vue-conventions.md` — reglas canónicas y plantillas de código (Result,
   clases base, `Paginated`, reglas de entidades/VOs/ports, form & screen mappers, auth,
   testing, env/constantes/utils). **Obligatorio para toda tarea** — el resumen de abajo lo
   sintetiza, pero la referencia es la fuente de la verdad.
2. `references/frontend-spa-vue.md` — setup específico de Vue, estructura del proyecto,
   cliente HTTP, DI con Awilix, MVVM, router, layouts y un ejemplo completo de módulo.

## Alcance

| Aplica a | NO aplica a |
|---|---|
| SPA de Vue 3 (Composition API) + Vue Router + Pinia + Pinia Colada, construida con Vite | NestJS, React SPA, React Native, Next.js → usa la skill `hexagonal-architecture` |

Confirma que el proyecto es una SPA de Vue mirando el `package.json` (`vue`, `vue-router`,
`vite`) o la estructura existente. **Si no puedes determinar el tipo de proyecto, pregunta
al usuario antes de generar nada.** Nunca adivines.

---

## Capas

Cada módulo tiene exactamente 4 capas:
1. **domain/** — entidades, value objects, ports (interfaces), props, excepciones de dominio
2. **application/** — casos de uso
3. **infrastructure/** — adapters (repositorios HTTP), mappers de infraestructura, DTOs/records
4. **presentation/** — screens (vistas), ViewModels, componentes, models, mappers, store, rutas

El **lado driving** de la SPA es la capa de presentación: el **ViewModel es el adapter
driving** (el análogo en SPA de un controller de backend) — es el único lugar que resuelve
casos de uso del contenedor de DI y el único que hace unwrap de un `Result`. El **lado
driven** es `infrastructure/`: repositorios HTTP que implementan los ports del dominio.

## Convenciones de nombres — Clases e interfaces

| Concepto | Convención | Ejemplo |
|---|---|---|
| Port (interfaz) | `<Entity><Role>` | `UserRepository` |
| Adapter (impl) | `<Infra><Entity><Role>` | `HttpUserRepository` |
| Caso de uso | `<Entity><Action>` | `UserCreator`, `UserFinder`, `UserUpdater` |
| Carpeta del caso de uso | `<entity>-<action>/` | `user-creator/` |
| Contenedor paginado (dominio) | `Paginated<T>` | `Paginated<User>` |
| Excepción de dominio | `<Entity><Reason>Exception` | `UserNotFoundException`, `UserAlreadyExistsException` |
| Excepción HTTP | `HttpServiceException` | `HttpServiceException` |
| Props | `<Action><Entity>Props` | `CreateUserProps`, `FindUserProps` |
| Props de función del port | `<FunctionName>Props` | `FindByEmailProps`, `SaveUserProps` |
| Value object | `<Entity><Field>ValueObject` | `UserEmailValueObject` |
| Screen (View) | `<Screen>Screen.vue` | `UserProfileScreen.vue` |
| ViewModel | `use<Screen>ViewModel.ts` | `useUserProfileViewModel.ts` |
| Store (Pinia) | `<module>.store.ts` | `user.store.ts` |
| Form Model | `<Action><Entity>FormModel` | `CreateUserFormModel` |
| Form Mapper | `<Action><Entity>FormMapper` | `CreateUserFormMapper` |
| Screen Mapper | `<Screen>Mapper` | `LoginScreenMapper`, `UserProfileScreenMapper` |
| Presentation Model | `<NombreLibre>Model` | `UserSummaryModel`, `UserPermissionsModel` |
| Layout | `<Scope>Layout.vue` | `PublicLayout.vue`, `PrivateLayout.vue` |
| ViewModel de layout | `use<Scope>LayoutViewModel.ts` | `usePrivateLayoutViewModel.ts` |

## Convenciones de nombres — Sufijos de archivo

| Concepto | Sufijo | Ejemplo |
|---|---|---|
| Entidad | `.entity.ts` | `user.entity.ts` |
| Value Object | `.value-object.ts` | `user-email.value-object.ts` |
| Port (interfaz) | `.repository.ts` / `.port.ts` | `user.repository.ts` |
| Adapter | `.repository.ts` | `http-user.repository.ts` |
| Caso de uso | `.use-case.ts` | `user-creator.use-case.ts` |
| Props | `.props.ts` | `create-user.props.ts`, `find-by-email.props.ts` |
| Excepción de dominio | `.exception.ts` | `user-not-found.exception.ts` |
| Mapper de infraestructura | `.mapper.ts` | `user.mapper.ts` |
| Screen Mapper | `<screen-kebab>.mapper.ts` | `user-profile-screen.mapper.ts` |
| Presentation Model | `.model.ts` | `user-summary.model.ts` |
| Form Model | `.form-model.ts` | `create-user.form-model.ts` |
| Form Mapper | `.form-mapper.ts` | `create-user.form-mapper.ts` |
| Store | `.store.ts` | `user.store.ts` |
| Rutas | `.routes.ts` | `user.routes.ts` |
| Test | `.spec.ts` | `user-creator.use-case.spec.ts` |

Todos los nombres de archivo usan **kebab-case**, excepto los SFC de Vue (`.vue`: screens,
layouts y componentes), que son **PascalCase**. Nunca uses camelCase para nombres de archivo.

## Reglas — Resumen

Las reglas completas y las plantillas de código canónicas viven en
`references/vue-conventions.md`. Resumen de lo no negociable:

- **Manejo de errores (patrón Result)** — `domain/` y `application/` devuelven `Result<T>`
  (`Result.ok` / `Result.err`, el lado de error siempre una subclase de `DomainException`) y
  **nunca lanzan**. Los adapters hacen `try/catch` del cliente HTTP y convierten **todo**
  fallo en `Result.err` (mapeando estados conocidos a excepciones de dominio;
  `HttpServiceException` es el fallback genérico). El **ViewModel** hace unwrap del `Result`
  dentro de su función `query` / `mutation` de Pinia Colada con
  `if (result.isErr()) throw result.getError()` — el **único `throw` de la SPA** — y mapea
  el `code` del error capturado a estado de UI; fuera de esas funciones nunca lanza, y
  `try/catch` no es su canal de errores.
- **Clases base de dominio** (`src/base/lib/domain/`) — `Command`, `Query`, `Props`
  (nominales gracias a un campo privado `_brand`) y `DomainException extends Error` (`code`
  abstracto y estable, mensaje vía `super`). Todo input y todo error extiende su base.
  Genera estos archivos desde las plantillas de la referencia — nunca improvises su forma.
  En la SPA **no** hay base `Output` ni base `Event`.
- **`Paginated<T>`** (`src/base/lib/domain/paginated.ts`) — retorno canónico para
  colecciones: `items / total / page / limit`. Sin `totalPages` — se deriva en la capa de
  presentación.
- **Entidades de dominio** — `id` es obligatorio y no nulable; `createdAt` / `updatedAt` se
  declaran solo cuando la fuente de datos los devuelve (una proyección de lectura puede no
  tener ninguno), y nunca se inventan; no hay clase base `Entity` compartida. `new <Entity>(...)` solo se permite en mappers de
  infraestructura→dominio (caso canónico), en builders de test o, excepcionalmente, en un
  caso de uso cuando todos los campos ya existen en memoria. Nunca construyas una entidad
  para datos que aún no existen — pasa los Props al adapter; él persiste y devuelve la entidad.
- **Value objects** — VO de valor único validado = clase `<Entity><Field>ValueObject`; VO de
  datos estructurales = interfaz plana sin sufijo. Los VOs de negocio compartidos por ≥2
  módulos van en `modules/shared/domain/value-objects/`; los propios de un agregado, en su
  módulo. Nunca en `src/base/` (solo maquinaria técnica).
- **Aislamiento del archivo de port** — un archivo de port
  (`domain/ports/<entity>.repository.ts`) declara **solo** la(s) interfaz(ces) del port.
  Todo tipo de apoyo que referencie — formas de argumentos y wrappers de retorno — se extrae
  a su propio archivo en `domain/props/`, nunca inline en el archivo del port.
- **Parámetros de los ports** — los ports reciben clases de dominio y siempre devuelven
  `Promise<Result<T>>`: reutiliza los Props del caso de uso cuando los parámetros coinciden
  exactamente; si no, crea un `<FunctionName>Props` dedicado en `domain/props/` (sufijo
  `.props.ts`) que contenga **solo los campos que ese método del port necesita**.
- **Base UseCase** — todo caso de uso `extends UseCase<I, O>` (nunca `implements`) y
  devuelve **datos de dominio envueltos en un `Result`** — nunca un Presentation Model, un
  DTO HTTP crudo ni un valor sin envolver.
- **Form Model y Form Mapper** — los formularios usan una interfaz FormModel en
  `presentation/models/` más un FormMapper estático en `presentation/mappers/` que la
  convierte en un `Props` de dominio, descartando campos solo de UI (p. ej.
  `passwordConfirmation`).
- **Screen Mapper y Presentation Models** — un mapper por pantalla (`<Screen>Mapper`),
  métodos nombrados `<source>To<target>` (nunca `toModel` / `fromModel`), los models son
  interfaces con solo los campos que la UI renderiza. Es el equivalente en SPA a un Output
  de backend.
- **MVVM (obligatorio)** — la Screen (View) es pasiva: consume únicamente lo que devuelve su
  ViewModel y nunca importa `domain/` / `application/`, el contenedor de DI, stores ni
  mappers. El ViewModel (`use<Screen>ViewModel`) recibe los parámetros de ruta como
  argumentos, se auto-inicializa (su `useQuery` corre solo — no hace falta `onMounted` para
  cargar), resuelve los casos de uso del contenedor de Awilix una sola vez arriba, construye
  clases Props, hace unwrap del `Result` y devuelve **solo** Presentation Models + estado de
  UI (`isLoading`, `error` como `string | null`) + funciones de acción — nunca los objetos
  query/mutation.
- **Estado de servidor con Pinia Colada (confinado a los ViewModels)** — las lecturas usan
  `useQuery`; las escrituras usan `useMutation` e invalidan las keys que cambiaron
  (`useQueryCache().invalidateQueries`). Solo los ViewModels importan `@pinia/colada` (más
  `main.ts`, `src/base/config/colada/` y el helper de tests del ViewModel). La caché guarda
  **datos de dominio**; el ViewModel los mapea a Presentation Models con `computed` + el
  Screen Mapper. Las keys son `[module, action, ...params]` (un getter cuando un parámetro es
  reactivo). La política técnica (`staleTime`) vive en
  `src/base/config/colada/colada.options.ts`.
- **DI con Awilix** — el contenedor tipado vive en `src/base/config/di/container.ts` (no hay
  DI a nivel de framework). Los ports se registran ligados a su adapter; solo los ViewModels
  resuelven del contenedor. **Cablea cada registro a mano con
  `asFunction((cradle) => new Foo(cradle.bar))` — nunca `asClass` + `InjectionMode.CLASSIC`,
  que resuelve por los nombres de los parámetros del constructor y se rompe en cualquier
  build de producción minificado** (`Could not resolve 'e'`), mientras pasa en dev para
  siempre.
- **Constructores de casos de uso** — un caso de uso `extends UseCase`, así que cualquier
  constructor que declare **debe llamar a `super()`**; omitirlo es un error de compilación
  (TS2377).
- **Rutas del módulo** — `*.routes.ts` exporta un `RouteRecordRaw[]` (un **array**, incluso
  para una sola ruta); el router compone los módulos con spread, así que exportar un objeto
  suelto falla en runtime con `is not iterable`.
- **Store de Pinia** — guarda **solo** estado de cliente transversal entre pantallas (sesión,
  filtros recordados); los datos del servidor viven en la caché de Pinia Colada, nunca
  copiados a un store. Los ViewModels lo leen/escriben; las Views nunca lo importan. Los
  tokens nunca viven en el store.
- **Testing** — dominio: tests unitarios puros; aplicación: tests unitarios con ports
  mockeados; infraestructura: tests unitarios con el cliente HTTP mockeado; presentación:
  tests del ViewModel (sobrescribiendo los registros del contenedor con `asValue(mock)`,
  ejecutados con un helper `withSetup` que instala un Pinia + Pinia Colada nuevo por test) y
  tests de componentes con `@vue/test-utils`. Los tests viven en `tests/` replicando `src/`.
  **Genera los tests desde las plantillas canónicas** de la referencia — mockea solo en la
  costura port / caso de uso; los fixtures de entidades vienen de builders en
  `tests/modules/<module>/builders/`.
- **Regla de dependencias** (impuesta con `eslint-plugin-boundaries`) — un módulo nunca
  importa las tripas de otro módulo; el código *de negocio* compartido va a
  `modules/shared/`; `domain/` no depende de nada; `application/` → `domain/`;
  `infrastructure/` y `presentation/` → `application/` + `domain/`. **Solo las pantallas
  cruzan módulos:** una pantalla (su View + ViewModel bajo `presentation/screens/`) puede
  ejecutar casos de uso de otro módulo e importar sus Props de entrada y los tipos de
  dominio que devuelven, así un caso de uso vive una sola vez, en el módulo dueño del
  recurso. Nada más cruza módulos (componentes, stores, mappers, modelos y nunca
  `domain/` / `application/` / `infrastructure/`), y un módulo nunca duplica un port o
  adapter que pertenece a otro.
- **`src/base/`** — solo técnico transversal: `config/` (env, http, auth, logger, di,
  router), `constants/index.ts` (UPPER_SNAKE_CASE), `lib/` (clases base + `utils/` puras
  agrupadas por concern), más `lib/infrastructure/criteria/` para el lenguaje de consulta del
  backend.
- **Listados con filtros — el Criteria de la API es infraestructura** — el query string
  genérico de filtros/orden/paginación del backend es el lenguaje de consulta de la API remota,
  no dominio del SPA. El caso de uso recibe un `Find<Entities>Props` con solo los filtros que
  ofrece la UI (valores de negocio, opcional = sin filtro, sin valores de UI como `'all'`), el
  port es `find(props)` y el caso de uso lo pasa directo; un `<Entity>QueryMapper` por módulo
  convierte el Props en un `ApiCriteria` compartido, serializado por `ApiCriteriaSerializer`
  (`base/lib/infrastructure/criteria/`). Nunca `matching(criteria)` ni value objects de
  Criteria en `domain/`.
- **Variables de entorno** — el agente crea `.env.example` (versionado); el **prefijo
  `VITE_` es obligatorio** (solo `VITE_*` llega a `import.meta.env`); solo
  `src/base/config/env/` lee `import.meta.env`. Nunca pongas un secreto en una variable
  `VITE_` — todo viaja al navegador.
- **Auth y sesión** — los tokens son maquinaria técnica en `src/base/config/auth/` (port
  `TokenStorage` + adapter de `localStorage`); el cliente HTTP base inyecta `Authorization`
  y cabeceras de identidad estables, y ejecuta un **refresh single-flight ante un 401** con
  un cliente pelado; el estado de sesión vive en el store del módulo auth (los tokens nunca
  entran al store); login/registro son un módulo `modules/auth/` normal. El cliente HTTP
  nunca navega — navegar es trabajo de presentación.
- **Calidad de código** — hooks pre-commit con `husky` + `lint-staged` que corren lint y format.

---

## Flujo — Crear un proyecto Vue nuevo

1. **Confirma que es una SPA de Vue** con el usuario si no es obvio.
2. **Verifica la documentación oficial** — antes de hacer scaffolding, consulta
   https://vuejs.org/guide/quick-start para confirmar la forma recomendada de crear el
   proyecto y la versión vigente. Los comandos y flags de CLI cambian entre versiones; nunca
   te fíes solo de los comandos del archivo de referencia.
3. **Lee `references/vue-conventions.md` y `references/frontend-spa-vue.md`.**
4. **Haz el scaffolding** con el comando verificado (TypeScript, Vue Router, Pinia, ESLint,
   Prettier).
5. **Instala las dependencias** listadas en la referencia de Vue (Tailwind, Axios, Awilix,
   zod, Pinia Colada, Vitest, `@vue/test-utils`, `eslint-plugin-boundaries`, husky, lint-staged).
6. **Crea `src/base/`** con `config/` y `lib/` como describen las referencias, e instala
   Pinia Colada en `main.ts` con las opciones de `src/base/config/colada/`.
7. **Configura la imposición de límites** (ESLint) y los hooks de Husky + lint-staged.
8. **Crea el primer módulo** (si el usuario indicó un dominio) siguiendo la estructura de
   carpetas y las 4 capas.
9. **Registra la DI** — liga ports con adapters en el contenedor de Awilix.
10. **Registra las rutas** — agrega las rutas del módulo a `src/base/config/router/`.

## Flujo — Agregar un módulo o feature

1. **Confirma que el proyecto es una SPA de Vue** por el `package.json` / la estructura.
2. **Lee `references/vue-conventions.md` y `references/frontend-spa-vue.md`.**
3. **Crea la carpeta del módulo** dentro de `src/modules/<domain>/` con sus 4 capas.
4. **Genera los archivos** siguiendo las convenciones de nombres (nombres de clase *y*
   sufijos de archivo).
5. **Respeta la regla de dependencias** — `domain/` no importa nada de capas externas.
6. **Aplica MVVM** — cada pantalla nueva tiene su ViewModel; la View queda pasiva; los datos
   del servidor pasan por Pinia Colada dentro del ViewModel.
7. **Registra en la DI** — agrega el port/adapter y el caso de uso al contenedor de Awilix.
8. **Agrega las rutas** al `*.routes.ts` del módulo y regístralas en el router.
9. **Genera los tests** en `tests/` replicando `src/`: `src/.../foo.ts` →
   `tests/.../foo.spec.ts`.

No omitas los pasos 6-9 salvo que el usuario lo pida explícitamente.

---

## Archivos de referencia

- **Reglas canónicas y plantillas de código** → lee `references/vue-conventions.md`
- **Setup de Vue, estructura, DI, MVVM, router, layouts, ejemplo completo de módulo** → lee
  `references/frontend-spa-vue.md`

## Skills relacionadas

- `hexagonal-architecture` — la misma arquitectura para NestJS, React SPA, React Native y
  Next.js. **No** cubre Vue: esta skill es la única fuente para SPAs de Vue. Úsala cuando la
  tarea **no** sea una SPA de Vue.
- `criteria-pattern` — el Criteria de dominio de un backend (cualquier consulta detrás de un
  único `matching(criteria)`). En una SPA de Vue el Criteria de la API es infraestructura (ver
  el resumen y `vue-conventions.md`); usa esa skill solo si una pantalla ofrece un constructor
  de consultas libre.

---

## Mantenimiento de esta skill

Este SKILL.md es deliberadamente delgado: se carga completo en cada activación, así que cada
línea cuesta contexto en toda tarea de código. Las reglas completas y las plantillas viven en
`references/vue-conventions.md`, que se lee bajo demanda. Mantén ese contrato:

- **Si una regla se sigue violando en la práctica** (el agente se salta la referencia y
  genera algo mal), la solución es **promover esa regla concreta al resumen de arriba** — un
  bullet, sin bloques de código. NO pegues la sección completa de vuelta en este archivo.
- **Las convenciones nuevas** van a `references/vue-conventions.md` (o a
  `references/frontend-spa-vue.md` si son específicas del framework), más como mucho un
  bullet de resumen aquí.
- Nunca dupliques plantillas de código entre este archivo y las referencias — la única
  fuente de verdad es la referencia; la divergencia es peor que la verbosidad.
- Esta skill deriva de `hexagonal-architecture`. Cuando cambie una regla **compartida** en
  cualquiera de las dos (Result, clases base, entidades, VOs, ports, testing, boundaries),
  revisa la otra para que no diverjan en silencio.
- Cualquier edición de este archivo debe reflejarse en español en `README.es.md` (ver la
  skill `create-skill`, que lo automatiza).
