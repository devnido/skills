# Criteria Toolkit — copy-ready implementations

Canonical building blocks of the Criteria pattern (CodelyTV *Design Patterns: Criteria*).
Default language is **TypeScript** (hexagonal). PHP/Java porting notes are at the end.

Place these under `src/contexts/shared/domain/criteria/`. They must stay **framework-free**.

---

## Domain — value objects

### `FilterField.ts`
```ts
export class FilterField {
  constructor(public readonly value: string) {}
}
```

### `FilterValue.ts`
```ts
export class FilterValue {
  constructor(public readonly value: string) {}
}
```

### `FilterOperator.ts`
```ts
export enum Operator {
  EQUAL = "=",
  NOT_EQUAL = "!=",
  GT = ">",
  LT = "<",
  CONTAINS = "CONTAINS",
  NOT_CONTAINS = "NOT_CONTAINS",
}

export class FilterOperator {
  constructor(public readonly value: Operator) {}
}
```

### `Filter.ts`
```ts
import { FilterField } from "./FilterField";
import { FilterOperator, Operator } from "./FilterOperator";
import { FilterValue } from "./FilterValue";

export type FiltersPrimitives = {
  field: string;
  operator: string;
  value: string;
};

export class Filter {
  constructor(
    readonly field: FilterField,
    readonly operator: FilterOperator,
    readonly value: FilterValue,
  ) {}

  static fromPrimitives(field: string, operator: string, value: string): Filter {
    return new Filter(
      new FilterField(field),
      new FilterOperator(Operator[operator as keyof typeof Operator]),
      new FilterValue(value),
    );
  }

  toPrimitives(): FiltersPrimitives {
    return {
      field: this.field.value,
      operator: this.operator.value,
      value: this.value.value,
    };
  }
}
```

### `Filters.ts`
```ts
import { Filter, FiltersPrimitives } from "./Filter";

export class Filters {
  constructor(public readonly value: Filter[]) {}

  static fromPrimitives(filters: FiltersPrimitives[]): Filters {
    return new Filters(
      filters.map((f) => Filter.fromPrimitives(f.field, f.operator, f.value)),
    );
  }

  toPrimitives(): FiltersPrimitives[] {
    return this.value.map((f) => f.toPrimitives());
  }

  isEmpty(): boolean {
    return this.value.length === 0;
  }
}
```

### `OrderType.ts`
```ts
export enum OrderTypes {
  ASC = "ASC",
  DESC = "DESC",
  NONE = "NONE",
}

export class OrderType {
  constructor(public readonly value: OrderTypes) {}

  isNone(): boolean {
    return this.value === OrderTypes.NONE;
  }
}
```

### `OrderBy.ts`
```ts
export class OrderBy {
  constructor(public readonly value: string) {}
}
```

### `Order.ts`
```ts
import { OrderBy } from "./OrderBy";
import { OrderType, OrderTypes } from "./OrderType";

export class Order {
  constructor(
    public readonly orderBy: OrderBy,
    public readonly orderType: OrderType,
  ) {}

  static none(): Order {
    return new Order(new OrderBy(""), new OrderType(OrderTypes.NONE));
  }

  static fromPrimitives(orderBy: string | null, orderType: string | null): Order {
    return orderBy !== null
      ? new Order(new OrderBy(orderBy), new OrderType(orderType as OrderTypes))
      : Order.none();
  }

  isNone(): boolean {
    return this.orderType.isNone();
  }
}
```

### `Criteria.ts`
Pagination fields (`pageSize` / `pageNumber`) are optional — include them only when
pagination is in scope. Page number requires page size.
```ts
import { FiltersPrimitives } from "./Filter";
import { Filters } from "./Filters";
import { Order } from "./Order";

export class Criteria {
  constructor(
    public readonly filters: Filters,
    public readonly order: Order,
    public readonly pageSize: number | null = null,
    public readonly pageNumber: number | null = null,
  ) {
    if (pageNumber !== null && pageSize === null) {
      throw new Error("Page size is required when page number is defined");
    }
  }

  static fromPrimitives(
    filters: FiltersPrimitives[],
    orderBy: string | null,
    orderType: string | null,
    pageSize: number | null = null,
    pageNumber: number | null = null,
  ): Criteria {
    return new Criteria(
      Filters.fromPrimitives(filters),
      Order.fromPrimitives(orderBy, orderType),
      pageSize,
      pageNumber,
    );
  }

  hasFilters(): boolean {
    return !this.filters.isEmpty();
  }

  hasOrder(): boolean {
    return !this.order.isNone();
  }
}
```

---

## Domain — repository port

```ts
// src/contexts/<ctx>/<module>/domain/<Entity>Repository.ts
import { Criteria } from "../../../shared/domain/criteria/Criteria";

export interface UserRepository {
  save(user: User): Promise<void>;
  search(id: UserId): Promise<User | null>;
  matching(criteria: Criteria): Promise<User[]>; // the ONLY search method
}
```

---

## Infrastructure — converters (the only backend-aware code)

### SQL converter — **parameterized** (no string interpolation of values)
```ts
// src/contexts/shared/infrastructure/criteria/CriteriaToSqlConverter.ts
import { Criteria } from "../../domain/criteria/Criteria";
import { Operator } from "../../domain/criteria/FilterOperator";

export class CriteriaToSqlConverter {
  convert(
    fields: string[],
    table: string,
    criteria: Criteria,
  ): { query: string; params: unknown[] } {
    const params: unknown[] = [];
    let query = `SELECT ${fields.join(", ")} FROM ${table}`;

    if (criteria.hasFilters()) {
      const where = criteria.filters.value.map((filter) => {
        const field = filter.field.value;
        switch (filter.operator.value) {
          case Operator.CONTAINS:
            params.push(`%${filter.value.value}%`);
            return `${field} LIKE ?`;
          case Operator.NOT_CONTAINS:
            params.push(`%${filter.value.value}%`);
            return `${field} NOT LIKE ?`;
          default:
            params.push(filter.value.value);
            return `${field} ${filter.operator.value} ?`;
        }
      });
      query += ` WHERE ${where.join(" AND ")}`;
    }

    if (criteria.hasOrder()) {
      query += ` ORDER BY ${criteria.order.orderBy.value} ${criteria.order.orderType.value}`;
    }

    if (criteria.pageSize !== null) {
      query += ` LIMIT ?`;
      params.push(criteria.pageSize);
      if (criteria.pageNumber !== null) {
        query += ` OFFSET ?`;
        params.push(criteria.pageSize * (criteria.pageNumber - 1));
      }
    }

    return { query: `${query};`, params };
  }
}
```

### Elasticsearch converter (shape sketch)
```ts
// src/contexts/shared/infrastructure/criteria/CriteriaToElasticsearchConverter.ts
import { Criteria } from "../../domain/criteria/Criteria";
import { Operator } from "../../domain/criteria/FilterOperator";

export class CriteriaToElasticsearchConverter {
  convert(criteria: Criteria): Record<string, unknown> {
    const body: Record<string, unknown> = {};

    if (criteria.hasFilters()) {
      body.query = {
        bool: {
          must: criteria.filters.value.map((filter) => {
            const field = filter.field.value;
            const value = filter.value.value;
            switch (filter.operator.value) {
              case Operator.CONTAINS:
                return { wildcard: { [field]: `*${value}*` } };
              case Operator.NOT_EQUAL:
                return { bool: { must_not: { term: { [field]: value } } } };
              case Operator.GT:
                return { range: { [field]: { gt: value } } };
              case Operator.LT:
                return { range: { [field]: { lt: value } } };
              default:
                return { term: { [field]: value } };
            }
          }),
        },
      };
    }

    if (criteria.hasOrder()) {
      body.sort = [
        { [criteria.order.orderBy.value]: criteria.order.orderType.value.toLowerCase() },
      ];
    }

    if (criteria.pageSize !== null) {
      body.size = criteria.pageSize;
      if (criteria.pageNumber !== null) {
        body.from = criteria.pageSize * (criteria.pageNumber - 1);
      }
    }

    return body;
  }
}
```

---

## Application — search use case

```ts
// src/contexts/<ctx>/<module>/application/search/<Entity>Searcher.ts
import { Criteria } from "../../../../shared/domain/criteria/Criteria";

export class UserSearcher {
  constructor(private readonly repository: UserRepository) {}

  async search(
    filters: FiltersPrimitives[],
    orderBy: string | null,
    orderType: string | null,
    pageSize: number | null,
    pageNumber: number | null,
  ): Promise<User[]> {
    const criteria = Criteria.fromPrimitives(
      filters, orderBy, orderType, pageSize, pageNumber,
    );
    return this.repository.matching(criteria);
  }
}
```

## Domain — CriteriaFactory for named/complex searches

```ts
// src/contexts/<ctx>/<module>/domain/PossibleScamUsersCriteriaFactory.ts
import { Criteria } from "../../shared/domain/criteria/Criteria";

export class PossibleScamUsersCriteriaFactory {
  static create(): Criteria {
    return Criteria.fromPrimitives(
      [{ field: "profilePicture", operator: "EQUAL", value: "" }],
      "createdAt",
      "DESC",
      null,
      null,
    );
  }
}
```

---

## Porting notes (other backends / languages)

- **TypeORM / Mongo** — same domain objects; the converter builds a `FindOptionsWhere`
  (TypeORM) or a Mongo filter document instead of a SQL string. `CONTAINS` → `Like('%v%')`
  / `$regex`.
- **PHP (Doctrine)** — mirror the value objects as PHP classes; the converter produces a
  Doctrine `QueryBuilder` / `DQL`. The course also packages it as a library integrated with
  **Laravel** and **Symfony**.
- **Java (Hibernate)** — value objects as immutable Java classes; converter builds a JPA
  `CriteriaQuery` / `Predicate` list.

The domain layer is identical across all of them — only the converter changes. That is the
whole point of the pattern.
