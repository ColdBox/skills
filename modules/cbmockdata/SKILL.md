---
name: cbmockdata
description: >
  Use this skill when generating realistic fake seed/test data in ColdBox/BoxLang using the cbmockdata
  module. Covers installation, the mock() API, built-in data types, entity population, testing integration,
  and bulk generation patterns for prototyping and test suites.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# CBMockdata Skill

## When to Use This Skill

Load this skill when:
- Generating realistic seed data for development databases
- Creating large datasets for load/performance testing
- Populating test fixtures in TestBox specs
- Prototyping UI with varied, realistic content
- Generating fake API payloads for contract testing

## Installation

```bash
box install cbmockdata
```

## Language Mode Reference

| Feature | BoxLang | CFML |
|---------|---------|------|
| Struct syntax | `{}` | `{}` or `structNew()` |
| Array syntax | `[]` | `[]` or `arrayNew(1)` |
| Arrow functions | `( x ) => x.name` | Not supported |

## Core API

### Injection

```js
property name="mockDataService" inject="MockDataService@cbmockdata";
```

### Basic Generation

```js
// Generate an array of 10 user structs
var users = mockDataService.mock(
    $num  = 10,
    id    = "uuid",
    name  = "name",
    email = "email",
    age   = "num:18:65",
    bio   = "sentence:2:4"
)
```

### Built-in Data Types

| Type | Example Value |
|------|---------------|
| `uuid` | `3f2504e0-4f89-...` |
| `name` | `John Doe` |
| `fname` | `Jane` |
| `lname` | `Smith` |
| `email` | `john@example.com` |
| `url` | `https://example.com` |
| `phone` | `555-123-4567` |
| `num:min:max` | `42` (integer in range) |
| `string:n` | Random string, `n` chars |
| `sentence:min:max` | 2-4 sentence lorem ipsum |
| `lorem:n` | `n` words of lorem |
| `date` | `2024-03-15` |
| `datetime` | `2024-03-15 10:45:00` |
| `boolean` | `true`/`false` |
| `imageUrl:w:h` | Placeholder image URL |
| `oneof:a:b:c` | One of the listed values |
| `country` | `United States` |
| `state` | `California` |
| `zipcode` | `90210` |
| `website` | `https://acme.com` |
| `color` | `#FF5733` |

### Fixed Values

```js
var posts = mockDataService.mock(
    $num    = 5,
    title   = "sentence:1:2",
    status  = "oneof:draft:published:archived",
    userId  = "num:1:100",
    created = "datetime"
)
```

### Nested Objects

```js
var orders = mockDataService.mock(
    $num      = 20,
    id        = "uuid",
    customer  = {
        name  = "name",
        email = "email"
    },
    total     = "num:10:500",
    items     = mockDataService.mock(
        $num    = 3,
        product = "sentence:1:2",
        qty     = "num:1:10",
        price   = "num:5:100"
    )
)
```

## Production Patterns

### TestBox Seed Helper

```js
// tests/resources/SeedHelper.bx
class {

    property name="mockDataService" inject="MockDataService@cbmockdata";

    function seedUsers( count = 10 ) {
        return mockDataService.mock(
            $num      = count,
            id        = "uuid",
            firstName = "fname",
            lastName  = "lname",
            email     = "email",
            createdAt = "datetime",
            isActive  = "boolean"
        )
    }

    function seedProducts( count = 50 ) {
        return mockDataService.mock(
            $num        = count,
            id          = "uuid",
            name        = "sentence:1:3",
            description = "sentence:2:5",
            price       = "num:1:999",
            sku         = "string:8"
        )
    }
}
```

### Database Seeder

```js
function run( event, rc, prc ) {
    var users = mockDataService.mock(
        $num     = 500,
        id       = "uuid",
        name     = "name",
        email    = "email",
        password = "string:40"     // placeholder — hash in afterEach
    )

    for ( var user in users ) {
        queryExecute(
            "INSERT INTO users (id, name, email, password) VALUES (:id, :name, :email, :password)",
            {
                id       : { value: user.id,       cfsqltype: "cf_sql_varchar" },
                name     : { value: user.name,     cfsqltype: "cf_sql_varchar" },
                email    : { value: user.email,    cfsqltype: "cf_sql_varchar" },
                password : { value: bcrypt.hashPassword( user.password ),
                             cfsqltype: "cf_sql_varchar" }
            }
        )
    }

    return "Seeded 500 users."
}
```

### API Prototype Handler

```js
function index( event, rc, prc ) {
    prc.data = mockDataService.mock(
        $num      = rc.limit ?: 20,
        id        = "uuid",
        title     = "sentence:2:5",
        body      = "sentence:4:8",
        author    = "name",
        published = "datetime",
        tags      = [
            "oneof:tech:news:health:sports",
            "oneof:tech:news:health:sports"
        ]
    )

    event.renderData( type = "json", data = prc.data )
}
```

## Best Practices

- **Use `oneof:` for status/type columns** — realistic constraint-safe values
- **Use `uuid` for primary keys** in prototyping — avoids sequence/auto-increment conflicts
- **Nest with real `mock()` calls** for related data arrays — do not hand-craft nested loops
- **Hash passwords generated with `string:`** — mock passwords should never be stored plain
- **Seed only in dev/test environments** — gate seeders with an environment check
- **Keep seed counts small in unit tests** — use large counts only in dedicated perf tests

## Documentation

- cbmockdata: https://github.com/coldbox-modules/cbmockdata
