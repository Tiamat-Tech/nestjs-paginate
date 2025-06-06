# Nest.js Paginate

![Main CI](https://github.com/ppetzold/nestjs-paginate/workflows/Main%20CI/badge.svg)
[![npm](https://img.shields.io/npm/v/nestjs-paginate.svg)](https://www.npmjs.com/package/nestjs-paginate)
[![downloads](https://img.shields.io/npm/dt/nestjs-paginate.svg)](https://www.npmjs.com/package/nestjs-paginate)
[![codecov](https://codecov.io/gh/ppetzold/nestjs-paginate/branch/master/graph/badge.svg)](https://codecov.io/gh/ppetzold/nestjs-paginate)
[![code style: prettier](https://img.shields.io/badge/code_style-prettier-ff69b4.svg)](https://github.com/prettier/prettier)
[![semantic-release](https://img.shields.io/badge/%20%20%F0%9F%93%A6%F0%9F%9A%80-semantic--release-e10079.svg)](https://github.com/semantic-release/semantic-release)
![GitHub](https://img.shields.io/github/license/ppetzold/nestjs-paginate)

Pagination and filtering helper method for TypeORM repositories or query builders using [Nest.js](https://nestjs.com/) framework.

- Pagination conforms to [JSON:API](https://jsonapi.org/)
- Sort by multiple columns
- Search across columns
- Select columns
- Filter using operators (`$eq`, `$not`, `$null`, `$in`, `$gt`, `$gte`, `$lt`, `$lte`, `$btw`, `$ilike`, `$sw`, `$contains`)
- Include relations and nested relations
- Virtual column support
- Cursor-based pagination

## Installation

```
npm install nestjs-paginate
```

## Usage

### Example

The following code exposes a route that can be utilized like so:

#### Endpoint

```url
http://localhost:3000/cats?limit=5&page=2&sortBy=color:DESC&search=i&filter.age=$gte:3&select=id,name,color,age
```

#### Result

```json
{
  "data": [
    {
      "id": 4,
      "name": "George",
      "color": "white",
      "age": 3
    },
    {
      "id": 5,
      "name": "Leche",
      "color": "white",
      "age": 6
    },
    {
      "id": 2,
      "name": "Garfield",
      "color": "ginger",
      "age": 4
    },
    {
      "id": 1,
      "name": "Milo",
      "color": "brown",
      "age": 5
    },
    {
      "id": 3,
      "name": "Kitty",
      "color": "black",
      "age": 3
    }
  ],
  "meta": {
    "itemsPerPage": 5,
    "totalItems": 12,
    "currentPage": 2,
    "totalPages": 3,
    "sortBy": [["color", "DESC"]],
    "search": "i",
    "filter": {
      "age": "$gte:3"
    }
  },
  "links": {
    "first": "http://localhost:3000/cats?limit=5&page=1&sortBy=color:DESC&search=i&filter.age=$gte:3",
    "previous": "http://localhost:3000/cats?limit=5&page=1&sortBy=color:DESC&search=i&filter.age=$gte:3",
    "current": "http://localhost:3000/cats?limit=5&page=2&sortBy=color:DESC&search=i&filter.age=$gte:3",
    "next": "http://localhost:3000/cats?limit=5&page=3&sortBy=color:DESC&search=i&filter.age=$gte:3",
    "last": "http://localhost:3000/cats?limit=5&page=3&sortBy=color:DESC&search=i&filter.age=$gte:3"
  }
}
```

### Example (Cursor-based Pagination)

The following code exposes a route using cursor-based pagination:

#### Endpoint

```url
http://localhost:3000/cats?limit=5&sortBy=lastVetVisit:ASC&cursor=V998328469600000
```

#### Result

```json
{
  "data": [
    {
      "id": 3,
      "name": "Shadow",
      "lastVetVisit": "2022-12-21T10:00:00.000Z"
    },
    {
      "id": 4,
      "name": "Luna",
      "lastVetVisit": "2022-12-22T10:00:00.000Z"
    },
    {
      "id": 5,
      "name": "Pepper",
      "lastVetVisit": "2022-12-23T10:00:00.000Z"
    },
    {
      "id": 6,
      "name": "Simba",
      "lastVetVisit": "2022-12-24T10:00:00.000Z"
    },
    {
      "id": 7,
      "name": "Tiger",
      "lastVetVisit": "2022-12-25T10:00:00.000Z"
    }
  ],
  "meta": {
    "itemsPerPage": 5,
    "cursor": "V998328469600000"
  },
  "links": {
    "previous": "http://localhost:3000/cats?limit=5&sortBy=lastVetVisit:DESC&cursor=V001671616800000",
    "current": "http://localhost:3000/cats?limit=5&sortBy=lastVetVisit:ASC&cursor=V998328469600000",
    "next": "http://localhost:3000/cats?limit=5&sortBy=lastVetVisit:ASC&cursor=V998328037600000"
  }
}
```

#### Code

```ts
import { Controller, Injectable, Get } from '@nestjs/common'
import { InjectRepository } from '@nestjs/typeorm'
import { FilterOperator, FilterSuffix, Paginate, PaginateQuery, paginate, Paginated } from 'nestjs-paginate'
import { Repository, Entity, PrimaryGeneratedColumn, Column } from 'typeorm'

@Entity()
export class CatEntity {
  @PrimaryGeneratedColumn()
  id: number

  @Column('text')
  name: string

  @Column('text')
  color: string

  @Column('int')
  age: number

  @Column({ nullable: true })
  lastVetVisit: Date | null

  @CreateDateColumn()
  createdAt: string
}

@Injectable()
export class CatsService {
  constructor(
    @InjectRepository(CatEntity)
    private readonly catsRepository: Repository<CatEntity>
  ) {}

  public findAll(query: PaginateQuery): Promise<Paginated<CatEntity>> {
    return paginate(query, this.catsRepository, {
      sortableColumns: ['id', 'name', 'color', 'age'],
      nullSort: 'last',
      defaultSortBy: [['id', 'DESC']],
      searchableColumns: ['name', 'color', 'age'],
      select: ['id', 'name', 'color', 'age', 'lastVetVisit'],
      filterableColumns: {
        name: [FilterOperator.EQ, FilterSuffix.NOT],
        age: true,
      },
    })
  }
}

@Controller('cats')
export class CatsController {
  constructor(private readonly catsService: CatsService) {}

  @Get()
  public findAll(@Paginate() query: PaginateQuery): Promise<Paginated<CatEntity>> {
    return this.catsService.findAll(query)
  }
}
```

### Config

````ts
const paginateConfig: PaginateConfig<CatEntity> {
  /**
   * Required: true (must have a minimum of one column)
   * Type: (keyof CatEntity)[]
   * Description: These are the columns that are valid to be sorted by.
   */
  sortableColumns: ['id', 'name', 'color'],

  /**
   * Required: false
   * Type: 'first' | 'last'
   * Description: Define whether to put null values at the beginning
   * or end of the result set.
   */
  nullSort: 'last',

  /**
   * Required: false
   * Type: [keyof CatEntity, 'ASC' | 'DESC'][]
   * Default: [[sortableColumns[0], 'ASC]]
   * Description: The order to display the sorted entities.
   */
  defaultSortBy: [['name', 'DESC']],

  /**
   * Required: false
   * Type: (keyof CatEntity)[]
   * Description: These columns will be searched through when using the search query
   * param. Limit search scope further by using `searchBy` query param.
   */
  searchableColumns: ['name', 'color'],

  /**
   * Required: false
   * Type: (keyof CatEntity)[]
   * Default: None
   * Description: TypeORM partial selection. Limit selection further by using `select` query param.
   * https://typeorm.io/select-query-builder#partial-selection
   * Note: if you do not contain the primary key in the select array, primary key will be added automatically.
   * 
   * Wildcard support:
   * - Use '*' to select all columns from the main entity.
   * - Use 'relation.*' to select all columns from a relation.
   * - Use 'relation.subrelation.*' to select all columns from nested relations.
   * 
   * Examples:
   * select: ['*'] - Selects all columns from main entity
   * select: ['id', 'name', 'toys.*'] - Selects id, name from main entity and all columns from toys relation
   * select: ['*', 'toys.*'] - Selects all columns from both main entity and toys relation
   */
  select: ['id', 'name', 'color'],

  /**
   * Required: false
   * Type: number
   * Default: 100
   * Description: The maximum amount of entities to return per page.
   * Set it to -1, in conjunction with limit=-1 on query param, to disable pagination.
   */
  maxLimit: 20,

  /**
   * Required: false
   * Type: number
   * Default: 20
   */
  defaultLimit: 50,

  /**
   * Required: false
   * Type: TypeORM find options
   * Default: None
   * https://typeorm.io/#/find-optionsfind-options.md
   */
  where: { color: 'ginger' },

  /**
   * Required: false
   * Type: { [key in CatEntity]?: FilterOperator[] } - Operators based on TypeORM find operators
   * Default: None
   * https://typeorm.io/#/find-options/advanced-options
   */
  filterableColumns: { age: [FilterOperator.EQ, FilterOperator.IN] },

  /**
   * Required: false
   * Type: RelationColumn<CatEntity>
   * Description: Indicates what relations of entity should be loaded.
   */
  relations: [],

  /**
   * Required: false
   * Type: boolean
   * Default: false
   * Description: Load eager relations using TypeORM's eager property.
   * Only works if `relations` is not defined.
   */
  loadEagerRelations: true,

  /**
   * Required: false
   * Type: boolean
   * Description: Disables the global condition of "non-deleted" for the entity with delete date columns.
   * https://typeorm.io/select-query-builder#querying-deleted-rows
   */
  withDeleted: false,

  /**
   * Required: false
   * Type: string
   * Description: Allow user to choose between limit/offset and take/skip, or cursor-based pagination.
   * Default: PaginationType.TAKE_AND_SKIP
   * Options: PaginationType.LIMIT_AND_OFFSET, PaginationType.TAKE_AND_SKIP, PaginationType.CURSOR
   *
   * However, using limit/offset can cause problems with relations.
   */
  paginationType: PaginationType.LIMIT_AND_OFFSET,

  /**
   * Required: false
   * Type: boolean
   * Default: false
   * Description: Generate relative paths in the resource links.
   */
  relativePath: true,

  /**
   * Required: false
   * Type: string
   * Description: Overrides the origin of absolute resource links if set.
   */
  origin: 'http://cats.example',

  /**
   * Required: false
   * Type: boolean
   * Default: false
   * Description: Prevent `searchBy` query param from limiting search scope further. Search will depend upon `searchableColumns` config option only
   */
  ignoreSearchByInQueryParam: true,

  /**
   * Required: false
   * Type: boolean
   * Default: false
   * Description: Prevent `select` query param from limiting selection further. Partial selection will depend upon `select` config option only
   */
  ignoreSelectInQueryParam: true,

  /**
   * Required: false
   * Type: 'leftJoinAndSelect' | 'innerJoinAndSelect'
   * Default: 'leftJoinAndSelect'
   * Description: Relationships will be joined with either LEFT JOIN or INNER JOIN, and their columns selected. Can be specified per column with `joinMethods` configuration.
   */
  defaultJoinMethod: 'leftJoinAndSelect',

  /**
   * Required: false
   * Type: MappedColumns<T, JoinMethod>
   * Default: false
   * Description: Overrides the join method per relationship.
   */
  joinMethods: {age: 'innerJoinAndSelect', size: 'leftJoinAndSelect'},

  /**
   * Required: false
   * Type: boolean
   * Default: false
   * Description: Enable multi-word search behavior. When true, each word in the search query
   * will be treated as a separate search term, allowing for more flexible matching.
   */
  multiWordSearch: false,

  /**
   * Required: false
   * Type: (qb: SelectQueryBuilder<T>) => SelectQueryBuilder<any>
   * Default: undefined
   * Description: Callback that lets you override the COUNT query executed by
   * paginate(). The function receives a **clone** of the original QueryBuilder,
   * so it already contains every WHERE clause and parameter parsed by
   * nestjs-paginate.
   *
   * Typical use-case: remove expensive LEFT JOINs or build a lighter DISTINCT
   * count when getManyAndCount() becomes a bottleneck.
   *
   * Example:
   * ```ts
   * buildCountQuery: qb => {
   *   qb.expressionMap.joinAttributes = [];   // drop all joins
   *   qb.select('p.id').distinct(true);       // keep DISTINCT on primary key
   *   return qb;                              // paginate() will call .getCount()
   * }
   * ```
   */
  buildCountQuery: (qb: SelectQueryBuilder<T>) => SelectQueryBuilder<any>,
}
````

## Usage with Query Builder

You can paginate custom queries by passing on the query builder:

### Example

```typescript
const queryBuilder = repo
  .createQueryBuilder('cats')
  .leftJoinAndSelect('cats.owner', 'owner')
  .where('cats.owner = :ownerId', { ownerId })

const result = await paginate<CatEntity>(query, queryBuilder, config)
```

## Usage with Relations

Similar as with repositories, you can utilize `relations` as a simplified left-join form:

### Example

#### Endpoint

```url
http://localhost:3000/cats?filter.toys.name=$in:Mouse,String
```

#### Code

```typescript
const config: PaginateConfig<CatEntity> = {
  relations: ['toys'],
  sortableColumns: ['id', 'name', 'toys.name'],
  filterableColumns: {
    'toys.name': [FilterOperator.IN],
  },
}

const result = await paginate<CatEntity>(query, catRepo, config)
```

**Note:** Embedded columns on relations have to be wrapped with brackets:

```typescript
const config: PaginateConfig<CatEntity> = {
  sortableColumns: ['id', 'name', 'toys.(size.height)', 'toys.(size.width)'],
  searchableColumns: ['name'],
  relations: ['toys'],
}
```

## Usage with Nested Relations

Similar as with relations, you can specify nested relations for sorting, filtering and searching:

### Example

#### Endpoint

```url
http://localhost:3000/cats?filter.home.pillows.color=pink
```

#### Code

```typescript
const config: PaginateConfig<CatEntity> = {
  relations: { home: { pillows: true } },
  sortableColumns: ['id', 'name', 'home.pillows.color'],
  searchableColumns: ['name', 'home.pillows.color'],
  filterableColumns: {
    'home.pillows.color': [FilterOperator.EQ],
  },
}

const result = await paginate<CatEntity>(query, catRepo, config)
```

## Usage with Eager Loading

Eager loading should work with TypeORM's eager property out of the box:

### Example

#### Code

```typescript
@Entity()
export class CatEntity {
  // ...

  @OneToMany(() => CatToyEntity, (catToy) => catToy.cat, {
    eager: true,
  })
  toys: CatToyEntity[]
}

const config: PaginateConfig<CatEntity> = {
  loadEagerRelations: true,
  sortableColumns: ['id', 'name', 'toys.name'],
  filterableColumns: {
    'toys.name': [FilterOperator.IN],
  },
}

const result = await paginate<CatEntity>(query, catRepo, config)
```

## Filters

Filter operators must be whitelisted per column in `PaginateConfig`.

### Examples

#### Code

```typescript
const config: PaginateConfig<CatEntity> = {
  // ...
  filterableColumns: {
    // Enable individual operators on a column
    id: [FilterOperator.EQ, FilterSuffix.NOT],

    // Enable all operators on a column
    age: true,
  },
}
```

`?filter.name=$eq:Milo` is equivalent with `?filter.name=Milo`

`?filter.age=$btw:4,6` where column `age` is between `4` and `6`

`?filter.id=$not:$in:2,5,7` where column `id` is **not** `2`, `5` or `7`

`?filter.summary=$not:$ilike:term` where column `summary` does **not** contain `term`

`?filter.summary=$sw:term` where column `summary` starts with `term`

`?filter.seenAt=$null` where column `seenAt` is `NULL`

`?filter.seenAt=$not:$null` where column `seenAt` is **not** `NULL`

`?filter.createdAt=$btw:2022-02-02,2022-02-10` where column `createdAt` is between the dates `2022-02-02` and `2022-02-10`

`?filter.createdAt=$lt:2022-12-20T10:00:00.000Z` where column `createdAt` is before iso date `2022-12-20T10:00:00.000Z`

`?filter.roles=$contains:moderator` where column `roles` is an array and contains the value `moderator`

`?filter.roles=$contains:moderator,admin` where column `roles` is an array and contains the values `moderator` and `admin`

## Jsonb Filters

You can filter on jsonb columns by using the dot notation. Json columns is limited to `$eq` operators only.

`?filter.metadata.enabled=$eq:true` where column `metadata` is jsonb and contains an object with the key `enabled`.

## Multi Filters

Multi filters are filters that can be applied to a single column with a comparator.

### Examples

`?filter.createdAt=$gt:2022-02-02&filter.createdAt=$lt:2022-02-10` where column `createdAt` is after `2022-02-02` **and** before `2022-02-10`

`?filter.roles=$contains:moderator&filter.roles=$or:$contains:admin` where column `roles` is an array and contains `moderator` **or** `admin`

`?filter.id=$gt:3&filter.id=$and:$lt:5&filter.id=$or:$eq:7` where column `id` is greater than `3` **and** less than `5` **or** equal to `7`

**Note:** The `$and` comparators are not required. The above example is equivalent to:

`?filter.id=$gt:3&filter.id=$lt:5&filter.id=$or:$eq:7`

**Note:** The first comparator on the the first filter is ignored because the filters are grouped by the column name and chained with an `$and` to other filters.

`...&filter.id=5&filter.id=$or:7&filter.name=Milo&...`

is resolved to:

`WHERE ... AND (id = 5 OR id = 7) AND name = 'Milo' AND ...`

## Cursor-based Pagination

- `paginationType: PaginationType.CURSOR`
- Cursor format:
  - Numbers: `[prefix1][integer:11 digits][prefix2][decimal:4 digits]` (e.g., `Y00000000001V2500` for -1.25 in ASC).
  - Dates: `[prefix][value:15 digits]` (e.g., `V001671444000000` for a timestamp in DESC).
- Prefixes:
  - `null`: `A` (lowest priority, last in results).
  - ASC:
    - positive-int: `V` (greater than or equal to 1), `X` (less than 1)
    - positive-decimal: `V` (not zero), `X` (zero)
    - zero-int: `X`
    - zero-decimal: `X`
    - negative-int: `Y`
    - negative-decimal: `V`
  - DESC:
    - positive-int: `V`
    - positive-decimal: `V`
    - zero-int: `N`
    - zero-decimal: `X`
    - negative-int: `M` (less than or equal to -1), `N` (greater than -1)
    - negative-decimal: `V` (not zero), `X` (zero)
- Logic:
  - Numbers: Split into integer (11 digits) and decimal (4 digits) parts, with separate prefixes. Supports negative values, with sorting adjusted per direction.
  - Dates: Single prefix with 15-digit timestamp padded with zeros.
  - ASC: Negative → Zero → Positive → Null.
  - DESC: Positive → Zero → Negative → Null.
- Notes:
  - Multiple columns: `sortBy` can include multiple columns to create and sort by the cursor (e.g., `sortBy=age:ASC&sortBy=createdAt:DESC`), but at least one column must be unique to ensure consistent ordering.
  - Supported columns: Cursor sorting is available for numeric and date-related columns (string columns are not supported).
  - Decimal support: Numeric columns can include decimals, limited to 11 digits for the integer part and 4 digits for the decimal part.

## Swagger

You can use two default decorators @ApiOkResponsePaginated and @ApiPagination to generate swagger documentation for your endpoints

`@ApiOkPaginatedResponse` is for response body, return http[](https://) status is 200

`@ApiPaginationQuery` is for query params

```typescript
  @Get()
  @ApiOkPaginatedResponse(
    UserDto,
    USER_PAGINATION_CONFIG,
  )
  @ApiPaginationQuery(USER_PAGINATION_CONFIG)
  async findAll(
    @Paginate()
    query: PaginateQuery,
  ): Promise<Paginated<UserEntity>> {

  }
```

There is also some syntax sugar for this, and you can use only one decorator `@PaginatedSwaggerDocs` for both response body and query params

```typescript
  @Get()
  @PaginatedSwaggerDocs(UserDto, USER_PAGINATION_CONFIG)
  async findAll(
    @Paginate()
    query: PaginateQuery,
  ): Promise<Paginated<UserEntity>> {

  }
```

## Troubleshooting

The package does not report error reasons in the response bodies. They are instead
reported as `debug` level [logging](https://docs.nestjs.com/techniques/logger#logger).

Common errors include missing `sortableColumns` or `filterableColumns` (the latter only affects filtering).
