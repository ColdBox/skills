---
name: coldbox-wirebox-aop
description: "Use this skill when implementing Aspect-Oriented Programming (AOP) in ColdBox with WireBox, creating method interceptors for cross-cutting concerns like logging, performance monitoring, security authorization, caching, transaction management, retry logic, or rate limiting using before/after/around/onException advice."
---

# WireBox Aspect-Oriented Programming (AOP)

## Overview

WireBox AOP enables separation of cross-cutting concerns from business logic through method interception. Interceptors wrap method calls to add behavior before, after, or around execution without modifying the target class.

## Language Mode Reference

Examples use **BoxLang (`.bx`)** syntax by default. Adapt for your target language:

| Concept | BoxLang (`.bx`) | CFML (`.cfc`) |
|---------|-----------------|---------------|
| Class declaration | `class [extends="..."] {` | `component [extends="..."] {` |
| DI annotation | `@inject` above `property name="svc";` | `property name="svc" inject="svc";` |
| View templates | `.bxm` suffix | `.cfm` / `.cfml` suffix |
| Tag prefix | `<bx:if>`, `<bx:output>`, `<bx:set>` | `<cfif>`, `<cfoutput>`, `<cfset>` |

> **CFML Compat Mode**: With BoxLang + CFML Compat module, `.bx` and `.cfc` files coexist freely. BoxLang-native classes use `class {}` (`.bx` files); CFML-compat classes use `component {}` (`.cfc` files).

## Core AOP Concepts

| Term | Meaning |
|------|---------|
| **Aspect** | The cross-cutting concern (logging, security, caching) |
| **Join Point** | Method execution point where the aspect applies |
| **Advice** | The code that runs at the join point (before/after/around) |
| **Pointcut** | Expression defining which methods to intercept |
| **Interceptor** | Component implementing the aspect |
| **Weaving** | Process of applying aspects to target objects |

## Advice Types

- **before** — runs before method execution
- **after** — runs after method execution (has access to result)
- **around** — wraps method execution; must call `invocation.proceed()` to run the original method
- **onException** — runs when the method throws an exception

## Wiring Interceptors to Objects

```boxlang
// config/WireBox.cfc
class {

    function configure() {

        // Single interceptor
        map( "UserService" )
            .to( "models.UserService" )
            .asSingleton()
            .addInterceptor( "LoggingInterceptor" )

        // Multiple interceptors — executed in declaration order
        map( "OrderService" )
            .to( "models.OrderService" )
            .asSingleton()
            .addInterceptor( "SecurityInterceptor" )
            .addInterceptor( "LoggingInterceptor" )
            .addInterceptor( "PerformanceInterceptor" )
    }
}
```

## Invocation Object API

```boxlang
function before( invocation ) {
    var methodName = invocation.getMethod()   // method being called
    var args       = invocation.getArgs()     // struct of arguments
    var target     = invocation.getTarget()   // the target object
    var metadata   = invocation.getTargetMD() // metadata of the target
}

function around( invocation ) {
    // Must call proceed() to execute the original method
    var result = invocation.proceed()
    return result
}
```

## Basic Logging Interceptor

```boxlang
/**
 * interceptors/LoggingInterceptor.cfc
 */
class {

    property name="log" inject="logbox:logger:{this}"

    function before( invocation ) {
        log.debug( "Calling #invocation.getMethod()# with #serializeJSON( invocation.getArgs() )#" )
    }

    function after( invocation, result ) {
        log.debug( "#invocation.getMethod()# completed" )
        return result
    }

    function onException( invocation, exception ) {
        log.error( "#invocation.getMethod()# failed: #exception.message#" )
        rethrow
    }
}
```

## Performance Monitoring

```boxlang
/**
 * interceptors/PerformanceInterceptor.cfc
 */
class {

    property name="log" inject="logbox:logger:{this}"

    function around( invocation ) {
        var start      = getTickCount()
        var methodName = invocation.getMethod()

        try {
            var result   = invocation.proceed()
            var duration = getTickCount() - start

            if ( duration > 1000 ) {
                log.warn( "SLOW method: #methodName# (#duration#ms)" )
            } else {
                log.debug( "Method: #methodName# (#duration#ms)" )
            }

            return result
        } catch ( any e ) {
            log.error( "#methodName# failed after #getTickCount() - start#ms" )
            rethrow
        }
    }
}
```

## Audit Logging

```boxlang
/**
 * interceptors/AuditInterceptor.cfc
 */
class {

    property name="auditService" inject="AuditService"
    property name="authService"  inject="cbauth@cbauth"

    function after( invocation, result ) {
        var methodName = invocation.getMethod()

        // Only audit mutating methods
        if ( methodName.findNoCase( "create" ) ||
             methodName.findNoCase( "update" ) ||
             methodName.findNoCase( "delete" ) ) {

            auditService.log(
                action:    methodName,
                user:      authService.isLoggedIn() ? authService.getUser().getID() : "anonymous",
                data:      invocation.getArgs(),
                timestamp: now()
            )
        }

        return result
    }
}
```

## Security Authorization (Annotation-Based)

```boxlang
/**
 * interceptors/SecurityInterceptor.cfc
 * Reads @secured annotation from method metadata.
 */
class {

    property name="cbsecurity" inject="@CBSecurity"
    property name="log"        inject="logbox:logger:{this}"

    function before( invocation ) {
        var method   = invocation.getMethod()
        var target   = invocation.getTarget()
        var metadata = getMetadata( target[ method ] )

        if ( structKeyExists( metadata, "secured" ) ) {
            var requiredRole = metadata.secured

            if ( !cbsecurity.has( requiredRole ) ) {
                log.warn( "Unauthorized access attempt to #method#" )
                throw( type: "SecurityException", message: "Access denied to #method#" )
            }
        }
    }
}

/**
 * Usage — annotate service methods
 */
class singleton {

    /**
     * @secured admin
     */
    function deleteUser( userID ) { ... }

    /**
     * @secured manager,admin
     */
    function approveOrder( orderID ) { ... }
}
```

## Transaction Management

```boxlang
/**
 * interceptors/TransactionInterceptor.cfc
 * Wraps @transactional methods in a DB transaction.
 */
class {

    property name="log" inject="logbox:logger:{this}"

    function around( invocation ) {
        var method   = invocation.getMethod()
        var target   = invocation.getTarget()
        var metadata = getMetadata( target[ method ] )

        if ( !structKeyExists( metadata, "transactional" ) ) {
            return invocation.proceed()
        }

        transaction {
            try {
                var result = invocation.proceed()
                transaction action="commit"
                return result
            } catch ( any e ) {
                transaction action="rollback"
                log.error( "Transaction rolled back for #method#: #e.message#" )
                rethrow
            }
        }
    }
}

// Usage
class singleton {
    /**
     * @transactional true
     */
    function transferFunds( fromID, toID, amount ) {
        debit( fromID, amount )
        credit( toID, amount )
    }
}
```

## Caching Aspect

```boxlang
/**
 * interceptors/CacheInterceptor.cfc
 * Caches results of methods annotated with @cacheable {minutes}.
 */
class {

    property name="cache" inject="cachebox:default"
    property name="log"   inject="logbox:logger:{this}"

    function around( invocation ) {
        var method   = invocation.getMethod()
        var target   = invocation.getTarget()
        var metadata = getMetadata( target[ method ] )

        if ( !structKeyExists( metadata, "cacheable" ) ) {
            return invocation.proceed()
        }

        var cacheKey    = method & "_" & hash( serializeJSON( invocation.getArgs() ) )
        var cachedResult = cache.get( cacheKey )

        if ( !isNull( cachedResult ) ) {
            log.debug( "Cache HIT for #method#" )
            return cachedResult
        }

        log.debug( "Cache MISS for #method#" )
        var result = invocation.proceed()
        cache.set( cacheKey, result, val( metadata.cacheable ) )

        return result
    }
}

// Usage
class singleton {
    /**
     * @cacheable 60
     */
    function getAppConfig() {
        // expensive DB fetch — cached for 60 minutes
    }
}
```

## Retry Logic

```boxlang
/**
 * interceptors/RetryInterceptor.cfc
 * Retries @retry {n} times on exception with exponential back-off.
 */
class {

    property name="log" inject="logbox:logger:{this}"

    function around( invocation ) {
        var method   = invocation.getMethod()
        var target   = invocation.getTarget()
        var metadata = getMetadata( target[ method ] )

        if ( !structKeyExists( metadata, "retry" ) ) {
            return invocation.proceed()
        }

        var maxRetries = val( metadata.retry )
        var attempt    = 0

        while ( attempt < maxRetries ) {
            try {
                return invocation.proceed()
            } catch ( any e ) {
                attempt++
                if ( attempt >= maxRetries ) {
                    log.error( "#method# failed after #attempt# attempt(s)" )
                    rethrow
                }
                log.warn( "#method# attempt #attempt#/#maxRetries# failed — retrying..." )
                sleep( 1000 * attempt )  // exponential back-off
            }
        }
    }
}

// Usage
class singleton {
    /**
     * @retry 3
     */
    function callExternalAPI() { ... }
}
```

## Combining Multiple Interceptors

```boxlang
// config/WireBox.cfc
map( "PaymentService" )
    .to( "models.PaymentService" )
    .asSingleton()
    .addInterceptor( "SecurityInterceptor" )    // 1st — check access
    .addInterceptor( "TransactionInterceptor" ) // 2nd — wrap in transaction
    .addInterceptor( "AuditInterceptor" )       // 3rd — log the action
    .addInterceptor( "PerformanceInterceptor" ) // 4th — measure timing
```

## AOP Best Practices

- **Keep interceptors focused** — one concern per interceptor
- **Declare order deliberately** — security before transaction, audit after execution
- **Use `around` for transaction/retry/caching** — needs control over execution flow
- **Use `before` for authorization/validation** — can abort before the method runs
- **Use `after`/`onException` for audit logging** — can inspect result or exception
- **Avoid heavy logic in `before`** — it runs on every call; cache lookups if needed
- **Always `rethrow` in `onException`** — unless you intentionally swallow the error
