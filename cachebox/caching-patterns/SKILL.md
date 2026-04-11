---
name: coldbox-cachebox-patterns
description: Use this skill when implementing CacheBox caching patterns in ColdBox, configuring cache providers, using cache stores with named caches, implementing cache-aside patterns, cache warming, distributed caching with Redis, or monitoring cache performance.
---

# CacheBox Caching Patterns

## When to Use This Skill

Use this skill when working with CacheBox directly — configuring cache providers, managing named caches, and implementing advanced caching patterns.

## Core Concepts

CacheBox is the standalone caching framework in ColdFusion/BoxLang that:
- Manages multiple named cache stores with different providers
- Supports RAM, Caffeine, Redis, Couchbase, Ehcache, and JDBC providers
- Provides a consistent API across all providers
- Is fully integrated with ColdBox and WireBox

## CacheBox Configuration (BoxLang)

```boxlang
// config/CacheBox.cfc
class CacheBox extends coldbox.system.cache.config.CacheBoxConfig {

    function configure() {

        // Default cache (in-memory Caffeine)
        defaultCache = {
            provider                       : "coldbox.system.cache.providers.ColdBoxCaffeineProvider",
            objectDefaultTimeout           : 60,
            objectDefaultLastAccessTimeout : 30,
            useLastAccessTimeouts          : true,
            reapFrequency                  : 2,
            evictionPolicy                 : "LRU",
            evictCount                     : 1,
            maxObjects                     : 200
        }

        caches = {
            // Template/view cache
            template : {
                provider                       : "coldbox.system.cache.providers.ColdBoxCaffeineProvider",
                objectDefaultTimeout           : 60,
                objectDefaultLastAccessTimeout : 30,
                maxObjects                     : 500
            },

            // Product catalog cache (longer TTL)
            products : {
                provider            : "coldbox.system.cache.providers.ColdBoxCaffeineProvider",
                objectDefaultTimeout : 240,  // 4 hours
                maxObjects          : 1000
            },

            // Session/auth token cache (match session timeout)
            sessions : {
                provider            : "coldbox.system.cache.providers.ColdBoxCaffeineProvider",
                objectDefaultTimeout : 30,
                maxObjects          : 5000
            }
        }
    }
}
```

## Redis Provider Configuration

```boxlang
// config/CacheBox.cfc
caches = {
    redis : {
        provider   : "cbRedis.models.providers.RedisProvider",
        properties : {
            host                 : getSystemSetting( "REDIS_HOST", "localhost" ),
            port                 : getSystemSetting( "REDIS_PORT", 6379 ),
            password             : getSystemSetting( "REDIS_PASSWORD", "" ),
            database             : 0,
            objectDefaultTimeout : 60,
            keyPrefix            : "myapp:"
        }
    }
}
```

## Cache Operations API

```boxlang
class ProductService {

    @inject( "cachebox:products" )
    property name="cache";

    @inject( "cachebox" )
    property name="cacheBox";

    // Basic get/set
    function getById( id ) {
        var key     = "product_#id#"
        var product = cache.get( key )

        if( !isNull( product ) ){
            return product
        }

        product = queryDB( id )
        cache.set( key, product, 120, 60 )
        return product
    }

    // Check and get (atomic)
    function getWithCheck( id ) {
        var key = "product_#id#"

        if( cache.lookup( key ) ){
            return cache.get( key )
        }

        var product = queryDB( id )
        cache.set( key, product )
        return product
    }

    // Get or set (cache-or-compute)
    function getOrCompute( id ) {
        return cache.getOrSet(
            objectKey  = "product_#id#",
            produce    = () => queryDB( id ),
            timeout    = 120,
            lastAccess = 30
        )
    }

    // Eviction
    function invalidate( id ) {
        cache.clear( "product_#id#" )
    }

    // Clear by prefix pattern
    function invalidateAll() {
        cache.clearByKeySnippet( "product_" )
    }

    // Cache statistics
    function getStats() {
        return cache.getStats()
    }

    // Warm the cache
    function warmCache() {
        var products = queryAllFromDB()
        products.each( ( p ) => {
            cache.set( "product_#p.id#", p, 240 )
        } )
    }
}
```

## Advanced Cache Patterns

```boxlang
// Cache-aside pattern with stampede protection
function getSafeValue( key, loader, timeout = 60 ) {
    // Double-check locking to prevent stampede
    var value = cache.get( key )
    if( !isNull( value ) ){ return value }

    lock name="cache_load_#key#" timeout="10" throwontimeout="false" {
        value = cache.get( key )
        if( isNull( value ) ){
            value = loader()
            cache.set( key, value, timeout )
        }
    }
    return value
}

// Write-through cache
function save( entity ) {
    var saved = repository.save( entity )
    cache.set( "entity_#saved.id#", saved, 60 )
    return saved
}

// Write-behind cache (async)
function saveAsync( entity ) {
    var saved = repository.save( entity )
    // Async cache update
    runAsync( () => cache.set( "entity_#saved.id#", saved, 60 ) )
    return saved
}
```

## Named Cache Injection Patterns

```boxlang
// Inject different caches based on use case
class CacheConsumer {

    @inject( "cachebox:default" )
    property name="defaultCache";

    @inject( "cachebox:template" )
    property name="templateCache";

    @inject( "cachebox:products" )
    property name="productCache";

    @inject( "cachebox:redis" )
    property name="redisCache";

    @inject( "cachebox" )
    property name="cacheBox";

    function getCacheByName( name ) {
        return cacheBox.getCache( name )
    }
}
```

## Cache Monitoring and Stats

```boxlang
function getCacheReport() {
    var report = {}

    // Get cache stats
    report.default = cache.getStats()

    // Check specific key
    report.keyExists  = cache.lookup( "my_key" )
    report.keyExpiry  = cache.getCachedObjectMetadata( "my_key" )

    // Get all cached objects
    report.keys       = cache.getKeys()
    report.objectCount = cache.getSize()

    return report
}
```

## CacheBox Best Practices

- Use named caches for different data domains (products, sessions, templates)
- Set TTLs based on data update frequency — not too long or too short
- Use `getOrSet()` pattern to eliminate boilerplate null-check code
- Clear by prefix pattern (`clearByKeySnippet`) when invalidating groups of related keys
- Use Redis for multi-server/distributed deployments
- Monitor cache hit rates — low hit rates indicate key strategy problems
- Warm critical caches on application startup (listener on `applicationStart`)
- Never cache user-specific data in shared caches — include user ID in key
