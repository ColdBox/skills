---
name: cbq
description: >
  Use this skill when implementing asynchronous background job processing in ColdBox/BoxLang with
  cbq. Covers job class creation, queue configuration (database, Redis, SQS), dispatching jobs,
  delayed dispatch, queue routing, retry handling, worker pool management, and monitoring patterns.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# CBQ Skill

## When to Use This Skill

Load this skill when:
- Offloading slow tasks (email, PDF, API calls) to background workers
- Implementing retry logic for unreliable external integrations
- Routing different job types to different queue backends
- Scheduling delayed job execution
- Building worker consumers for database or Redis queues

## Installation

```bash
box install cbq
```

## Configuration

### config/modules/cbq.cfc

```js
function configure() {
    return {
        // Default queue connection
        defaultConnection : "database",

        connections : {
            database : {
                driver    : "CBQDatabase",
                dsn       : "myDatasource",
                tableName : "cbq_jobs",
                queue     : "default",
                timeout   : 60
            },
            redis : {
                driver : "CBQRedis",
                host   : getSystemSetting( "REDIS_HOST", "127.0.0.1" ),
                port   : getSystemSetting( "REDIS_PORT", "6379" ),
                queue  : "default",
                timeout: 60
            }
        },

        // Worker configuration
        workers : {
            numThreads  : 5,
            sleep       : 1,    // seconds between polls
            maxAttempts : 3     // global default
        }
    }
}
```

### Create Database Table (cfmigrations)

```js
function up( schema ) {
    schema.create( "cbq_jobs", ( table ) => {
        table.increments( "id" )
        table.string( "queue" ).default( "default" )
        table.string( "connection" ).default( "database" )
        table.longText( "payload" )
        table.unsignedTinyInteger( "attempts" ).default( 0 )
        table.boolean( "reserved" ).default( false )
        table.dateTime( "reserved_at" ).nullable()
        table.dateTime( "available_at" )
        table.dateTime( "created_at" )
    } )

    schema.create( "cbq_failed_jobs", ( table ) => {
        table.increments( "id" )
        table.string( "connection" )
        table.string( "queue" )
        table.longText( "payload" )
        table.longText( "exception" )
        table.dateTime( "failed_at" )
    } )
}
```

## Creating Jobs

```js
// jobs/SendWelcomeEmailJob.bx
class extends="cbq.models.AbstractJob" {

    // Inject services the job needs
    property name="mailService" inject="MailService@myApp";

    // Job payload properties
    property name="userId"  type="numeric";
    property name="email"   type="string";

    // Optional — override defaults from config
    this.queue       = "emails"
    this.connection  = "default"
    this.maxAttempts = 3
    this.timeout     = 30

    function handle() {
        var user = userService.get( userId )

        mailService
            .newMail(
                to      = email,
                subject = "Welcome to the platform!",
                type    = "html"
            )
            .setBodyTemplate( "email/templates/welcome" )
            .setBodyTokens( { name: user.getName() } )
            .send()
    }

    // Called when all retries exhausted
    function failed( exception ) {
        log.error( "Failed to send welcome email to #email# after #this.maxAttempts# attempts: #exception.message#" )
    }
}
```

## Dispatching Jobs

```js
property name="sendWelcomeEmailJob" inject="SendWelcomeEmailJob";

// Dispatch immediately
sendWelcomeEmailJob
    .setUserId( newUser.getId() )
    .setEmail(  newUser.getEmail() )
    .dispatch()

// Delay execution (seconds)
sendWelcomeEmailJob
    .setUserId( rc.id )
    .setEmail(  rc.email )
    .delay( 30 )
    .dispatch()

// Specify queue and connection
sendWelcomeEmailJob
    .setUserId( rc.id )
    .setEmail(  rc.email )
    .onQueue(      "high-priority" )
    .onConnection( "redis" )
    .dispatch()
```

## Production Patterns

### Handler — Dispatch on User Registration

```js
function register( event, rc, prc ) {
    var user = userService.create( rc )

    // Dispatch async background jobs
    sendWelcomeEmailJob.setUserId( user.getId() ).setEmail( user.getEmail() ).dispatch()
    setupDefaultPreferencesJob.setUserId( user.getId() ).dispatch()

    // Respond immediately — do not wait for jobs
    messagebox.success( "Registration successful! Check your email." )
    relocate( "dashboard" )
}
```

### Chaining Jobs

```js
// Job A dispatches Job B upon completion
class extends="cbq.models.AbstractJob" {

    property name="generateReportJob" inject="GenerateReportJob";

    function handle() {
        // ... do work ...

        // Dispatch next job in the chain
        generateReportJob
            .setReportId( reportId )
            .delay( 5 )
            .dispatch()
    }
}
```

### Idempotent Job

```js
class extends="cbq.models.AbstractJob" {

    property name="cache" inject="cachebox:default";

    function handle() {
        var lockKey = "job_processed_#orderId#"

        if ( cache.get( lockKey ) ) {
            return // Already processed — skip
        }

        // Process order...
        processOrder( orderId )

        // Mark as done (TTL: 1 hour)
        cache.set( lockKey, true, 60 )
    }
}
```

## Best Practices

- **Keep jobs small and focused** — one job should do one thing
- **Make jobs idempotent** — safe to run multiple times in case of retries
- **Never trust job payload without validation** — validate properties in `handle()` before use
- **Use the `failed()` hook** to log failures and optionally alert via monitoring
- **Set appropriate `timeout`** per job type — short for emails, longer for PDF generation
- **Route by priority** — use `high` and `low` named queues with different worker counts
- **Avoid large payloads** — store IDs in the job, not entire entity serializations
- **Test job logic in unit tests** — call `handle()` directly without dispatching

## Documentation

- cbq: https://github.com/coldbox-modules/cbq
