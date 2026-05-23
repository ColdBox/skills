---
name: contentbox-boxlang-theme-development
description: "Use this skill when creating or customizing ContentBox themes, including theme structure, metadata/settings, layout and view composition, collection templates, widget overrides, and theme lifecycle callbacks."
applyTo: "**/*.{bx,bxm,cfc,cfm,cfml}"
---

# ContentBox Theme Development (BoxLang)

Build custom themes for ContentBox CMS using BoxLang. Themes control the visual presentation of all public-facing content — blog entries, pages, archives, search results, and error pages.

## Theme Structure

A ContentBox theme is a directory under `modules_app/contentbox-custom/_themes/` (custom) or `modules/contentbox/themes/` (core) containing:

```
MyTheme/
├── Theme.bx               ← Theme metadata, settings, lifecycle callbacks
├── screenshot.png         ← Theme preview image (shown in admin)
├── layouts/
│   ├── blog.bx            ← MANDATORY: Blog entry layout
│   └── pages.bx           ← MANDATORY: Page layout
│   ├── maintenance.bx     ← Optional: Maintenance mode layout
│   └── search.bx          ← Optional: Search results layout (defaults to pages)
├── views/
│   ├── index.bx           ← MANDATORY: Home page (blog entry listing)
│   ├── entry.bx           ← MANDATORY: Single blog entry with comments
│   ├── page.bx            ← MANDATORY: Single page rendering
│   ├── archives.bx        ← MANDATORY: Blog archives view
│   ├── error.bx           ← MANDATORY: Error display
│   ├── notfound.bx        ← Optional: Entry not found view
│   └── maintenance.bx     ← Optional: Maintenance mode view
├── templates/
│   ├── entry.bx           ← Collection template for entry iterations
│   ├── category.bx        ← Collection template for category iterations
│   └── comment.bx         ← Collection template for comment iterations
├── widgets/               ← Theme-specific widget overrides
│   └── MyWidget.bx        ← Overrides core widgets of the same name
└── includes/              ← Help files, assets, etc.
```

## Theme.bx

The `Theme.bx` defines metadata, settings, and lifecycle callbacks:

```boxlang
// Theme Metadata
this.name          = "My Custom Theme";
this.description   = "A beautiful custom theme for ContentBox";
this.version       = "1.0.0";
this.author        = "Your Name";
this.authorURL     = "https://example.com";
this.screenShotURL = "screenshot.png";

// Theme Settings — array of setting structs
this.settings = [
	{
		name         : "siteTitle",
		defaultValue : "My Site",
		type         : "text",
		label        : "Site Title:",
		required     : true,
		group        : "General"
	},
	{
		name         : "primaryColor",
		defaultValue : "#3b82f6",
		type         : "color",
		label        : "Primary Color:",
		group        : "Colors"
	},
	{
		name         : "showSidebar",
		defaultValue : true,
		type         : "boolean",
		label        : "Show Sidebar:",
		group        : "Layout"
	},
	{
		name         : "layoutStyle",
		defaultValue : "grid",
		type         : "select",
		label        : "Entry Layout:",
		options      : "grid,list,masonry",
		group        : "Layout"
	},
	{
		name         : "footerText",
		defaultValue : "",
		type         : "textarea",
		label        : "Footer Text:",
		group        : "General"
	}
];

/**
 * Called when the theme is activated
 */
function onActivation(){
	// Run setup logic, create default content, etc.
}

/**
 * Called when the theme is deactivated
 */
function onDeactivation(){
	// Cleanup logic
}

/**
 * Called when the theme is deleted
 */
function onDelete(){
	// Cleanup logic
}
```

### Setting Types

| Type | Description |
|------|-------------|
| `text` | Single-line text input (default) |
| `textarea` | Multi-line text area |
| `boolean` | Checkbox toggle |
| `select` | Dropdown select box |
| `color` | Color picker |

### Setting Struct Keys

| Key | Required | Description |
|-----|----------|-------------|
| `name` | Yes | Setting name (saved as `cb_themeName_settingName`) |
| `defaultValue` | Yes | Default value |
| `type` | No | HTML control type (default: `text`) |
| `label` | No | HTML label (defaults to `name`) |
| `required` | No | Whether the setting is required (default: `false`) |
| `title` | No | HTML title attribute |
| `options` | No | For `select`: comma-separated list or array of values, or array of `{name, value}` structs |
| `optionsUDF` | No | UDF name (no parentheses) that returns options, e.g., `getColors` |
| `group` | No | Group name for organizing settings |
| `groupIntro` | No | Description text for a group |
| `fieldDescription` | No | Description for an individual field |
| `fieldHelp` | No | HTML for a modal help popup (use `loadHelpFile()` helper) |

## Accessing Theme Settings in Views

Theme settings are available via the `cb` helper:

```boxlang
// Get a theme setting
cb.getThemeSetting( "siteTitle" )
cb.getThemeSetting( "primaryColor" )

// With fallback default
cb.getThemeSetting( "showSidebar", true )
```

## Layout Files

### blog.bx — Blog Entry Layout

```boxlang
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<title>#{cb.getContent().getTitle()}# — #{cb.getThemeSetting( "siteTitle" )}#</title>
	#{renderView( view = "_assets/head" )}#
</head>
<body>
	#{renderView( view = "_assets/header" )}#

	<main class="container">
		#{renderView()}#
	</main>

	#{renderView( view = "_assets/footer" )}#
</body>
</html>
```

### pages.bx — Page Layout

Similar structure to `blog.bx`, used for rendering static pages.

## View Files

### index.bx — Home Page

```boxlang
// Render entries using collection template
cb.renderCollection(
	template   : "entry",
	collection : prc.entries,
	counter    : prc.start,
	totalItems : prc.totalRecords
)

// Pagination
cb.paginator(
	totalRecords : prc.totalRecords,
	maxRows      : prc.maxRows,
	page         : prc.page,
	pageLink     : cb.siteURL() & "/page/{page}",
	align        : "center"
)
```

### entry.bx — Single Blog Entry

```boxlang
entry = cb.getContent()

<article class="entry">
	<header>
		<h1>#{entry.getTitle()}#</h1>
		<div class="meta">
			By #{entry.getAuthor().getFullName()}#
			on #{dateFormat( entry.getCreatedDate(), "mmmm d, yyyy" )}#
		</div>
	</header>

	<div class="content">
		#{entry.getHTMLContent()}#
	</div>

	// Categories
	if( entry.getCategories().recordCount ){
		writeOutput( '
			<div class="categories">
				#{cb.renderCollection(
					template   : "category",
					collection : entry.getCategories()
				)}#
			</div>
		' )
	}

	// Comments
	cb.widget( "CommentForm" )
</article>
```

### page.bx — Single Page

```boxlang
page = cb.getContent()

<article class="page">
	<h1>#{page.getTitle()}#</h1>
	<div class="content">
		#{page.getHTMLContent()}#
	</div>
</article>
```

### archives.bx — Archives View

```boxlang
<h1>Archives</h1>

// Monthly archives
<div class="archives-by-month">
	#{cb.renderCollection(
		template   : "entry",
		collection : prc.entries
	)}#
</div>

// Pagination
#{cb.paginator(
	totalRecords : prc.totalRecords,
	maxRows      : prc.maxRows,
	page         : prc.page
)}#
```

### error.bx — Error Display

```boxlang
<div class="error-page">
	<h1>Error</h1>
	<p>#{prc.errorMessage ?: "An unexpected error occurred."}#</p>
	<a href="#{cb.siteURL()}#" class="btn">Return Home</a>
</div>
```

## Collection Templates

Templates in `templates/` are used with `cb.renderCollection()`. Each template receives:

- `_counter` — Current iteration index (1-based)
- `_items` — Total number of items in the collection
- `{templateName}` — The object being rendered (e.g., `entry`, `category`, `comment`)

### templates/entry.bx

```boxlang
<article class="entry-preview">
	<h2>
		<a href="#{cb.entryURL( entry )}#">#{entry.getTitle()}#</a>
	</h2>
	<div class="meta">
		#{dateFormat( entry.getCreatedDate(), "mmmm d, yyyy" )}#
		by #{entry.getAuthor().getFullName()}#
	</div>
	<div class="excerpt">
		#{entry.getHTMLContentExcerpt()}#
	</div>
</article>
```

### templates/category.bx

```boxlang
<a href="#{cb.categoryURL( category )}#" class="category-tag">
	#{category.getCategory()}#
</a>
```

### templates/comment.bx

```boxlang
<div class="comment">
	<div class="comment-author">#{comment.getAuthor()}#</div>
	<div class="comment-date">#{dateFormat( comment.getCreatedDate(), "mmmm d, yyyy" )}#</div>
	<div class="comment-body">#{comment.getComment()}#</div>
</div>
```

## Widget Overrides

Place widgets in `widgets/` to override core widgets of the same name:

```boxlang
// widgets/Menu.bx — overrides the core Menu widget
component extends="contentbox.models.ui.BaseWidget" singleton {

	function init(){
		setName( "Menu" )
		setVersion( "1.0.0" )
		setDescription( "Custom menu widget override" )
	}

	any function renderIt( string menuName = "main" ){
		// Custom menu rendering
	}

}
```

## The CB Helper

The `cb` helper (`CBHelper@contentbox`) is the primary API for theme development:

```boxlang
// Site info
cb.site()                    // Current site entity
cb.siteURL()                 // Site base URL
cb.siteName()                // Site name

// Content
cb.getContent()              // Current content (entry/page)
cb.entryURL( entry )         // Entry permalink
cb.pageURL( page )           // Page URL
cb.categoryURL( category )   // Category URL

// Theme settings
cb.getThemeSetting( "name" )

// Widgets
cb.widget( "WidgetName", { arg1 : "value" } )

// Rendering
cb.renderCollection( template : "entry", collection : query )
cb.renderView( view = "partial" )

// Menus
cb.menu( "main" )

// RSS feeds
cb.rssURL()
cb.rssCommentsURL()

// Search
cb.searchURL()
cb.searchURL( "query" )

// Subscriptions
cb.subscribeURL()
cb.unsubscribeURL()
```

## Theme Discovery and Registration

ContentBox discovers themes from two locations:

1. **Core themes**: `modules/contentbox/themes/`
2. **Custom themes**: `modules_app/contentbox-custom/_themes/`

The `ThemeService@contentbox` builds the theme registry at startup. Custom themes override core themes of the same name.

## Theme Switching

Themes can be switched per-site via admin settings. The active theme is resolved at runtime:

```boxlang
// In a handler or service
property name="themeService" inject="themeService@contentbox"

activeTheme = themeService.getActiveTheme()
themePath   = themeService.getThemePath( activeTheme )
```

## Best Practices

1. **Always include mandatory files**: `Theme.bx`, `blog.bx`, `pages.bx`, `index.bx`, `entry.bx`, `page.bx`, `archives.bx`, `error.bx`
2. **Use `cb` helper** for all URL generation — never hardcode paths
3. **Use collection templates** for iterating over entries, categories, comments
4. **Group theme settings** logically using the `group` key
5. **Provide `screenshot.png`** for admin theme preview
6. **Use `loadHelpFile()`** for field help that reads from `includes/help/`
7. **Keep theme-specific widgets** in the theme's `widgets/` folder
8. **Test with multiple content types** — entries, pages, categories, search results
9. **Use `prc` scope** for handler-passed data in views
10. **Leverage BoxLang features** — null coalescing (`?:`), Elvis operator, modern syntax

## Engine Compatibility

This skill targets **BoxLang** engine. For CFML-specific syntax (Lucee 5+, Adobe ColdFusion 2018+), see the CFML variant of this skill.

Key BoxLang advantages:
- Use `#{...}#` for inline expression output in `.bx` templates
- No `<cfoutput>` wrapper needed
- Modern syntax: `?:` null coalescing, `?.` safe navigation
- Cleaner function calls without parentheses when no arguments
- Native support for modern data structures
