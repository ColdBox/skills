---
name: contentbox-boxlang-multi-site
description: "Use this skill when implementing or maintaining ContentBox multi-site setups, including domain/site resolution, site-specific content and themes, settings isolation, menu behavior, and operational best practices."
applyTo: "**/*.{bx,bxm,cfc,cfm,cfml}"
---

# ContentBox Multi-Site Management (BoxLang)

Build and manage multi-site installations in ContentBox CMS using BoxLang. ContentBox supports running multiple sites from a single installation with shared or isolated content, themes, and settings.

## Multi-Site Architecture

### Site Entity

The `Site` entity (`models/system/Site.cfc`) represents a single site:

```boxlang
property name="siteService" inject="siteService@contentbox"

// Site properties
site.getSiteID()
site.getSlug()              // Unique site identifier
site.getName()
site.getDescription()
site.getDomain()            // Primary domain
site.getSiteURL()           // Full site URL
site.getIsActive()
site.getCreatedDate()
site.getModifiedDate()
```

### SiteService

```boxlang
property name="siteService" inject="siteService@contentbox"

// Site CRUD
siteService.getAll()                    // Get all sites
siteService.findBySlug( slug )          // Find site by slug
siteService.findByDomain( domain )      // Find site by domain
siteService.getCurrentSite()            // Get current site from request
siteService.save( site )                // Save site
siteService.delete( site )              // Delete site

// Site resolution
siteService.resolveSiteByDomain()       // Auto-resolve from request domain
siteService.setCurrentSite( site )      // Set current working site
```

## Site Resolution

ContentBox resolves the current site based on:

1. **Domain matching** — matches request domain to site's `domain` field
2. **Default site** — falls back to the default site if no match
3. **Admin context** — in admin, uses the currently selected working site

### Current Site in Handlers

```boxlang
// In any handler or service
property name="cb" inject="CBHelper@contentbox"

// Get current site
site = cb.site()
siteId = site.getSiteID()
siteSlug = site.getSlug()
siteURL = site.getSiteURL()
```

### Current Site in Widgets

Widgets use `getSite()` from `BaseWidget`:

```boxlang
function renderIt(){
	site = getSite()
	siteId = site.getSiteID()

	// Query content for this site
	entries = entryService.findPublishedContent( siteID : siteId )
}
```

## Site-Specific Settings

Each site can have its own settings overrides:

```boxlang
// In core ModuleConfig.bx
settings = {
	"global" : {
		// Global settings applied to all sites
	},
	"sites" : {
		// Site-specific overrides by slug
		"mysite" : {
			"cb_site_title" : "My Custom Site Title",
			"cb_site_tagline" : "My Tagline"
		},
		"othersite" : {
			"cb_site_title" : "Other Site Title"
		}
	}
}
```

### SettingService for Sites

```boxlang
property name="settingService" inject="settingService@contentbox"

// Get setting (resolves site-specific override)
title = settingService.getSetting( "cb_site_title" )

// Get setting for specific site
title = settingService.getSetting( "cb_site_title", siteId : siteId )
```

## Site-Specific Content

All content (entries, pages, categories, menus) is associated with a site:

```boxlang
// Create entry for specific site
entry = entryService.new( {
	title  : "My Entry",
	siteID : siteId
} )
entryService.save( entry )

// Query entries for specific site
entries = entryService.findPublishedContent(
	siteID    : siteId,
	max       : 10,
	sortOrder : "publishedDate DESC"
)

// Query pages for specific site
pages = pageService.findAllWhere( { siteID : siteId } )
```

## Site-Specific Themes

Each site can have its own active theme:

```boxlang
property name="themeService" inject="themeService@contentbox"

// Get active theme for current site
theme = themeService.getActiveTheme()

// Get theme for specific site
theme = themeService.getActiveTheme( siteId )

// Switch theme for a site
themeService.setActiveTheme( themeName, siteId )
```

## Site-Specific Menus

Menus are scoped to sites:

```boxlang
property name="menuService" inject="menuService@contentbox"

// Get menus for current site
menus = menuService.findAllWhere( { siteID : siteId } )

// Find menu by slug for specific site
menu = menuService.findBySlug( slug : "main", siteId : siteId )

// Render menu (automatically uses current site)
cb.menu( "main" )
```

## Creating a New Site

### Via Admin

1. Navigate to **Sites** in the admin
2. Click **Create Site**
3. Fill in: Name, Slug, Domain, Description
4. Configure site-specific settings
5. Activate the site

### Via Code

```boxlang
property name="siteService" inject="siteService@contentbox"

site = siteService.new( {
	name        : "My New Site",
	slug        : "mynewsite",
	domain      : "mynewsite.example.com",
	description : "A new site",
	isActive    : true
} )
siteService.save( site )
```

## Multi-Site Routing

ContentBox resolves routes based on the current site:

- **Domain-based routing** — each domain maps to a site
- **Shared routes** — routes are shared across sites, content is filtered by site
- **Site-specific URLs** — URLs include site context when needed

## Best Practices

1. **Always use `siteID`** in content queries for multi-site installations
2. **Use `cb.site()`** for getting the current site context
3. **Use `getSite()`** in widgets for site-aware rendering
4. **Scope menus and themes** to specific sites
5. **Use site-specific settings** for per-site configuration
6. **Test with multiple sites** — verify content isolation
7. **Use domain resolution** — configure domains for each site
8. **Handle missing sites** — gracefully handle unresolved sites
9. **Use `provider:` injection** for siteService to avoid circular deps
10. **Document site requirements** — note which features are site-scoped

## Engine Compatibility

This skill targets **BoxLang** engine. For CFML-specific syntax (Lucee 5+, Adobe ColdFusion 2018+), see the CFML variant of this skill.

Key BoxLang advantages:
- Cleaner script syntax without `<cfcomponent>` / `<cffunction>` tags
- No parentheses needed for zero-argument function calls
- `#{...}#` for inline expression output in `.bx` templates
- Modern syntax: `?:` null coalescing, `?.` safe navigation
- Native support for modern data structures
