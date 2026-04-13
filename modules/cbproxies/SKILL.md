---
name: cbproxies
description: >
  Use this skill when creating dynamic Java-compatible proxies of ColdBox/BoxLang components using
  cbproxies. Covers ProxyFactory injection, creating proxies for Java interfaces, lazy-loading
  components, method interception, and integration with Java frameworks that require interface
  implementations (JDBC, Hibernate, Java threading, etc.).
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# CBProxies Skill

## When to Use This Skill

Load this skill when:
- Passing a CFML/BoxLang component to a Java API that requires a Java interface implementation
- Creating lazily-instantiated wrappers that defer object creation until first method call
- Implementing Java functional interfaces (Runnable, Callable, Comparator) with BoxLang code
- Wrapping components for transparent logging, caching, or transaction interception

## Installation

```bash
box install cbproxies
```

## Core API

### Injection

```js
property name="proxyFactory" inject="ProxyFactory@cbproxies";
```

### Create a Java Interface Proxy

```js
// Wrap a BoxLang/CFML component so Java can call it as a Java interface
var proxy = proxyFactory.create(
    target    = myComponent,            // the BoxLang component instance
    interfaces = [ "java.lang.Runnable" ]
)

// Pass to a Java thread
var thread = createObject( "java", "java.lang.Thread" ).init( proxy )
thread.start()
```

### Lazy Proxy

```js
// The target is not instantiated until the first method call
var lazyProxy = proxyFactory.createLazy(
    targetClass = "models.HeavyService",
    interfaces  = [ "models.IHeavyService" ]
)
```

### Method Interceptor (Decorator)

```js
// Intercept all method calls on the target
var loggingProxy = proxyFactory.createIntercepted(
    target    = myService,
    interceptors = [
        {
            // Called before each method
            before : function( methodName, args ) {
                log.debug( "Calling #methodName#" )
            },
            // Called after each method
            after : function( methodName, result, args ) {
                log.debug( "#methodName# returned: #serializeJSON( result )#" )
            }
        }
    ]
)
```

## Production Patterns

### Implement Java Comparator

```js
// Sort a Java collection using a BoxLang comparator
class ProductComparator {
    numeric function compare( a, b ) {
        return a.getPrice() < b.getPrice() ? -1 : ( a.getPrice() > b.getPrice() ? 1 : 0 )
    }
}

var comparator = proxyFactory.create(
    target    = new ProductComparator(),
    interfaces = [ "java.util.Comparator" ]
)

var javaList = createObject( "java", "java.util.ArrayList" ).init()
products.each( ( p ) => javaList.add( p ) )

createObject( "java", "java.util.Collections" ).sort( javaList, comparator )
```

### Implement Runnable for Background Task

```js
class EmailTask {

    property name="mailService" inject="MailService@cbmailservices";

    void function run() {
        mailService.send( pendingMails )
    }
}

var runnable = proxyFactory.create(
    target    = wirebox.getInstance( "EmailTask" ),
    interfaces = [ "java.lang.Runnable" ]
)

var executor = createObject( "java", "java.util.concurrent.Executors" ).newSingleThreadExecutor()
executor.submit( runnable )
executor.shutdown()
```

### Lazy-Load Expensive Service

```js
// The service is only constructed when actually called
property name="reportService" inject="ReportService";

function init() {
    // Wrap in lazy proxy — avoids initialization cost at startup
    variables.reportProxy = proxyFactory.createLazy(
        targetClass = "services.HeavyReportService",
        interfaces  = [ "services.IReportService" ]
    )
}
```

## Best Practices

- **Use lazy proxies** for expensive services that are not always needed in a request
- **Prefer typed interfaces** over raw class proxying — use interface lists to enforce contract
- **Use interceptor proxies** for cross-cutting concerns (logging, metrics) rather than modifying the original service
- **Ensure thread safety** when sharing proxied components across threads — avoid mutable shared state
- **Only proxy when Java interop is needed** — for pure BoxLang/CFML use cases, use WireBox directly

## Documentation

- cbproxies: https://github.com/coldbox-modules/cbproxies
