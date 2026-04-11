---
name: coldbox-logbox-patterns
description: Use this skill when configuring LogBox logging in ColdBox, setting up appenders (console, file, rolling file, socket), creating category-specific loggers, injecting loggers into services with WireBox, using structured logging with extra data, or configuring log levels per environment.
---

# LogBox Logging Patterns

## When to Use This Skill

Use this skill when setting up and using LogBox — the ColdBox logging framework — for structured, configurable application logging.

## Language Mode Reference

Examples use **BoxLang (`.bx`)** syntax by default. Adapt for your target language:

| Concept | BoxLang (`.bx`) | CFML (`.cfc`) |
|---------|-----------------|---------------|
| Class declaration | `class [extends="..."] {` | `component [extends="..."] {` |
| DI annotation | `@inject` above `property name="svc";` | `property name="svc" inject="svc";` |
| View templates | `.bxm` suffix | `.cfm` / `.cfml` suffix |
| Tag prefix | `<bx:if>`, `<bx:output>`, `<bx:set>` | `<cfif>`, `<cfoutput>`, `<cfset>` |

> **CFML Compat Mode**: With BoxLang + CFML Compat module, `.bx` and `.cfc` files coexist freely. BoxLang-native classes use `class {}` (`.bx` files); CFML-compat classes use `component {}` (`.cfc` files).

## Core Concepts

LogBox provides:
- **Appenders** — output destinations (console, file, DB, socket, etc.)
- **Loggers** — named logging channels mapped to categories
- **Log levels** — `FATAL`, `ERROR`, `WARN`, `INFO`, `DEBUG`, `TRACE`
- **WireBox integration** — inject loggers anywhere via DSL
- **Structured logging** — log with extra contextual data

## LogBox Configuration

```boxlang
// config/LogBox.cfc (standalone) OR inline in ColdBox.cfc
class LogBox extends coldbox.system.logging.config.LogBoxConfig {

    function configure() {

        // ----- Appenders -----
        appenders = {
            // Console output (great for development + Docker)
            console : {
                class    : "coldbox.system.logging.appenders.ConsoleAppender",
                levelMin : "DEBUG",
                levelMax : "FATAL"
            },

            // Flat file appender
            fileApp : {
                class    : "coldbox.system.logging.appenders.FileAppender",
                levelMin : "INFO",
                levelMax : "FATAL",
                properties : {
                    filePath : "#getDirectoryFromPath( getBaseTemplatePath() )#logs",
                    fileName : "application"
                }
            },

            // Rolling file appender
            rollingApp : {
                class    : "coldbox.system.logging.appenders.RollingFileAppender",
                levelMin : "WARN",
                levelMax : "FATAL",
                properties : {
                    filePath        : "#getDirectoryFromPath( getBaseTemplatePath() )#logs",
                    fileName        : "app",
                    fileMaxSize     : 5000,
                    fileMaxArchives : 3
                }
            }
        }

        // ----- Root Logger -----
        root = {
            levelMin  : "WARN",
            levelMax  : "FATAL",
            appenders : "rollingApp"
        }

        // ----- Category Loggers -----
        categories = {
            // All models
            "models" : {
                levelMin  : "DEBUG",
                levelMax  : "INFO",
                appenders : "console,fileApp"
            },

            // Specific service
            "models.UserService" : {
                levelMin  : "DEBUG",
                levelMax  : "DEBUG",
                appenders : "console"
            },

            // SQL queries
            "cborm" : {
                levelMin  : "DEBUG",
                levelMax  : "DEBUG",
                appenders : "console"
            }
        }
    }
}
```

**CFML (`.cfc`):**

```cfml
// config/LogBox.cfc (standalone) OR inline in ColdBox.cfc
component extends="coldbox.system.logging.config.LogBoxConfig" {

    function configure() {

        // ----- Appenders -----
        appenders = {
            // Console output (great for development + Docker)
            console : {
                class    : "coldbox.system.logging.appenders.ConsoleAppender",
                levelMin : "DEBUG",
                levelMax : "FATAL"
            },

            // Flat file appender
            fileApp : {
                class    : "coldbox.system.logging.appenders.FileAppender",
                levelMin : "INFO",
                levelMax : "FATAL",
                properties : {
                    filePath : "#getDirectoryFromPath( getBaseTemplatePath() )#logs",
                    fileName : "application"
                }
            },

            // Rolling file appender
            rollingApp : {
                class    : "coldbox.system.logging.appenders.RollingFileAppender",
                levelMin : "WARN",
                levelMax : "FATAL",
                properties : {
                    filePath        : "#getDirectoryFromPath( getBaseTemplatePath() )#logs",
                    fileName        : "app",
                    fileMaxSize     : 5000,
                    fileMaxArchives : 3
                }
            }
        }

        // ----- Root Logger -----
        root = {
            levelMin  : "WARN",
            levelMax  : "FATAL",
            appenders : "rollingApp"
        }

        // ----- Category Loggers -----
        categories = {
            // All models
            "models" : {
                levelMin  : "DEBUG",
                levelMax  : "INFO",
                appenders : "console,fileApp"
            },

            // Specific service
            "models.UserService" : {
                levelMin  : "DEBUG",
                levelMax  : "DEBUG",
                appenders : "console"
            },

            // SQL queries
            "cborm" : {
                levelMin  : "DEBUG",
                levelMax  : "DEBUG",
                appenders : "console"
            }
        }
    }
}
```

## Logger Injection in Services

```boxlang
class UserService {

    // Inject a logger for this class category
    @inject( "logBox:logger:{this}" )
    property name="log";

    // Inject the root logger
    @inject( "logBox:root" )
    property name="rootLog";

    // Inject a named logger
    @inject( "logbox:logger:models.UserService" )
    property name="userLog";

    function create( data ) {
        log.info( "Creating user: #data.email#" )

        try {
            var user = userRepository.create( data )
            log.info( "User created with ID #user.getId()#" )
            return user
        } catch( any e ){
            log.error( "Failed to create user: #e.message#", e )
            rethrow
        }
    }

    function authenticate( email, password ) {
        log.debug( "Authentication attempt for: #email#" )

        var user = userRepository.findByEmail( email )
        if( isNull( user ) ){
            log.warn( "Authentication failed - user not found: #email#" )
            return false
        }

        var success = passwordEncoder.matches( password, user.getPasswordHash() )
        if( !success ){
            log.warn( "Authentication failed - invalid password for: #email#" )
        } else {
            log.info( "User authenticated: #email#" )
        }

        return success
    }
}
```

**CFML (`.cfc`):**

```cfml
component {

    // Inject a logger for this class category
    property name="log" inject="logBox:logger:{this}";

    // Inject the root logger
    property name="rootLog" inject="logBox:root";

    // Inject a named logger
    property name="userLog" inject="logbox:logger:models.UserService";

    function create( data ) {
        log.info( "Creating user: #data.email#" )

        try {
            var user = userRepository.create( data )
            log.info( "User created with ID #user.getId()#" )
            return user
        } catch( any e ){
            log.error( "Failed to create user: #e.message#", e )
            rethrow
        }
    }

    function authenticate( email, password ) {
        log.debug( "Authentication attempt for: #email#" )

        var user = userRepository.findByEmail( email )
        if( isNull( user ) ){
            log.warn( "Authentication failed - user not found: #email#" )
            return false
        }

        var success = passwordEncoder.matches( password, user.getPasswordHash() )
        if( !success ){
            log.warn( "Authentication failed - invalid password for: #email#" )
        } else {
            log.info( "User authenticated: #email#" )
        }

        return success
    }
}
```

## Log Levels and When to Use Them

```boxlang
class ExampleService {

    @inject( "logBox:logger:{this}" )
    property name="log";

    function demonstrate() {
        // TRACE — very fine-grained diagnostic info
        log.trace( "Entering method with arg: #arg#" )

        // DEBUG — diagnostic info useful during development
        log.debug( "Query result count: #results.recordcount#" )

        // INFO — significant business events
        log.info( "User #userId# subscribed to plan #planId#" )

        // WARN — something unexpected but still working
        log.warn( "Deprecated API used by #callerInfo#" )

        // ERROR — an error condition, but app continues
        log.error( "Payment gateway timeout for order #orderId#", exception )

        // FATAL — severe error, application may not continue
        log.fatal( "Database connection pool exhausted" )

        // Check if level is enabled before expensive operations
        if( log.canDebug() ){
            var expensiveData = buildDebugData()
            log.debug( "Debug data: #expensiveData.toString()#" )
        }
    }
}
```

**CFML (`.cfc`):**

```cfml
component {

    property name="log" inject="logBox:logger:{this}";

    function demonstrate() {
        // TRACE — very fine-grained diagnostic info
        log.trace( "Entering method with arg: #arg#" )

        // DEBUG — diagnostic info useful during development
        log.debug( "Query result count: #results.recordcount#" )

        // INFO — significant business events
        log.info( "User #userId# subscribed to plan #planId#" )

        // WARN — something unexpected but still working
        log.warn( "Deprecated API used by #callerInfo#" )

        // ERROR — an error condition, but app continues
        log.error( "Payment gateway timeout for order #orderId#", exception )

        // FATAL — severe error, application may not continue
        log.fatal( "Database connection pool exhausted" )

        // Check if level is enabled before expensive operations
        if( log.canDebug() ){
            var expensiveData = buildDebugData()
            log.debug( "Debug data: #expensiveData.toString()#" )
        }
    }
}
```

## Structured Logging with Extra Data

```boxlang
class OrderService {

    @inject( "logBox:logger:{this}" )
    property name="log";

    function processOrder( order ) {
        // Log with structured extra data (JSON-friendly)
        log.info(
            "Order processing started",
            {
                orderId    : order.getId(),
                userId     : order.getUserId(),
                total      : order.getTotal(),
                itemCount  : order.getItems().len(),
                timestamp  : now()
            }
        )

        try {
            var result = paymentService.charge( order )

            log.info(
                "Payment successful",
                {
                    orderId         : order.getId(),
                    transactionId   : result.transactionId,
                    amount          : result.amount
                }
            )
        } catch( PaymentException e ){
            log.error(
                "Payment failed",
                {
                    orderId : order.getId(),
                    error   : e.message,
                    code    : e.errorCode
                }
            )
            rethrow
        }
    }
}
```

**CFML (`.cfc`):**

```cfml
component {

    property name="log" inject="logBox:logger:{this}";

    function processOrder( order ) {
        // Log with structured extra data (JSON-friendly)
        log.info(
            "Order processing started",
            {
                orderId    : order.getId(),
                userId     : order.getUserId(),
                total      : order.getTotal(),
                itemCount  : order.getItems().len(),
                timestamp  : now()
            }
        )

        try {
            var result = paymentService.charge( order )

            log.info(
                "Payment successful",
                {
                    orderId         : order.getId(),
                    transactionId   : result.transactionId,
                    amount          : result.amount
                }
            )
        } catch( PaymentException e ){
            log.error(
                "Payment failed",
                {
                    orderId : order.getId(),
                    error   : e.message,
                    code    : e.errorCode
                }
            )
            rethrow
        }
    }
}
```

## Environment-Specific Log Configuration

```boxlang
// config/ColdBox.cfc
logbox = {
    appenders : {
        console : {
            class    : "coldbox.system.logging.appenders.ConsoleAppender",
            levelMin : "DEBUG",
            levelMax : "FATAL"
        },
        rolling : {
            class    : "coldbox.system.logging.appenders.RollingFileAppender",
            levelMin : "WARN",
            levelMax : "FATAL",
            properties : {
                filePath : "#getDirectoryFromPath( getBaseTemplatePath() )#logs",
                fileName : "app"
            }
        }
    },
    root       : { levelMin : "WARN", levelMax : "FATAL", appenders : "rolling" },
    categories : {}
}

// Override for development (in development() method)
function development() {
    // Log everything to console in development
    logbox.root = { levelMin : "DEBUG", levelMax : "FATAL", appenders : "console" }
}

// Override for production
function production() {
    // Only WARN+ to file in production
    logbox.root = { levelMin : "WARN", levelMax : "FATAL", appenders : "rolling" }
}
```

## Custom Appenders and Layouts

```boxlang
// models/logging/SlackAppender.bx
class SlackAppender extends coldbox.system.logging.AbstractAppender {

    function init( name, properties = {}, layout = "" ) {
        super.init( argumentCollection = arguments )
        return this
    }

    function onRegistration() {
        variables.webhookUrl = getProperty( "webhookUrl", "" )
    }

    function logMessage( logEvent ) {
        if( !len( variables.webhookUrl ) ){
            return
        }

        http url=variables.webhookUrl method="POST" {
            httpparam type="body" value=serializeJSON( {
                text : "[#logEvent.getSeverity()#] #logEvent.getMessage()#"
            } )
        }
    }

    function onUnRegistration() {}
}
```

```boxlang
// models/logging/JsonLayout.bx
class JsonLayout extends coldbox.system.logging.AbstractLayout {

    function format( logEvent ) {
        return serializeJSON( {
            severity  : logEvent.getSeverity(),
            category  : logEvent.getCategory(),
            message   : logEvent.getMessage(),
            extraInfo : logEvent.getExtraInfo()
        } )
    }
}
```

Use custom LogBox extensions when the built-in appenders or layouts do not match your delivery target:
- Extend `AbstractAppender` for new sinks such as Slack, webhooks, queues, or observability platforms
- Extend `AbstractLayout` for JSON or vendor-specific log formats
- Keep appender setup in configuration and keep transport logic isolated inside the appender class

## LogBox Best Practices

- Use `@inject( "logBox:logger:{this}" )` to get a logger named after the current class
- Check `log.canDebug()` / `log.canInfo()` before building expensive log messages
- Use structured extra data (struct argument) for machine-parseable logs
- Configure category loggers to match your package structure for granular control
- In production: WARN+ to file, nothing to console (performance)
- In development: DEBUG+ to console for live feedback
- Never log passwords, tokens, or PII — redact before logging
- Use rolling file appenders in production with size/archive limits to manage disk
- Prefer custom appenders and layouts over duplicating transport/formatting logic throughout the app
