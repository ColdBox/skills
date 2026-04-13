---
name: cfmigrations
description: >
  Use this skill when managing database schema changes in ColdBox/BoxLang using cfmigrations and
  its Schema Builder. Covers migration file structure, the Schema Builder API (create/alter/drop
  tables, column types, indexes, foreign keys), seed files, and running migrations via CommandBox CLI.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# CFMigrations Skill

## When to Use This Skill

Load this skill when:
- Creating or altering database tables via code-versioned migration files
- Using the fluent Schema Builder to define columns, constraints, and indexes
- Writing seed files for development and test data
- Integrating migrations into deployment pipelines
- Rolling back schema changes in a reliable, repeatable way

## Installation

```bash
box install commandbox-migrations
```

### Configure (box.json or .env)

```json
{
    "cfmigrations": {
        "schema"     : "",
        "connectionInfo": {
            "class"            : "com.mysql.cj.jdbc.Driver",
            "connectionString" : "jdbc:mysql://localhost:3306/mydb",
            "username"         : "${DB_USER}",
            "password"         : "${DB_PASS}"
        },
        "defaultGrammar" : "MySQLGrammar@qb",
        "migrationsTable": "cfmigrations"
    }
}
```

Supported grammars: `MySQLGrammar@qb`, `PostgresGrammar@qb`, `MSSQLGrammar@qb`, `SQLiteGrammar@qb`

## CLI Commands

```bash
migrate create create_users_table    # generate a new migration file
migrate up                           # run all pending migrations
migrate down                         # roll back the last batch
migrate reset                        # roll back all migrations
migrate refresh                      # reset + run all migrations
migrate seed                         # run all seed files
migrate status                       # show migration status
```

## Migration File Structure

```js
// migrations/2024_01_15_120000_create_users_table.bx
class {

    function up( schema ) {
        schema.create( "users", ( table ) => {
            table.increments( "id" )
            table.string( "name", 100 )
            table.string( "email", 255 ).unique()
            table.string( "password", 255 )
            table.boolean( "isActive" ).default( true )
            table.timestamp( "emailVerifiedAt" ).nullable()
            table.timestamps()  // created_at, updated_at
        } )
    }

    function down( schema ) {
        schema.drop( "users" )
    }
}
```

## Schema Builder Column Types

| Method | SQL Type |
|--------|----------|
| `increments( "id" )` | UNSIGNED INT AUTO_INCREMENT PRIMARY KEY |
| `bigIncrements( "id" )` | UNSIGNED BIGINT AUTO_INCREMENT PK |
| `uuid( "id" )` | VARCHAR(36) |
| `string( "col", length=255 )` | VARCHAR |
| `text( "col" )` | TEXT |
| `mediumText( "col" )` | MEDIUMTEXT |
| `longText( "col" )` | LONGTEXT |
| `integer( "col" )` | INT |
| `bigInteger( "col" )` | BIGINT |
| `unsignedInteger( "col" )` | UNSIGNED INT |
| `tinyInteger( "col" )` | TINYINT |
| `boolean( "col" )` | TINYINT(1) |
| `decimal( "col", precision, scale )` | DECIMAL |
| `float( "col" )` | FLOAT |
| `date( "col" )` | DATE |
| `time( "col" )` | TIME |
| `dateTime( "col" )` | DATETIME |
| `timestamp( "col" )` | TIMESTAMP |
| `timestamps()` | created\_at + updated\_at |
| `json( "col" )` | JSON |
| `enum( "col", ["a","b"] )` | ENUM |

## Column Modifiers

```js
table.string( "subtitle" ).nullable()
table.integer( "sortOrder" ).default( 0 )
table.string( "code" ).unique()
table.boolean( "active" ).default( true ).comment( "Controls visibility" )
table.string( "login" ).unsigned()
```

## Indexes & Foreign Keys

```js
schema.create( "posts", ( table ) => {
    table.increments( "id" )
    table.unsignedInteger( "userId" )
    table.string( "title" )
    table.longText( "body" ).nullable()
    table.timestamps()

    // Index
    table.index( "userId" )
    table.index( [ "title", "createdAt" ] )

    // Foreign key
    table.foreignKey( "userId" )
        .references( "id" )
        .on( "users" )
        .onDelete( "CASCADE" )
} )
```

## Alter Tables

```js
function up( schema ) {
    schema.alter( "users", ( table ) => {
        table.addColumn( table.string( "phone", 20 ).nullable() )
        table.addColumn( table.boolean( "twoFactorEnabled" ).default( false ) )
        table.dropColumn( "legacyField" )
        table.renameColumn( "fname", "firstName" )
        table.modifyColumn( "name", table.string( "name", 200 ) )
        table.addIndex( "phone" )
    } )
}

function down( schema ) {
    schema.alter( "users", ( table ) => {
        table.dropColumn( "phone" )
        table.dropColumn( "twoFactorEnabled" )
        table.addColumn( table.string( "legacyField", 100 ).nullable() )
    } )
}
```

## Seed Files

```js
// seeds/UserSeed.bx
class {

    property name="bcrypt" inject="BCryptService@bcrypt";

    function run( qb ) {
        qb.table( "users" ).insert( [
            {
                name     : "Admin User",
                email    : "admin@example.com",
                password : bcrypt.hashPassword( "AdminSecret1!" ),
                isActive : true
            },
            {
                name     : "Test User",
                email    : "user@example.com",
                password : bcrypt.hashPassword( "UserSecret1!" ),
                isActive : true
            }
        ] )
    }
}
```

## Best Practices

- **Each migration does one thing** — one table or one logical change per file
- **Always write `down()`** — rollback must completely undo what `up()` did
- **Never edit a deployed migration** — create a new migration to amend a schema already in production
- **Use environment variables** for connection credentials — never hardcode in config files
- **Run `migrate status`** in CI before deploying — ensures no pending migrations are missed
- **Test `down()` locally** before pushing — confirm rollback is correct
- **Use `nullable()` for new columns** on existing tables — avoids data errors on live data

## Documentation

- cfmigrations: https://github.com/coldbox-modules/cfmigrations
- commandbox-migrations: https://github.com/coldbox-modules/commandbox-migrations
