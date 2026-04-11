---
name: coldbox-wirebox-di
description: Use this skill when implementing dependency injection in ColdBox with WireBox, using property/constructor/setter injection, configuring the WireBox binder, working with injection DSL (coldbox:, provider:, model:), mapping interfaces to implementations, or creating singleton vs transient instances.
---

# WireBox Dependency Injection

## When to Use This Skill

Use this skill when wiring dependencies in ColdBox applications using WireBox — the built-in IoC/DI framework.

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

WireBox provides:
- **Property injection** — via `@inject` annotations (most common)
- **Constructor injection** — arguments passed to `init()`
- **Setter injection** — via setter method calls
- **Injection DSL** — special token syntax for non-model objects
- **Binder configuration** — explicit mappings and factory methods
- **Singleton vs transient** — scope management

## Property Injection (Most Common)

```boxlang
class UserService {

    // Inject by type name (auto-discover)
    @inject
    property name="userRepository";

    // Inject with explicit mapping name
    @inject( "UserRepository" )
    property name="userRepo";

    // Inject a specific interface implementation
    @inject( "IPaymentGateway" )
    property name="paymentGateway";

    // Inject a ColdBox-specific object via DSL
    @inject( "coldbox:setting:appName" )
    property name="appName";

    @inject( "coldbox:moduleSettings:myModule" )
    property name="moduleSettings";

    @inject( "coldbox:cacheBox:default" )
    property name="cache";

    @inject( "coldbox:logBox" )
    property name="logBox";

    @inject( "logBox:logger:{this}" )
    property name="log";

    // Inject ColdBox itself
    @inject( "coldbox" )
    property name="controller";

    function init() {
        return this
    }
}
```

**CFML (`.cfc`):**

```cfml
component {

    // Inject by type name (auto-discover)
    property name="userRepository" inject="userRepository";

    // Inject with explicit mapping name
    property name="userRepo" inject="UserRepository";

    // Inject a specific interface implementation
    property name="paymentGateway" inject="IPaymentGateway";

    // Inject a ColdBox-specific object via DSL
    property name="appName" inject="coldbox:setting:appName";

    property name="moduleSettings" inject="coldbox:moduleSettings:myModule";

    property name="cache" inject="coldbox:cacheBox:default";

    property name="logBox" inject="coldbox:logBox";

    property name="log" inject="logBox:logger:{this}";

    // Inject ColdBox itself
    property name="controller" inject="coldbox";

    function init() {
        return this
    }
}
```

## Constructor Injection

```boxlang
class OrderService {

    function init( required OrderRepository orderRepo, required UserService userService ) {
        variables.orderRepo  = arguments.orderRepo
        variables.userService = arguments.userService
        return this
    }

    function create( data ) {
        var user  = userService.getById( data.userId )
        var order = orderRepo.create( data )
        return order
    }
}
```

**CFML (`.cfc`):**

```cfml
component {

    function init( required OrderRepository orderRepo, required UserService userService ) {
        variables.orderRepo  = arguments.orderRepo
        variables.userService = arguments.userService
        return this
    }

    function create( data ) {
        var user  = userService.getById( data.userId )
        var order = orderRepo.create( data )
        return order
    }
}
```

## WireBox Injection DSL

```boxlang
// Model injection (default - by class name or mapping)
@inject( "UserService" )
@inject( "model:UserService" )

// ColdBox framework objects
@inject( "coldbox" )                                 // ColdBox controller
@inject( "coldbox:appMapping" )                      // App mapping
@inject( "coldbox:setting:key" )                     // ColdBox setting
@inject( "coldbox:setting:key@moduleName" )          // Module setting
@inject( "coldbox:moduleSettings:myModule" )         // All module settings
@inject( "coldbox:flash" )                           // Flash RAM
@inject( "coldbox:requestContext" )                  // Event object
@inject( "coldbox:interceptorService" )              // Interceptor service

// CacheBox
@inject( "cachebox" )                                // CacheBox itself
@inject( "cachebox:default" )                        // Default cache
@inject( "cachebox:template" )                       // Template cache
@inject( "cachebox:myCache" )                        // Named cache

// LogBox
@inject( "logbox" )                                  // LogBox itself
@inject( "logbox:root" )                             // Root logger
@inject( "logbox:logger:com.example.MyService" )     // Named logger
@inject( "logBox:logger:{this}" )                    // Logger for current class

// Java classes
@inject( "java:java.util.UUID" )

// ID injection (for named mappings)
@inject( id = "mySpecialService" )
```

## WireBox Binder

```boxlang
// config/WireBox.cfc
class WireBox extends coldbox.system.ioc.config.Binder {

    function configure() {

        // Map by class path (auto-singleton)
        mapPath( "models.UserService" ).asSingleton()

        // Map with explicit ID
        map( "userService" ).to( "models.UserService" ).asSingleton()

        // Map interface to implementation
        map( "IPaymentGateway" )
            .to( "models.StripePaymentGateway" )
            .asSingleton()

        // Map with constructor arguments
        map( "apiClient" )
            .to( "models.APIClient" )
            .asSingleton()
            .initWith(
                apiKey  : getSetting( "apiKey" ),
                baseURL : getSetting( "apiBaseURL" )
            )

        // Transient (new instance per injection)
        map( "OrderForm" ).to( "models.forms.OrderForm" ).asTransient()

        // Factory method
        map( "dbConnection" )
            .toFactoryMethod( "models.DBFactory", "createConnection" )
            .asSingleton()

        // Provider (lazy factory)
        map( "myComponent" )
            .toProvider( "models.MyComponentProvider" )
    }
}
```

**CFML (`.cfc`):**

```cfml
// config/WireBox.cfc
component extends="coldbox.system.ioc.config.Binder" {

    function configure() {

        // Map by class path (auto-singleton)
        mapPath( "models.UserService" ).asSingleton()

        // Map with explicit ID
        map( "userService" ).to( "models.UserService" ).asSingleton()

        // Map interface to implementation
        map( "IPaymentGateway" )
            .to( "models.StripePaymentGateway" )
            .asSingleton()

        // Map with constructor arguments
        map( "apiClient" )
            .to( "models.APIClient" )
            .asSingleton()
            .initWith(
                apiKey  : getSetting( "apiKey" ),
                baseURL : getSetting( "apiBaseURL" )
            )

        // Transient (new instance per injection)
        map( "OrderForm" ).to( "models.forms.OrderForm" ).asTransient()

        // Factory method
        map( "dbConnection" )
            .toFactoryMethod( "models.DBFactory", "createConnection" )
            .asSingleton()

        // Provider (lazy factory)
        map( "myComponent" )
            .toProvider( "models.MyComponentProvider" )
    }
}
```

## Singleton vs Transient

```boxlang
// Singleton — one instance per application (default for most services)
map( "UserService" ).to( "models.UserService" ).asSingleton()

// Transient — new instance every injection
map( "UserForm" ).to( "models.forms.UserForm" ).asTransient()

// Request scope — one instance per HTTP request
map( "ShoppingCart" ).to( "models.ShoppingCart" ).asRequestScope()

// Session scope
map( "UserSession" ).to( "models.UserSession" ).asSessionScope()
```

## Manual Injection with WireBox API

```boxlang
class MyHandler extends coldbox.system.EventHandler {

    @inject
    property name="wirebox";

    function createWidget( event, rc, prc ) {
        // Manually get an instance
        var widget = wirebox.getInstance( "models.Widget" )

        // Get with override arguments
        var service = wirebox.getInstance(
            name         = "models.MailService",
            initArguments = { debug: true }
        )

        // Get by DSL
        var cache = wirebox.getInstance( dsl = "cachebox:default" )
    }
}
```

**CFML (`.cfc`):**

```cfml
component extends="coldbox.system.EventHandler" {

    property name="wirebox" inject="wirebox";

    function createWidget( event, rc, prc ) {
        // Manually get an instance
        var widget = wirebox.getInstance( "models.Widget" )

        // Get with override arguments
        var service = wirebox.getInstance(
            name         = "models.MailService",
            initArguments = { debug: true }
        )

        // Get by DSL
        var cache = wirebox.getInstance( dsl = "cachebox:default" )
    }
}
```

## Lazy Injection with Provider

```boxlang
class UserService {

    // Provider wraps an expensive service in a lazy loader
    @inject( "provider:ExpensivePDFService" )
    property name="pdfProvider";

    function generateReport( user ) {
        if( user.hasPDFAccess() ){
            // Only gets created when actually called
            return pdfProvider.$get().generate( user )
        }
        return ""
    }
}
```

**CFML (`.cfc`):**

```cfml
component {

    // Provider wraps an expensive service in a lazy loader
    property name="pdfProvider" inject="provider:ExpensivePDFService";

    function generateReport( user ) {
        if( user.hasPDFAccess() ){
            // Only gets created when actually called
            return pdfProvider.$get().generate( user )
        }
        return ""
    }
}
```

## Auto-Wiring Models (autoMapModels)

```boxlang
// ColdBox.cfc — enable auto-mapping for models/ directory
wirebox = {
    enabled       : true,
    singletonReload : true
}

// All models discovered automatically from models/ folder
// No binder needed for most use cases
```

## Child Injectors and Explicit Child DSL

```boxlang
// Ask a specific child injector for an instance
var tenantService = wirebox.getInstance(
    name     = "TenantService",
    injector = "tenantA"
)

// Inject directly from a named child injector
class TenantGateway {

    @inject( "wirebox:child:tenantA:TenantService" )
    property name="tenantService";
}
```

## Lazy Properties, Delegators, and the Object Populator

```boxlang
class ReportService {

    property name="formatter" lazy;

    function buildFormatter() {
        return wirebox.getInstance( "ReportFormatter" )
    }

    function hydrateUser( payload ) {
        return wirebox.getObjectPopulator().populateFromStruct(
            target         = wirebox.getInstance( "UserDTO" ),
            memento        = payload,
            composeRelationships = true
        )
    }
}
```

Use these advanced WireBox features when the default injection patterns are not enough:
- **Child injectors** for multi-tenant, modular, or hierarchical DI lookups
- **Lazy properties** when an expensive collaborator should only be created on first access
- **Delegators** when you want composition with automatic method forwarding instead of inheritance
- **Object populator** when hydrating DTOs, entities, or forms from structs, JSON, XML, or query data

## WireBox Best Practices

- Use `@inject` property annotations as the primary injection style
- Use `@inject( "logBox:logger:{this}" )` for class-specific loggers
- Register services as singletons (default) — only use transient for stateful/form objects
- Prefer constructor injection for required dependencies (fails fast if missing)
- Use the WireBox binder only for: interface-to-implementation mapping, factory methods, or complex initialization
- Avoid `getInstance()` in services — inject dependencies instead
- Use `provider:` DSL for expensive, potentially-unused dependencies
- Reach for child injectors only when you need explicit hierarchy boundaries; keep the default injector as the norm
- Use the object populator for controlled hydration instead of ad hoc property-copy loops
