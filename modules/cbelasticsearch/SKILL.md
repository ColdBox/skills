---
name: cbelasticsearch
description: >
  Use this skill when integrating Elasticsearch into a ColdBox/BoxLang application with the cbelasticsearch
  module. Covers installation, cluster configuration, document indexing (single and bulk), full-text search
  with the query DSL, aggregations, highlighting, scroll/pagination, index alias management, and production
  best practices.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# CBElasticsearch Skill

## When to Use This Skill

Load this skill when:
- Indexing ColdBox model/entity data into Elasticsearch
- Building full-text search with filters, ranges, and relevance tuning
- Running aggregations (faceted search, stats, terms)
- Performing bulk indexing or deletion operations
- Managing index mappings, aliases, and reindexing
- Setting up multi-cluster or environment-specific ES connections

## Language Mode Reference

| Concept | BoxLang (`.bx`) | CFML (`.cfc`) |
|---------|-----------------|---------------|
| Class declaration | `class {` | `component {` |
| DI annotation | `@inject` above property | `property name="x" inject="y";` |

## Installation & Configuration

```bash
box install cbelasticsearch
```

```js
// config/ColdBox.cfc
moduleSettings = {
    cbelasticsearch = {
        hosts = [
            {
                serverProtocol = "https",
                serverName     = getSystemSetting( "ES_HOST", "127.0.0.1" ),
                serverPort     = getSystemSetting( "ES_PORT", 9200 )
            }
        ],
        defaultIndex       = "myapp",
        defaultCredentials = {
            username = getSystemSetting( "ES_USER", "elastic" ),
            password = getSystemSetting( "ES_PASS", "" )
        }
    }
}
```

> Store ES credentials in environment variables — never hardcode passwords.

## Core API

### Injection

```js
property name="esClient" inject="Client@cbelasticsearch";
```

## Production Patterns

### Index a Single Document

```js
esClient
    .newDocument()
    .setIndex( "products" )
    .setId( product.getId() )
    .setMemento( product.getMemento() )
    .save()
```

### Bulk Index

```js
var documents = products.map( ( p ) => {
    return esClient
        .newDocument()
        .setIndex( "products" )
        .setId( p.getId() )
        .setMemento( p.getMemento() )
} )
esClient.saveAll( documents )
```

### Simple Match Search

```js
var results = esClient
    .newSearchBuilder()
    .setIndex( "products" )
    .match( "name", rc.q ?: "" )
    .execute()

prc.hits  = results.getHits()
prc.total = results.getHitCount()
```

### Advanced Search with Filters and Pagination

```js
var results = esClient
    .newSearchBuilder()
    .setIndex( "products" )
    .filterTerm( "category.keyword", "Electronics" )
    .filterRange( "price", { gte : 100, lte : 1000 } )
    .shouldMatch( "brand", "Apple" )
    .sort( "price", "asc" )
    .setFrom( ( rc.page - 1 ) * 20 )
    .setSize( 20 )
    .execute()
```

### Aggregations (Faceted Search)

```js
var results = esClient
    .newSearchBuilder()
    .setIndex( "products" )
    .setQuery( { "match_all" : {} } )
    .addAggregation( "by_category", {
        "terms" : { "field" : "category.keyword", "size" : 10 }
    } )
    .addAggregation( "price_stats", {
        "stats" : { "field" : "price" }
    } )
    .execute()

prc.categories = results.getAggregation( "by_category" )
prc.priceStats = results.getAggregation( "price_stats" )
```

### Result Highlighting

```js
var results = esClient
    .newSearchBuilder()
    .setIndex( "articles" )
    .match( "body", rc.q )
    .highlight( "body", {
        "pre_tags"  : [ "<mark>" ],
        "post_tags" : [ "</mark>" ]
    } )
    .execute()
```

### Delete a Document

```js
esClient
    .newDocument()
    .setIndex( "products" )
    .setId( productId )
    .delete()
```

### Index Alias (Zero-Downtime Reindex)

```js
// Create new index, populate, then atomically swap alias
esClient.createIndex( "products_v2", mapping )
// ... bulk index into products_v2 ...
esClient.applyAliases( [
    { remove : { index : "products_v1", alias : "products" } },
    { add    : { index : "products_v2", alias : "products" } }
] )
```

## Best Practices

- **Use aliases** — never point application code at a version-specific index name
- **Prefer bulk operations** for indexing > 10 documents; single-doc saves have higher overhead
- **Design mappings before indexing** — changing field types requires full reindex
- **Use `keyword` sub-field** for aggregations and exact filters on string fields: `"category.keyword"`
- **Store credentials in env vars** — never commit ES passwords to source control
- **Health-check on startup** — assert cluster is green before serving traffic
- **Use filters over queries** for non-scored operations — they are cached and faster

## Documentation

- cbelasticsearch: https://cbelasticsearch.ortusbooks.com
- Elasticsearch Query DSL: https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html
