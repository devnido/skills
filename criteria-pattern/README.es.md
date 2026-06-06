# Patrón Criteria para Microservicios

> ℹ️ Esta es una traducción al español de `SKILL.md` para lectura humana. **No** es
> contexto operativo para agentes: Claude Code solo carga el archivo `SKILL.md` (en inglés)
> como skill. Si editas la skill, mantén ambos archivos sincronizados.

Aplica el **patrón de diseño Criteria** a un microservicio para que la *búsqueda* (filtrar,
ordenar, paginar) viva en el dominio como objetos componibles e independientes del
framework, y cada backend de persistencia tenga un **converter** dedicado que traduzca un
`Criteria` a su query nativa. Es el patrón que enseña el curso de CodelyTV *Design Patterns:
Criteria*.

## Por qué (el problema que resuelve)

Los repositorios que acumulan `findByName`, `findByEmail`, `findByNameAndEmailContaining`,
`findByStatusOrderedByDate`… violan el principio Abierto/Cerrado y crecen de forma
combinatoria. El patrón Criteria los reemplaza por **un** método:

```ts
matching(criteria: Criteria): Promise<Entity[]>;
```

Quien llama compone un `Criteria` (filtros + orden + paginación); el converter del
repositorio lo traduce a SQL / Elasticsearch / ORM. Las búsquedas nuevas **no requieren un
nuevo método** en el repositorio.

## Cuándo usarla

Actívala cuando el usuario quiera consultas componibles de filtrar/ordenar/paginar detrás de
un repositorio, o pida añadir/convertir Criteria, construir un caso de uso de búsqueda con
filtros dinámicos, o dejar de escribir métodos `findByX`. La lista completa de disparadores
está en el campo `description` del frontmatter de `SKILL.md`.

**No** la uses para una única query hardcodeada que el usuario quiere mantener trivial, ni
para preguntas genéricas de ORM/SQL que no buscan construir una abstracción reutilizable de
criteria.

## Entradas necesarias (pregunta solo si no se pueden inferir)

1. **Stack / lenguaje objetivo** — por defecto **TypeScript + hexagonal** (coincide con el
   curso y con el trabajo del usuario en NestJS). Variantes soportadas: TypeORM, Mongo,
   Doctrine/PHP, Hibernate/Java. Confirma solo si el lenguaje del repo es ambiguo.
2. **Entidad / bounded context** que se busca (p. ej. `User`, `Order`, `Product`) y su
   ubicación `contexts/<context>/<module>`.
3. **Backend(s)** a los que convertir (SQL/MySQL, Elasticsearch, TypeORM, Mongo, …). Por
   defecto, la persistencia que ya usa el repo.
4. **Campos buscables** y si esta iteración necesita **paginación** (offset y/o cursor) y
   **joins**.

Si el usuario da la entidad y un repo existente, infiere el resto y continúa.

## Piezas y dónde viven (capas hexagonales)

Coloca el toolkit **compartido y reutilizable** de Criteria en un contexto compartido para
que todos los módulos lo reutilicen:

```
src/contexts/shared/domain/criteria/        ← dominio puro, SIN imports de framework
  Criteria.ts            (filtros + orden + pageSize/pageNumber opcionales)
  Filters.ts  Filter.ts  FilterField.ts  FilterOperator.ts  FilterValue.ts
  Order.ts    OrderBy.ts OrderType.ts
src/contexts/shared/infrastructure/criteria/ ← un converter por backend
  CriteriaToSqlConverter.ts
  CriteriaToElasticsearchConverter.ts
```

Por cada entidad buscada:

```
src/contexts/<ctx>/<module>/domain/<Entity>Repository.ts   ← interfaz: matching(criteria)
src/contexts/<ctx>/<module>/infrastructure/<Backend><Entity>Repository.ts ← usa el converter
src/contexts/<ctx>/<module>/domain/<Search>CriteriaFactory.ts ← nombra búsquedas complejas
src/contexts/<ctx>/<module>/application/search.../<Entity>Searcher.ts ← caso de uso
```

**Regla:** los objetos de dominio Criteria nunca deben importar el ORM/driver/framework.
Solo el converter (infraestructura) conoce el backend. Así el patrón es portable y testeable.

## Procedimiento

1. **Localiza la arquitectura.** Si existe una skill `hexagonal-architecture` o un
   `CLAUDE.md` del proyecto, sigue sus convenciones de carpetas/nombres. Detecta el lenguaje
   y la persistencia ya en uso.
2. **Genera el toolkit compartido** en `shared/domain/criteria/` usando las clases canónicas
   de `references/criteria-toolkit.md` (Criteria, Filters, Filter, FilterField,
   FilterOperator con el enum `Operator`: `EQUAL =`, `NOT_EQUAL !=`, `GT >`, `LT <`,
   `CONTAINS`, `NOT_CONTAINS`; FilterValue, Order, OrderBy, OrderType). Añade `pageSize` /
   `pageNumber` a `Criteria` solo si la paginación está en alcance (valida: el número de
   página requiere tamaño de página).
3. **Define/actualiza el puerto del repositorio** en el dominio de la entidad con
   `matching(criteria: Criteria): Promise<Entity[]>` — no añadas métodos `findByX`.
4. **Escribe el converter** en `shared/infrastructure/criteria/` para cada backend. **Nunca
   interpoles los valores de los filtros directamente en el string de la query** — usa
   binding de parámetros / placeholders para evitar inyección SQL (el curso lo corrige
   explícitamente; trátalo como obligatorio). Mapea cada `Operator` a la sintaxis del backend
   (p. ej. `CONTAINS` → `LIKE %..%` en SQL, `match`/`wildcard` en Elasticsearch).
5. **Implementa el repositorio de infraestructura** para que `matching` llame al converter y
   ejecute la query resultante.
6. **(Opcional) CriteriaFactory** para búsquedas recurrentes/complejas, de modo que los
   puntos de llamada se lean como intención (p. ej. `PossibleScamUsersCriteriaFactory`) en
   lugar de arrays de filtros crudos.
7. **Construye el caso de uso de aplicación** (`<Entity>Searcher`) que recibe primitivas
   desde la capa de transporte, construye el `Criteria` con `Criteria.fromPrimitives(...)` y
   llama a `matching`.
8. **Testéalo.** Haz pruebas unitarias del converter (Criteria de entrada → query/params
   esperados de salida) y de los objetos de dominio; usa object mothers para los fixtures,
   replicando el estilo de tests del curso.

## Convenciones / reglas

- Las clases de criteria del dominio son **value objects inmutables** (campos `readonly`) con
  `fromPrimitives` / `toPrimitives` para cruzar la frontera.
- Un objeto `Criteria` = filtros + orden + (opcional) paginación. Nada específico del backend.
- Un converter es el **único** lugar que conoce un backend; añade un converter nuevo por
  backend en vez de ramificar dentro del dominio.
- Parametriza los valores en las queries — sin interpolación de strings con input del usuario.
- Mantén el toolkit compartido realmente compartido; no lo dupliques por módulo.
- Para filtros complejos/anidados, operadores NOT o búsquedas con nombre, extiende
  `Filter`/`Filters` y usa una `CriteriaFactory` — mantén los puntos de llamada declarativos.

## Referencias

- `references/criteria-toolkit.md` — implementaciones canónicas y listas para copiar de cada
  clase núcleo y un esqueleto de converter SQL + Elasticsearch, más notas de portabilidad a
  PHP/Java. Léelo antes de generar el scaffold para que el código resultante siga la forma
  probada del curso.
