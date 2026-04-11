---
name: coldbox-cache-integration
description: Use this skill when implementing caching in ColdBox applications using CacheBox, caching database queries, caching view/event output, configuring Redis or Couchbase cache providers, using the @cacheable annotation, or managing cache lifecycle and eviction policies.
---

# Cache Integration

## When to Use This Skill

Use this skill when adding caching strategies to a ColdBox application using CacheBox — the built-in caching framework.

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

ColdBox uses **CacheBox** for all caching, which provides:
- **Default cache** — in-memory concurrent cache
- **Template cache** — for view/event output caching
- **Named caches** — custom named caches per use case
- **Providers** — pluggable backends (RAM, Redis, Couchbase, Ehcache, etc.)
- **`@cacheable` annotation** — declarative method caching (with cborm/model integration)

## Basic Cache Operations

```boxlang
class ProductService {

    // Inject the default cache
    @inject( "cachebox:default" )
    property name="cache";

    // Inject a named cache
    @inject( "cachebox:products" )
    property name="productCache";

    // Inject CacheBox itself
    @inject( "cachebox" )
    property name="cacheBox";

    function getById( id ) {
        var cacheKey = "product_#id#"

        // Try cache first
        var product = cache.get( cacheKey )
        if( !isNull( product ) ){
            return product
        }

        // Load from DB
        product = productRepository.findById( id )

        // Store in cache (timeout=60 min, lastAccess=30 min)
        cache.set( cacheKey, product, 60, 30 )

        return product
    }

    function list( page = 1, limit = 25 ) {
        var cacheKey = "products_list_#page#_#limit#"

        var results = cache.get( cacheKey )
        if( !isNull( results ) ){
            return results
        }

        results = productRepository.list( page = page, limit = limit )
        cache.set( cacheKey, results, 30, 15 )
        return results
    }

    function clearProductCache( id ) {
        cache.clear( "product_#id#" )
        // Clear all list caches
        cache.clearByKeySnippet( "products_list_" )
    }

    function clearAll() {
        cache.clearAll()
    }
}
```

## CacheBox Configuration (BoxLang)

```boxlang
// config/CacheBox.cfc
class CacheBox extends coldbox.system.cache.config.CacheBoxConfig {

    function configure() {
        // Default cache configuration
        defaultCache = {
            provider            : "coldbox.system.cache.providers.ColdBoxCaffeineProvider",
            objectDefaultTimeout : 60,
            objectDefaultLastAccessTimeout : 30,
            useLastAccessTimeouts : true,
            reapFrequency       : 2,
            freeMemoryPercentageThreshold : 0,
            evictionPolicy      : "LRU",
            evictCount          : 1,
            maxObjects          : 200
        }

        // Template cache for view/event output caching
        caches = {
            template : {
                provider            : "coldbox.system.cache.providers.ColdBoxCaffeineProvider",
                objectDefaultTimeout : 60,
                maxObjects          : 500
            },

            // Session cache (if needed)
            sessions : {
                provider            : "coldbox.system.cache.providers.ColdBoxCaffeineProvider",
                objectDefaultTimeout : 30,
                maxObjects          : 1000
            }
        }
    }
}
```

## Redis Cache Provider (BoxLang)

```boxlang
// config/CacheBox.cfc — Redis configuration
class CacheBox extends coldbox.system.cache.config.CacheBoxConfig {

    function configure() {
        defaultCache = {
            provider : "coldbox.system.cache.providers.ColdBoxCaffeineProvider",
            objectDefaultTimeout : 60,
            maxObjects           : 200
        }

        caches = {
            // Redis for distributed/persistent caching
            redis : {
                provider   : "cbRedis.models.providers.RedisProvider",
                properties : {
                    host     : getSystemSetting( "REDIS_HOST", "localhost" ),
                    port     : getSystemSetting( "REDIS_PORT", 6379 ),
                    password : getSystemSetting( "REDIS_PASSWORD", "" ),
                    objectDefaultTimeout : 60
                }
            },

            // Use Redis for sessions
            sessions : {
                provider   : "cbRedis.models.providers.RedisProvider",
                properties : {
                    host     : getSystemSetting( "REDIS_HOST", "localhost" ),
                    port     : getSystemSetting( "REDIS_PORT", 6379 ),
                    objectDefaultTimeout : 30
                }
            }
        }
    }
}
```

## Event/View Output Caching

```boxlang
class Catalog extends coldbox.system.EventHandler {

    // Cache entire handler events
    this.EVENT_CACHE_SUFFIX = "catalog"

    function index( event, rc, prc ) {

        // Cache this event output for 60 minutes
        event.setEventCacheableEntry(
            provider   = "template",
            timeout    = 60,
            lastAccess = 30
        )

        prc.categories = categoryService.list()
        event.setView( "catalog/index" )
    }

    function show( event, rc, prc ) {

        // Cache per product ID
        event.setEventCacheableEntry(
            provider   = "template",
            timeout    = 120,
            suffix     = rc.id ?: 0
        )

        prc.product = productService.getById( rc.id ?: 0 )
        event.setView( "catalog/show" )
    }
}
```

```cfml
<!--- View fragment caching --->
#renderView(
    view                 = "shared/featured_products",
    cache                = true,
    cacheTimeout         = 30,
    cacheLastAccessTimeout = 15
)#
```

## Query Caching

```boxlang
function getTopProducts() {
    return queryExecute(
        "SELECT * FROM products WHERE active = 1 ORDER BY views DESC LIMIT 10",
        {},
        {
            cachedWithin : createTimeSpan( 0, 0, 30, 0 ), // 30 minutes
            datasource   : "myapp"
        }
    )
}
```

## @cacheable Annotation (with ColdBox Models)

```boxlang
// With cborm/Quick ORM entities
class ProductService {

    @cacheable( timeout = 60, provider = "default" )
    function getById( id ) {
        return productRepository.findById( id )
    }

    @cacheable( timeout = 30, key = "products_list_#arguments.page#" )
    function list( page = 1 ) {
        return productRepository.list( page = page )
    }

    @cacheEvict( pattern = "products_" )
    function save( product ) {
        return productRepository.save( product )
    }
}
```

## Cache-Aside Pattern

```boxlang
function getOrCache( key, timeout = 60, producer ) {
    var value = cache.get( key )

    if( !isNull( value ) ){
        return value
    }

    value = producer()
    cache.set( key, value, timeout )
    return value
}

// Usage
var user = getOrCache(
    "user_#id#",
    60,
    () => userRepository.findById( id )
)
```

## Cache Best Practices

- Cache at the service layer, not the handler/view layer (except for view fragment caching)
- Include relevant parameters in cache keys (`"product_#id#_#locale#"`) for uniqueness
- Clear related cache entries when data changes via `clearByKeySnippet()`
- Use event output caching sparingly — it's coarse-grained and hard to invalidate
- Configure Redis for distributed/clustered deployments
- Set appropriate TTLs based on data volatility (seconds for rates, minutes for products, hours for reference data)
- Monitor cache hit rates and eviction counts to tune configuration
