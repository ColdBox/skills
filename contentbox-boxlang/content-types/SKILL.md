---
name: contentbox-boxlang-content-types
description: "Use this skill when implementing ContentBox content models and rendering flows, including entries/pages/content stores, custom fields, entity relationships, lifecycle callbacks, and content-type specific behaviors."
applyTo: "**/*.{bx,bxm,cfc,cfm,cfml}"
---

# ContentBox Content Types & Custom Fields (BoxLang)

Extend ContentBox content types and add custom fields using BoxLang. ContentBox supports entries, pages, and ContentStore items with extensible custom field capabilities.

## Content Types

ContentBox has three primary content types:

| Type | Entity | Service | Description |
|------|--------|---------|-------------|
| **Entries** | `Entry` | `entryService@contentbox` | Blog posts with dates, categories, comments |
| **Pages** | `Page` | `pageService@contentbox` | Static pages with hierarchical structure |
| **ContentStore** | `ContentStore` | `contentStoreService@contentbox` | Key-value content blocks |

### Entry Entity

```boxlang
property name="entryService" inject="entryService@contentbox"

// Entry properties
entry.getEntryID()
entry.getTitle()
entry.getSlug()
entry.getHTMLContent()
entry.getPlainTextContent()
entry.getPublishedDate()
entry.getCreatedDate()
entry.getModifiedDate()
entry.getIsActive()
entry.getIsPublished()
entry.getHits()
entry.getNumberOfComments()

// Relationships
entry.getAuthor()              // Author entity
entry.getCategories()          // Query of categories
entry.getComments()            // Query of comments
entry.getSite()                // Site entity
entry.getFeaturedImage()       // Media entity

// Status
entry.getStatus()              // "published", "draft", "pending"
entry.isPublished()
entry.isDraft()
```

### Page Entity

```boxlang
property name="pageService" inject="pageService@contentbox"

// Page properties
page.getPageID()
page.getTitle()
page.getSlug()
page.getHTMLContent()
page.getPlainTextContent()
page.getCreatedDate()
page.getModifiedDate()
page.getIsActive()
page.getIsPublished()
page.getMenuOrder()            // For ordering in menus
page.getParent()               // Parent page (hierarchical)
page.getChildren()             // Child pages

// Relationships
page.getAuthor()
page.getSite()
page.getFeaturedImage()
```

### ContentStore Entity

```boxlang
property name="contentStoreService" inject="contentStoreService@contentbox"

// ContentStore is a key-value store
contentStoreService.getValue( "key" )
contentStoreService.setValue( "key", "value" )
contentStoreService.deleteValue( "key" )
contentStoreService.getAll()

// ContentStore entity
item.getContentStoreID()
item.getSlug()                 // The key
item.getContent()              // The value
item.getCreatedDate()
item.getModifiedDate()
```

## Custom Fields

ContentBox supports custom fields through the `cbCustomField` entity and service:

```boxlang
property name="customFieldService" inject="customFieldService@contentbox"

// Create custom field
field = customFieldService.new( {
	name        : "subtitle",
	label       : "Subtitle",
	type        : "text",         // text, textarea, boolean, select, date, number
	required    : false,
	defaultValue: "",
	contentType : "entry",        // "entry", "page", or "all"
	order       : 1
} )
customFieldService.save( field )

// Get custom fields for content type
fields = customFieldService.findByContentType( "entry" )
```

### Custom Field Types

| Type | Description |
|------|-------------|
| `text` | Single-line text input |
| `textarea` | Multi-line text area |
| `boolean` | Checkbox toggle |
| `select` | Dropdown select |
| `date` | Date picker |
| `number` | Numeric input |
| `media` | Media picker |
| `html` | HTML editor |

### Accessing Custom Field Values

```boxlang
// Get custom field value from entry
subtitle = entry.getCustomFieldValue( "subtitle" )

// Get all custom field values
fields = entry.getCustomFields()
```

## Content Helpers (Mixins)

Inject mixins into all content objects via settings:

```boxlang
// In ModuleConfig.bx configure()
settings.contentHelpers = [
	"mymodule.models.mixins.MyContentMixin"
]
```

### Creating a Content Mixin

```boxlang
// models/mixins/MyContentMixin.bx
component {

	function getExcerpt( numeric maxLength = 200 ){
		return left( this.getPlainTextContent(), arguments.maxLength ) & "..."
	}

	function getReadingTime(){
		wordCount = arrayLen( listToArray( this.getPlainTextContent(), " " ) )
		return ceiling( wordCount / 200 )
	}

}
```

### Using Content Mixins

```boxlang
// After injecting, methods are available on all content objects
excerpt  = entry.getExcerpt( 150 )
readTime = entry.getReadingTime()
```

## Content Rendering

### Content Renderers

ContentBox uses interceptors to process content before rendering:

| Renderer | Class | Purpose |
|----------|-------|---------|
| **Link Renderer** | `LinkRenderer@contentbox` | Process internal links |
| **Widget Renderer** | `WidgetRenderer@contentbox` | Process `{widget:...}` shortcodes |
| **Setting Renderer** | `SettingRenderer@contentbox` | Process `{setting:...}` shortcodes |
| **Markdown Renderer** | `MarkdownRenderer@contentbox` | Process Markdown content |

### Widget Shortcode

```
{widget:WidgetName arg1="value" arg2="value"}
```

### Setting Shortcode

```
{setting:cb_site_title}
```

## Content Lifecycle

### Entry Lifecycle

```boxlang
// Interception points
cbadmin_preEntrySave      // Before entry saved
cbadmin_postEntrySave     // After entry saved
cbadmin_preEntryRemove    // Before entry deleted
cbadmin_postEntryRemove   // After entry deleted
cbadmin_onEntryStatusUpdate  // When status changes
cb_onContentRendering     // Before content rendered
```

### Page Lifecycle

```boxlang
// Interception points
cbadmin_prePageSave
cbadmin_postPageSave
cbadmin_prePageRemove
cbadmin_postPageRemove
cbadmin_onPageStatusUpdate
cb_onContentRendering
```

## Content Queries

### EntryService Queries

```boxlang
property name="entryService" inject="entryService@contentbox"

// Find published entries
entries = entryService.findPublishedContent(
	max       : 10,
	category  : "news",
	searchTerm: "search term",
	sortOrder : "publishedDate DESC",
	siteID    : siteId
)

// Find by slug
entry = entryService.findBySlug( "my-entry-slug" )

// Find by category
entries = entryService.findByCategory( "news" )

// Find by author
entries = entryService.findByAuthor( authorId )

// Count entries
count = entryService.countPublishedContent( siteID : siteId )
```

### PageService Queries

```boxlang
property name="pageService" inject="pageService@contentbox"

// Find all pages
pages = pageService.findAllWhere( { siteID : siteId, isPublished : true } )

// Find by slug
page = pageService.findBySlug( "about" )

// Find children
children = pageService.findChildren( parentId )

// Find hierarchical
tree = pageService.getHierarchicalPages( siteId )
```

## Best Practices

1. **Use services** — `entryService`, `pageService`, `contentStoreService`
2. **Use `findBySlug()`** — for human-readable URL lookups
3. **Always filter by `siteID`** — in multi-site installations
4. **Use content helpers** — for reusable content methods
5. **Use interception points** — for content lifecycle hooks
6. **Validate custom fields** — ensure data integrity
7. **Use `getPlainTextContent()`** — for excerpts and search
8. **Use `getHTMLContent()`** — for rendering
9. **Handle soft deletes** — check `isDeleted` flag
10. **Cache expensive queries** — use CacheBox for performance

## Engine Compatibility

This skill targets **BoxLang** engine. For CFML-specific syntax (Lucee 5+, Adobe ColdFusion 2018+), see the CFML variant of this skill.

Key BoxLang advantages:
- Cleaner script syntax without `<cfcomponent>` / `<cffunction>` tags
- No parentheses needed for zero-argument function calls
- `#{...}#` for inline expression output in `.bx` templates
- Modern syntax: `?:` null coalescing, `?.` safe navigation
- Native support for modern data structures
