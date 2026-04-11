---
name: coldbox-orm-database-migrations
description: "Use this skill when creating or running database migrations in ColdBox with cfmigrations and commandbox-migrations, defining schema changes with the SchemaBuilder API, creating tables and columns, adding indexes and foreign keys, writing seed data, or rolling back schema changes."
---

# Database Migrations in ColdBox

## Overview

Database migrations provide version control for your database schema. cfmigrations integrates with CommandBox, giving you `box migrate` commands and a SchemaBuilder API to define schema changes in code.

## Installation & Configuration

```bash
box install commandbox-migrations
```

```boxlang
// config/ColdBox.cfc
moduleSettings = {
    cfmigrations: {
        manager:             "cfmigrations.models.QBMigrationManager",
        migrationsDirectory: "/resources/database/migrations/",
        seedsDirectory:      "/resources/database/seeds/",
        properties: {
            defaultGrammar: "MySQLGrammar@qb",
            datasource:     "appDB"
        }
    }
}
```

## CLI Commands

```bash
# Create a new migration file
box migrate create create_users_table
box migrate create add_email_to_users_table --table=users

# Run pending migrations
box migrate up

# Rollback last batch
box migrate down

# Rollback all migrations
box migrate reset

# Rollback and re-run all
box migrate fresh

# Status of all migrations
box migrate status

# Run seeds
box migrate seed
```

## Migration Structure

```boxlang
/**
 * resources/database/migrations/2026_02_11_143000_create_users_table.cfc
 */
component {

    function up( schema, query ) {
        schema.create( "users", ( table ) => {
            table.increments( "id" )
            table.string( "firstName" )
            table.string( "lastName" )
            table.string( "email" ).unique()
            table.string( "password" )
            table.boolean( "isActive" ).default( true )
            table.timestamp( "createdAt" ).nullable()
            table.timestamp( "updatedAt" ).nullable()
        } )
    }

    function down( schema, query ) {
        schema.drop( "users" )
    }
}
```

## Schema Builder — Table Operations

```boxlang
// Create table
schema.create( "posts", ( table ) => {
    table.increments( "id" )
    table.unsignedInteger( "userId" )
    table.string( "title" )
    table.text( "content" )
    table.boolean( "published" ).default( false )
    table.timestamp( "publishedAt" ).nullable()
    table.timestamps()   // adds createdAt + updatedAt
} )

// Alter existing table
schema.alter( "posts", ( table ) => {
    table.addColumn( table.string( "slug" ).unique() )
    table.modifyColumn( "title", table.string( "title", 500 ) )
    table.dropColumn( "oldField" )
    table.renameColumn( "body", "content" )
} )

// Drop, rename, check existence
schema.drop( "posts" )
schema.dropIfExists( "posts" )
schema.rename( "posts", "articles" )

if ( schema.hasTable( "posts" ) ) { ... }
if ( schema.hasColumn( "posts", "slug" ) ) { ... }
```

## Column Types

```boxlang
// Numeric
table.increments( "id" )           // UNSIGNED INT AUTO_INCREMENT PK
table.bigIncrements( "id" )        // UNSIGNED BIGINT AUTO_INCREMENT PK
table.integer( "age" )
table.bigInteger( "largeNumber" )
table.unsignedInteger( "count" )
table.decimal( "price", 8, 2 )     // DECIMAL(8,2)
table.float( "amount" )

// String
table.string( "name" )             // VARCHAR(255)
table.string( "name", 100 )        // VARCHAR(100)
table.char( "code", 10 )
table.text( "description" )
table.mediumText( "content" )
table.longText( "body" )

// Date / Time
table.date( "birthDate" )
table.datetime( "startDate" )
table.timestamp( "createdAt" )
table.timestamps()                  // shortcut: createdAt + updatedAt
table.softDeletes()                 // adds deletedAt (nullable timestamp)
table.time( "startTime" )

// Boolean
table.boolean( "isActive" )

// JSON / Binary
table.json( "metadata" )
table.binary( "fileData" )

// Enum
table.enum( "status", [ "draft", "published", "archived" ] )

// UUID
table.uuid( "id" )
```

## Column Modifiers

```boxlang
table.string( "middleName" ).nullable()
table.boolean( "isActive" ).default( true )
table.string( "email" ).unique()
table.integer( "count" ).unsigned()
table.string( "name" ).comment( "User's full name" )

// Chain modifiers
table.string( "email" )
    .unique()
    .nullable( false )
    .comment( "User email address" )
```

## Indexes & Foreign Keys

```boxlang
// Indexes
table.index( "email" )                          // Simple index
table.index( [ "firstName", "lastName" ] )      // Composite index
table.unique( "email" )                         // Unique constraint
table.unique( [ "userId", "teamId" ] )          // Composite unique

// Foreign keys
table.unsignedInteger( "userId" )
table.foreignKey( "userId" ).references( "id" ).onTable( "users" )

// With cascade
table.foreignKey( "userId" )
    .references( "id" )
    .onTable( "users" )
    .onDelete( "CASCADE" )
    .onUpdate( "CASCADE" )

// Primary key
table.primaryKey( [ "userId", "teamId" ] )       // Composite PK
```

## Common Migration Patterns

### Add Column to Existing Table

```boxlang
component {
    function up( schema, query ) {
        schema.alter( "users", ( table ) => {
            table.addColumn( table.string( "phone" ).nullable() )
            table.addColumn( table.string( "avatar" ).nullable() )
        } )
    }

    function down( schema, query ) {
        schema.alter( "users", ( table ) => {
            table.dropColumn( "phone" )
            table.dropColumn( "avatar" )
        } )
    }
}
```

### Create Pivot Table (Many-to-Many)

```boxlang
component {
    function up( schema, query ) {
        schema.create( "user_roles", ( table ) => {
            table.unsignedInteger( "user_id" )
            table.unsignedInteger( "role_id" )
            table.primaryKey( [ "user_id", "role_id" ] )
            table.foreignKey( "user_id" ).references( "id" ).onTable( "users" ).onDelete( "CASCADE" )
            table.foreignKey( "role_id" ).references( "id" ).onTable( "roles" ).onDelete( "CASCADE" )
        } )
    }

    function down( schema, query ) {
        schema.dropIfExists( "user_roles" )
    }
}
```

### Data Migration with QB

```boxlang
component {
    function up( schema, query ) {
        // Schema change
        schema.alter( "users", ( table ) => {
            table.addColumn( table.string( "fullName" ).nullable() )
        } )

        // Data migration — populate the new column
        query.from( "users" )
            .update( {
                fullName: query.raw( "CONCAT(firstName, ' ', lastName)" )
            } )
    }

    function down( schema, query ) {
        schema.alter( "users", ( table ) => {
            table.dropColumn( "fullName" )
        } )
    }
}
```

## Seeds

```boxlang
/**
 * resources/database/seeds/UserSeeder.cfc
 */
component {

    function run( query, mockdata ) {
        // Insert fixed seed data
        query.from( "roles" ).insert( [
            { id: 1, name: "admin",  description: "System Administrator" },
            { id: 2, name: "member", description: "Regular Member" }
        ] )

        // Or use mockdata for fake users
        var users = mockdata.mock(
            $num:   10,
            id:     "uuid",
            name:   "name",
            email:  "email",
            role:   "oneof:admin:member"
        )
        query.from( "users" ).insert( users )
    }
}
```

## Migration Naming Conventions

| Pattern | Example |
|---------|---------|
| Create table | `create_{table}_table` |
| Add column | `add_{column}_to_{table}_table` |
| Remove column | `remove_{column}_from_{table}_table` |
| Rename column | `rename_{old}_to_{new}_in_{table}_table` |
| Create index | `add_{col}_index_to_{table}_table` |
| Create pivot | `create_{a}_{b}_table` |
| Data migration | `migrate_{description}_data` |
