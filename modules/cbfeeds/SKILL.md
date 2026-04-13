---
name: cbfeeds
description: >
  Use this skill when reading or generating RSS/Atom feeds in ColdBox/BoxLang using cbfeeds. Covers
  FeedReader for consuming external feeds, FeedGenerator for producing RSS 2.0 and Atom feeds,
  caching parsed feeds, and rendering feed XML from ColdBox handlers.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# CBFeeds Skill

## When to Use This Skill

Load this skill when:
- Parsing/consuming RSS or Atom feeds from external sources
- Generating RSS 2.0 or Atom feeds for your ColdBox application
- Caching parsed feed results to avoid repeated remote calls
- Exposing content as a syndication feed for readers or aggregators

## Installation

```bash
box install cbfeeds
```

## Reading Feeds

### Injection

```js
property name="feedReader" inject="FeedReader@cbfeeds";
```

### Parse an External Feed

```js
// Returns a struct with channel and items
var feed = feedReader.readFeed( "https://feeds.example.com/rss.xml" )

// Channel metadata
feed.channel.title
feed.channel.description
feed.channel.link
feed.channel.pubDate

// Items array
for ( var item in feed.items ) {
    item.title
    item.link
    item.description
    item.pubDate
    item.author
    item.enclosure    // podcast/media attachment
}
```

### Cache Parsed Feeds

```js
var cacheKey = "feed_" & hash( feedUrl )
var feed     = cache.getOrSet( cacheKey, function() {
    return feedReader.readFeed( feedUrl )
}, 30 )   // cache for 30 minutes
```

## Generating Feeds

### Injection

```js
property name="feedGenerator" inject="FeedGenerator@cbfeeds";
```

### Build an RSS 2.0 Feed

```js
var feed = feedGenerator.newFeed()
    .setTitle( "My Blog" )
    .setDescription( "Latest posts from My Blog" )
    .setLink( "https://www.example.com" )
    .setPubDate( now() )
    .setLanguage( "en-us" )

var posts = postService.getLatest( 20 )

for ( var post in posts ) {
    feed.addItem(
        title       = post.getTitle(),
        link        = "https://www.example.com/posts/" & post.getSlug(),
        description = post.getExcerpt(),
        pubDate     = post.getPublishedAt(),
        author      = post.getAuthor().getName(),
        guid        = "https://www.example.com/posts/" & post.getSlug()
    )
}
```

### Render Feed from a Handler

```js
// handlers/Feed.bx
@unsecured
class {

    property name="feedGenerator" inject="FeedGenerator@cbfeeds";

    function rss( event, rc, prc ) {
        var feed = feedGenerator.newFeed()
            .setTitle( "My Blog RSS" )
            .setDescription( "RSS feed for My Blog" )
            .setLink( "https://www.example.com" )
            .setPubDate( now() )

        var posts = postService.getPublished( 20 )

        for ( var post in posts ) {
            feed.addItem(
                title       = post.getTitle(),
                link        = "https://www.example.com/posts/" & post.getSlug(),
                description = post.getExcerpt(),
                pubDate     = post.getPublishedAt(),
                guid        = "https://www.example.com/posts/#post.getSlug()#"
            )
        }

        event.renderData(
            type        = "xml",
            data        = feed.render( "rss2" ),
            contentType = "application/rss+xml"
        )
    }

    function atom( event, rc, prc ) {
        // Same as above but render( "atom" )
        event.renderData(
            type        = "xml",
            data        = feed.render( "atom" ),
            contentType = "application/atom+xml"
        )
    }
}
```

### Register the Route

```js
// config/Router.bx
route( "/feed.rss" ).to( "Feed.rss" )
route( "/feed.atom" ).to( "Feed.atom" )
```

## Best Practices

- **Cache consumed feeds** — external feeds are slow and rate-limited; cache for at least 15-30 minutes
- **Handle HTTP errors** from `readFeed()` with try/catch — remote feeds are unreliable
- **Validate feed content** before exposing in your UI — externally sourced text should be encoded
- **Set a `<link rel="alternate">` tag** in your HTML `<head>` to advertise the feed URL to readers
- **Keep feed items to ≤50 entries** — larger feeds waste bandwidth for consumers
- **Use `guid` in each item** — ensures RSS readers correctly identify new vs. existing items
- **Set `Content-Type: application/rss+xml`** for RSS and `application/atom+xml` for Atom

## Documentation

- cbfeeds: https://github.com/coldbox-modules/cbfeeds
