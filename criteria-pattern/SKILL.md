---
name: criteria-pattern
description: >-
  Applies the Criteria design pattern (CodelyTV "Design Patterns: Criteria") inside a
  microservice built with hexagonal architecture / DDD. Encapsulates dynamic search —
  filters + ordering + pagination — in framework-agnostic domain objects (Criteria,
  Filters/Filter, Order) and translates them to a concrete backend through a converter
  (Criteria→SQL, Criteria→Elasticsearch, Criteria→TypeORM/Mongo/Doctrine/Hibernate),
  so repositories expose a single `matching(criteria)` method instead of a pile of
  `findByX` methods (respecting SOLID/OCP). Use this skill whenever the user asks to
  "aplica/aplicar el patrón Criteria", "patrón criteria", "implementa criteria",
  "añade criteria a este repositorio", "añade criteria a un microservicio",
  "buscador con filtros dinámicos", "filtros + orden + paginación en el repositorio",
  "evitar findByX / métodos findBy en el repositorio", "convertir criteria a SQL",
  "criteria to sql", "criteria to elasticsearch", "criteria converter", "apply the
  Criteria pattern", "criteria pattern", "add criteria to this repository / service",
  "dynamic filtering in the repository", "build a search use case with filters and
  pagination", or any equivalent phrase about adding criteria-based, composable querying
  to a service. Trigger even if "hexagonal" or "DDD" are not named, as long as the task
  is about composable filter/order/paginate querying behind a repository. Do NOT trigger
  for one-off hardcoded queries the user explicitly wants kept simple, or for ORM/SQL
  questions unrelated to building a reusable criteria abstraction.
---

# Criteria Pattern for Microservices

Apply the **Criteria design pattern** to a microservice so that *searching* (filtering,
ordering, paginating) lives in the domain as composable, framework-agnostic objects, and
each persistence backend gets a dedicated **converter** that turns a `Criteria` into its
native query. This is the pattern taught in CodelyTV's *Design Patterns: Criteria* course.

## Why (the problem it solves)

Repositories that grow `findByName`, `findByEmail`, `findByNameAndEmailContaining`,
`findByStatusOrderedByDate`… violate the Open/Closed Principle and explode combinatorially.
The Criteria pattern replaces them with **one** method:

```ts
matching(criteria: Criteria): Promise<Entity[]>;
```

The caller composes a `Criteria` (filters + order + pagination); the repository's converter
translates it to SQL / Elasticsearch / ORM. New searches require **no new repository method**.

## When to use

Trigger when the user wants composable filter/order/paginate querying behind a repository,
or asks to add/convert Criteria, build a search use case with dynamic filters, or stop
writing `findByX` methods. See the frontmatter `description` for the full trigger list.

Do **not** use it for a single hardcoded query the user wants to keep trivial, or for
generic ORM/SQL questions that aren't about building a reusable criteria abstraction.

## Required inputs (ask only if not inferable)

1. **Target stack / language** — default **TypeScript + hexagonal** (matches the course and
   the user's NestJS work). Also supported as variants: TypeORM, Mongo, Doctrine/PHP,
   Hibernate/Java. Confirm only if the repo's language is ambiguous.
2. **Entity / bounded context** being searched (e.g. `User`, `Order`, `Product`) and its
   `contexts/<context>/<module>` location.
3. **Backend(s)** to convert to (SQL/MySQL, Elasticsearch, TypeORM, Mongo, …). Default to
   the persistence the repo already uses.
4. **Searchable fields** and whether **pagination** (offset and/or cursor) and **joins** are
   needed for this iteration.

If the user gives the entity and an existing repo, infer the rest and proceed.

## Building blocks and where they live (hexagonal layers)

Place the **shared, reusable** Criteria toolkit under a shared context so every module reuses it:

```
src/contexts/shared/domain/criteria/        ← pure domain, NO framework imports
  Criteria.ts            (filters + order + optional pageSize/pageNumber)
  Filters.ts  Filter.ts  FilterField.ts  FilterOperator.ts  FilterValue.ts
  Order.ts    OrderBy.ts OrderType.ts
src/contexts/shared/infrastructure/criteria/ ← one converter per backend
  CriteriaToSqlConverter.ts
  CriteriaToElasticsearchConverter.ts
```

Per searched entity:

```
src/contexts/<ctx>/<module>/domain/<Entity>Repository.ts   ← interface: matching(criteria)
src/contexts/<ctx>/<module>/infrastructure/<Backend><Entity>Repository.ts ← uses converter
src/contexts/<ctx>/<module>/domain/<Search>CriteriaFactory.ts ← names complex searches
src/contexts/<ctx>/<module>/application/search.../<Entity>Searcher.ts ← use case
```

**Rule:** domain Criteria objects must never import the ORM/driver/framework. Only the
converter (infrastructure) knows the backend. This keeps the pattern portable and testable.

## Procedure

1. **Locate the architecture.** If a `hexagonal-architecture` skill or project `CLAUDE.md`
   exists, follow its conventions for folders/naming. Detect the language and persistence
   already in use.
2. **Scaffold the shared toolkit** in `shared/domain/criteria/` using the canonical classes
   in `references/criteria-toolkit.md` (Criteria, Filters, Filter, FilterField,
   FilterOperator with the `Operator` enum: `EQUAL =`, `NOT_EQUAL !=`, `GT >`, `LT <`,
   `CONTAINS`, `NOT_CONTAINS`; FilterValue, Order, OrderBy, OrderType). Add `pageSize` /
   `pageNumber` to `Criteria` only if pagination is in scope (validate: page number requires
   page size).
3. **Define/Update the repository port** in the entity's domain with
   `matching(criteria: Criteria): Promise<Entity[]>` — do not add `findByX` methods.
4. **Write the converter** in `shared/infrastructure/criteria/` for each backend. **Never
   interpolate filter values directly into the query string** — use parameter binding /
   placeholders to avoid SQL injection (the course explicitly fixes this; treat it as
   mandatory). Map each `Operator` to the backend's syntax (e.g. `CONTAINS` → `LIKE %..%`
   in SQL, `match`/`wildcard` in Elasticsearch).
5. **Implement the infrastructure repository** so `matching` calls the converter and runs
   the resulting query.
6. **(Optional) CriteriaFactory** for recurring/complex searches so call sites read as
   intent (e.g. `PossibleScamUsersCriteriaFactory`) instead of raw filter arrays.
7. **Build the application use case** (`<Entity>Searcher`) that receives primitives from the
   transport layer, builds the `Criteria` via `Criteria.fromPrimitives(...)`, and calls
   `matching`.
8. **Test it.** Unit-test the converter (Criteria in → expected query string/params out) and
   the domain objects; use object mothers for fixtures, mirroring the course's test style.

## Conventions / rules

- Domain criteria classes are **immutable value objects** (`readonly` fields) with
  `fromPrimitives` / `toPrimitives` for crossing the boundary.
- One `Criteria` object = filters + order + (optional) pagination. Nothing backend-specific.
- A converter is the **only** place that knows a backend; add a new converter per backend
  rather than branching inside the domain.
- Parameterize values in queries — no string interpolation of user input.
- Keep the shared toolkit truly shared; do not duplicate it per module.
- For complex/nested filters, NOT operators, or named searches, extend `Filter`/`Filters`
  and use a `CriteriaFactory` — keep call sites declarative.

## References

- `references/criteria-toolkit.md` — canonical, copy-ready implementations of every core
  class and a SQL + Elasticsearch converter skeleton, plus PHP/Java porting notes. Read it
  before scaffolding so the generated code matches the course's proven shape.
