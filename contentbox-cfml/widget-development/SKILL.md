---
name: contentbox-cfml-widget-development
description: "Use this skill when building ContentBox widgets, including BaseWidget patterns, render logic, dependency injection, editor metadata annotations, caching choices, and safe output/rendering practices."
applyTo: "**/*.{cfc,cfm,cfml}"
---

# ContentBox Widget Development (CFML)

Build custom widgets for ContentBox CMS using CFML. Widgets are reusable, self-contained components that render dynamic content anywhere in pages, entries, sidebars, or layouts.

## Widget Architecture

Widgets are singleton CFCs that extend `contentbox.models.ui.BaseWidget`. They are discovered automatically from four locations (in priority order):

1. **Active theme widgets**: `themes/{activeTheme}/widgets/`
2. **Custom widgets**: `modules_app/contentbox-custom/_widgets/`
3. **Core widgets**: `modules/contentbox/widgets/`
4. **Module widgets**: Any registered module's `widgets/` folder

Theme widgets override core/custom widgets of the same name.

## BaseWidget Properties

All widgets inherit from `BaseWidget` which provides:

### Widget Metadata Properties

| Property | Type | Description |
|----------|------|-------------|
| `name` | string | Widget display name |
| `version` | string | Widget version |
| `description` | string | Widget description |
| `author` | string | Author name |
| `authorURL` | string | Author website |
| `forgeBoxSlug` | string | ForgeBox package slug |
| `category` | string | Widget category (e.g., "Content", "Blog", "Navigation") |
| `icon` | string | Icon name (used in admin UI) |

### Auto-Injected Services

Every widget receives these services via WireBox DI:

| Property | DSL | Description |
|----------|-----|-------------|
| `siteService` | `siteService@contentbox` | Multi-site management |
| `categoryService` | `categoryService@contentbox` | Category CRUD |
| `entryService` | `entryService@contentbox` | Blog entry CRUD |
| `pageService` | `pageService@contentbox` | Page CRUD |
| `contentService` | `contentService@contentbox` | Unified content service |
| `contentVersionService` | `contentVersionService@contentbox` | Content versioning |
| `authorService` | `authorService@contentbox` | Author management |
| `commentService` | `commentService@contentbox` | Comment management |
| `contentStoreService` | `contentStoreService@contentbox` | ContentStore (key-value) |
| `menuService` | `menuService@contentbox` | Menu management |
| `cb` | `CBHelper@contentbox` | CBHelper for theme/UI operations |
| `securityService` | `securityService@contentbox` | Authentication/authorization |
| `html` | `HTMLHelper@coldbox` | HTML helper utilities |
| `controller` | `coldbox` | ColdBox controller |
| `log` | `logbox:logger:{this}` | Logger instance |

## Creating a Widget

### Basic Widget

```cfml
<!--- modules_app/contentbox-custom/_widgets/HelloWorld.cfc --->
<cfcomponent extends="contentbox.models.ui.BaseWidget" singleton>

	<cffunction name="init" access="public" returntype="HelloWorld" output="false">
		<cfset setName( "Hello World" )>
		<cfset setVersion( "1.0.0" )>
		<cfset setDescription( "Displays a hello world message" )>
		<cfset setAuthor( "Your Name" )>
		<cfset setAuthorURL( "https://example.com" )>
		<cfset setIcon( "smile" )>
		<cfset setCategory( "General" )>
		<cfreturn this>
	</cffunction>

	<cffunction name="renderIt" access="public" returntype="any" output="false">
		<cfargument name="greeting" type="string" default="Hello" hint="The greeting text">
		<cfargument name="name" type="string" default="World" hint="The name to greet">

		<cfreturn "<p class='hello'>#arguments.greeting#, #arguments.name#!</p>">
	</cffunction>

</cfcomponent>
```

### Script Syntax Widget

```cfml
<!--- modules_app/contentbox-custom/_widgets/RecentEntries.cfc --->
component extends="contentbox.models.ui.BaseWidget" singleton {

	RecentEntries function init(){
		setName( "Recent Entries" );
		setVersion( "1.0" );
		setDescription( "Shows the most recent blog entries" );
		setAuthor( "Your Name" );
		setIcon( "list" );
		setCategory( "Blog" );
		return this;
	}

	/**
	 * Show recent blog entries
	 *
	 * @max         The number of entries to show (default: 5)
	 * @max.options 3,5,10,15,20
	 * @title       Optional title displayed as heading
	 * @titleLevel  Heading level (h1-h6), default: h2
	 * @category    Filter by category slug
	 * @category.multiOptionsUDF getAllCategories
	 * @sortOrder   Sort order for entries
	 * @sortOrder.options Most Recent,Most Popular,Most Commented
	 */
	any function renderIt(
		numeric max       = 5,
		title             = "",
		string titleLevel = "2",
		string category   = "",
		string sortOrder  = "Most Recent"
	){
		// Determine sort order
		switch( arguments.sortOrder ){
			case "Most Popular":
				arguments.sortOrder = "hits DESC";
				break;
			case "Most Commented":
				arguments.sortOrder = "numberOfComments DESC";
				break;
			default:
				arguments.sortOrder = "publishedDate DESC";
		}

		// Fetch entries
		var entries = entryService.findPublishedContent(
			max       : arguments.max,
			category  : arguments.category,
			sortOrder : arguments.sortOrder,
			siteID    : getSite().getSiteID()
		);

		// Build output
		var output = "";
		saveContent variable="output" {
			if( len( arguments.title ) ){
				writeOutput( "<h#arguments.titleLevel#>#arguments.title#</h#arguments.titleLevel#>" );
			}
			writeOutput( "<ul class='recent-entries'>" );
			for( var i = 1; i <= entries.recordCount; i++ ){
				var entry = entries;
				entries.absoluteRow = i;
				writeOutput( "<li>" );
				writeOutput( "<a href='#cb.entryURL( entry )#'>#entry.getTitle()#</a>" );
				writeOutput( " <small>#dateFormat( entry.getPublishedDate(), 'mmm d, yyyy' )#</small>" );
				writeOutput( "</li>" );
			}
			writeOutput( "</ul>" );
		}

		return output;
	}

	/**
	 * Get all categories for the select box
	 * @cbignore
	 */
	array function getAllCategories(){
		return categoryService.getAllSlugs();
	}

}
```

## renderIt() Method

The `renderIt()` method is the entry point for every widget. It:

- Must be implemented by every concrete widget
- Can accept any number of arguments
- Arguments become configurable in the admin widget editor
- Must return a string (the rendered HTML)

### Argument Annotations for Admin UI

Use metadata in function comments to control how arguments appear in the admin widget editor:

| Annotation | Description | Example |
|------------|-------------|---------|
| `@hint` | Help text for the argument | `@hint The menu slug` |
| `@label` | Custom label in the editor | `@label Search Term` |
| `@options` | Comma-separated select options | `@options red,blue,green` |
| `@optionsUDF` | UDF name that returns options | `@optionsUDF getCategories` |
| `@multiOptionsUDF` | UDF for multi-select options | `@multiOptionsUDF getAllCategories` |
| `@defaultValue` | Default value in editor | `@defaultValue 5` |

### @cbignore Annotation

Mark helper methods with `@cbignore` to exclude them from the widget editor:

```cfml
/**
 * Internal helper — not shown in widget editor
 * @cbignore
 */
array function getInternalList(){
	return [ "a", "b", "c" ];
}
```

## Using getSite()

The `getSite()` method (inherited from `BaseWidget`) detects the current context:

```cfml
function renderIt(){
	// In admin: returns the current working site being edited
	// In UI: returns the site for the current request
	var site = getSite();
	var siteId = site.getSiteID();

	// Use siteId for content queries
	var entries = entryService.findPublishedContent( siteID : siteId );
}
```

## Rendering Widgets

### In Content (via Admin Editor)

Widgets are inserted into entry/page content using the widget shortcode:

```
{widget:WidgetName arg1="value" arg2="value"}
```

The `WidgetRenderer@contentbox` interceptor processes these at render time.

### In Theme Views

```cfml
<!--- Render widget with arguments --->
#cb.widget( "RecentEntries", { max = 10, title = "Latest Posts" } )#

<!--- Render widget without arguments --->
#cb.widget( "SearchForm" )#

<!--- Render widget with dynamic arguments --->
#cb.widget( "Menu", { slug = "footer" } )#
```

### In Handlers

```cfml
property name="widgetService" inject="widgetService@contentbox";

// Get widget instance and render
var widget = widgetService.getWidget( "RecentEntries" );
var html   = widget.renderIt( max = 5, title = "Recent" );
```

## Widget Service API

```cfml
property name="widgetService" inject="widgetService@contentbox";

// Get all widgets as a query
var widgets = widgetService.getWidgets();

// Get widget names as an array
var names = widgetService.getWidgetsList();

// Get distinct widget categories
var categories = widgetService.getWidgetCategories();

// Get a specific widget instance
var widget = widgetService.getWidget( "Menu" );

// Force reload widget registry
widgetService.getWidgets( reload = true );
```

## Widget Locations

| Location | Path | Override Priority |
|----------|------|-------------------|
| Active theme | `themes/{theme}/widgets/` | Highest (overrides all) |
| Custom | `modules_app/contentbox-custom/_widgets/` | 2nd |
| Core | `modules/contentbox/widgets/` | 3rd |
| Module | `{module}/widgets/` | Lowest |

## Core Widgets Reference

ContentBox ships with these built-in widgets:

| Widget | Category | Description |
|--------|----------|-------------|
| `Archives` | Blog | Monthly archive links |
| `Categories` | Blog | Category listing |
| `CommentForm` | Blog | Comment submission form |
| `ContentStore` | Content | Key-value content blocks |
| `EntryInclude` | Content | Include entry content by slug |
| `Menu` | Navigation | Render ContentBox menus |
| `Meta` | SEO | Meta tag generation |
| `PageInclude` | Content | Include page content by slug |
| `RSS` | Blog | RSS feed links |
| `RecentComments` | Blog | Recent comments listing |
| `RecentEntries` | Blog | Recent blog entries |
| `RecentPages` | Pages | Recent pages listing |
| `RelatedContent` | Content | Related entries/pages |
| `Relocate` | Utility | Page redirect widget |
| `Renderview` | Utility | Render a ColdBox view |
| `SearchForm` | Search | Search input form |
| `SubPageMenu` | Navigation | Sub-page navigation menu |
| `Viewlet` | Utility | Execute a ColdBox event |
| `cb` | Utility | Generic CBHelper access |

## Best Practices

1. **Always extend `BaseWidget`** — provides DI and utility methods
2. **Use `singleton` scope** — widgets are instantiated once and cached
3. **Implement `renderIt()`** — the only required method
4. **Use `getSite()`** — for multi-site aware content queries
5. **Annotate arguments** — for rich admin editor experience
6. **Use `@cbignore`** — for internal helper methods
7. **Return strings** — `renderIt()` must return rendered HTML
8. **Use `saveContent`** — for building complex output
9. **Leverage injected services** — no need to manually wire dependencies
10. **Categorize widgets** — set `category` for admin organization
11. **Provide icons** — use icon names for admin UI display
12. **Test in both contexts** — admin editor and public UI rendering

## Engine Compatibility

This skill targets **CFML engines** (Lucee 5+, Adobe ColdFusion 2018+). For BoxLang-specific syntax and features, see the BoxLang variant of this skill.

Key CFML considerations:
- Use `<cfcomponent>` tag syntax or script syntax with `component` keyword
- Use `<cffunction>` for tag-based or `function` for script-based
- Use `saveContent` for building output strings
- Use `structKeyExists()` for safe struct access
- Use `isDefined()` or `isNull()` for null checks
