---
name: contentbox-boxlang-api-headless
description: "Use this skill when implementing headless ContentBox APIs, including REST endpoint design, JWT authentication flows, content CRUD, custom API handlers, and integration patterns for decoupled frontends."
applyTo: "**/*.{bx,bxm,cfc,cfm,cfml}"
---

# ContentBox API & Headless Development (BoxLang)

Build headless and REST API integrations with ContentBox CMS using BoxLang. ContentBox provides a full REST API (v1) for managing all content types, enabling headless CMS architectures.

## API Architecture

The API module lives at `modules/contentbox/modules/contentbox-api/` with a nested v1 module at `modules/contentbox/modules/contentbox-api/modules/contentbox-api-v1/`.

### API v1 Handlers

| Handler | Resource | Description |
|---------|----------|-------------|
| `auth.bx` | Authentication | JWT token generation and validation |
| `authors.bx` | Authors | Author CRUD operations |
| `categories.bx` | Categories | Category CRUD operations |
| `comments.bx` | Comments | Comment management |
| `contentStore.bx` | ContentStore | Key-value content blocks |
| `contentTemplates.bx` | Templates | Content template management |
| `entries.bx` | Entries | Blog entry CRUD operations |
| `menus.bx` | Menus | Menu management |
| `pages.bx` | Pages | Page CRUD operations |
| `relocations.bx` | Relocations | URL redirect management |
| `settings.bx` | Settings | Global settings API |
| `siteSettings.bx` | Site Settings | Site-specific settings |
| `sites.bx` | Sites | Multi-site management |
| `versions.bx` | Versions | Content versioning |
| `echo.bx` | Health Check | API health check / echo |

### Base Handler Pattern

API handlers extend `BaseHandler` (which extends `cborm.models.resources.BaseHandler`):

```boxlang
// handlers/api/v1/MyResource.bx
component extends="contentbox.modules.contentbox-api.modules.contentbox-api-v1.handlers.baseHandler" singleton {

	// Inject the virtual entity service
	property name="ormService" inject="MyEntityService@contentbox"

	// Entity name (singular)
	variables.entity = "MyEntity"

	// Default sort order
	variables.sortOrder = "createdDate DESC"

	// Use native getOrFail() or getByIdOrSlugOrFail()
	variables.useGetOrFail = true

}
```

This automatically provides: `index`, `create`, `show`, `update`, `delete` methods.

## Authentication

### JWT Authentication

The API uses JWT tokens for authentication:

```
POST /api/v1/auth
Body: { "username": "admin", "password": "secret" }

Response: { "token": "eyJhbGciOiJIUzI1NiIs...", "expires": 3600 }
```

### Using Tokens

Include the token in the `Authorization` header:

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

## API Endpoints

### Entries

```
GET    /api/v1/entries              → List entries (paginated)
GET    /api/v1/entries/:id          → Get single entry
POST   /api/v1/entries              → Create entry
PUT    /api/v1/entries/:id          → Update entry
DELETE /api/v1/entries/:id          → Delete entry
```

#### Query Parameters

| Parameter | Description |
|-----------|-------------|
| `page` | Page number (default: 1) |
| `maxRows` | Results per page |
| `sortOrder` | Sort field and direction |
| `isDeleted` | Include soft-deleted entries |
| `includes` | Related entities to include |
| `excludes` | Fields to exclude from response |

### Pages

```
GET    /api/v1/pages                → List pages
GET    /api/v1/pages/:id            → Get single page (by ID or slug)
POST   /api/v1/pages                → Create page
PUT    /api/v1/pages/:id            → Update page
DELETE /api/v1/pages/:id            → Delete page
```

### Categories

```
GET    /api/v1/categories           → List categories
GET    /api/v1/categories/:id       → Get single category
POST   /api/v1/categories           → Create category
PUT    /api/v1/categories/:id       → Update category
DELETE /api/v1/categories/:id       → Delete category
```

### Authors

```
GET    /api/v1/authors              → List authors
GET    /api/v1/authors/:id          → Get single author
POST   /api/v1/authors              → Create author
PUT    /api/v1/authors/:id          → Update author
DELETE /api/v1/authors/:id          → Delete author
```

### ContentStore

```
GET    /api/v1/contentstore         → List all content store items
GET    /api/v1/contentstore/:key    → Get item by key
POST   /api/v1/contentstore         → Create item
PUT    /api/v1/contentstore/:key    → Update item
DELETE /api/v1/contentstore/:key    → Delete item
```

### Menus

```
GET    /api/v1/menus                → List menus
GET    /api/v1/menus/:slug          → Get menu by slug
POST   /api/v1/menus                → Create menu
PUT    /api/v1/menus/:slug          → Update menu
DELETE /api/v1/menus/:slug          → Delete menu
```

### Sites

```
GET    /api/v1/sites                → List sites
GET    /api/v1/sites/:id            → Get single site
POST   /api/v1/sites                → Create site
PUT    /api/v1/sites/:id            → Update site
DELETE /api/v1/sites/:id            → Delete site
```

## Response Format

### List Response

```json
{
	"data": [
		{ "id": "...", "title": "...", "slug": "...", ... }
	],
	"total": 100,
	"page": 1,
	"maxRows": 25
}
```

### Single Resource Response

```json
{
	"data": {
		"id": "...",
		"title": "...",
		"slug": "...",
		"content": "...",
		"author": { ... },
		"categories": [ ... ],
		...
	}
}
```

### Error Response

```json
{
	"error": true,
	"message": "Resource not found",
	"details": "..."
}
```

## Creating Custom API Endpoints

### Custom API Handler

```boxlang
// handlers/api/v1/CustomResource.bx
component extends="contentbox.modules.contentbox-api.modules.contentbox-api-v1.handlers.baseHandler" singleton {

	property name="ormService" inject="CustomEntityService@contentbox"
	variables.entity = "CustomEntity"
	variables.sortOrder = "createdDate DESC"

	// Override index for custom filtering
	function index( event, rc, prc, criteria, results ){
		// Custom filtering logic
		prc.criteria = ormService.newCriteria()
		if( structKeyExists( rc, "status" ) ){
			prc.criteria.isEq( "status", rc.status )
		}

		// Delegate to parent
		super.index( event, rc, prc, prc.criteria )
	}

	// Add custom action
	function publish( event, rc, prc ){
		entity = ormService.get( rc.id )
		entity.setStatus( "published" )
		ormService.save( entity )

		renderData(
			type       : "json",
			data       : { success : true, entity : entity.getMemento() },
			statusCode : 200
		)
	}

}
```

### Custom API Routes

Register routes in your module's `ModuleConfig.bx`:

```boxlang
routes = [
	// RESTful resources
	{ pattern : "/api/v1/custom", handler : "api/v1/customResource" },
	{ pattern : "/api/v1/custom/:id", handler : "api/v1/customResource" },
	// Custom actions
	{ pattern : "/api/v1/custom/:id/publish", handler : "api/v1/customResource", action : "publish" }
]
```

## Headless CMS Usage

### Frontend Integration

Use the API to build headless frontends:

```javascript
// Fetch entries
const response = await fetch('/api/v1/entries?page=1&maxRows=10', {
  headers: { 'Authorization': `Bearer ${token}` }
});
const { data, total, page } = await response.json();

// Fetch single page by slug
const pageResponse = await fetch('/api/v1/pages/my-page-slug', {
  headers: { 'Authorization': `Bearer ${token}` }
});
const { data: page } = await pageResponse.json();
```

### Content Rendering

The API returns content with all fields, including:

- `title`, `slug`, `content` (HTML)
- `publishedDate`, `createdDate`, `modifiedDate`
- `author` (nested object)
- `categories` (array)
- `customFields` (if configured)
- `featuredImage` (media reference)

## API Security

### Firewall Rules

API routes are protected by cbSecurity rules. Configure in settings:

```boxlang
settings.cbsecurity = {
	firewall : {
		invalidAuthenticationEvent : "cbapi/auth/unauthorized",
		defaultAuthenticationAction : "redirect",
		invalidAuthorizationEvent : "cbapi/auth/forbidden",
		defaultAuthorizationAction : "redirect"
	}
}
```

### Rate Limiting

The core `RateLimiter@contentbox` interceptor protects against brute-force attacks.

## Best Practices

1. **Extend `baseHandler`** — get CRUD operations for free
2. **Inject `ormService`** — use the correct virtual entity service
3. **Use `variables.entity`** — set the singular entity name
4. **Set `variables.sortOrder`** — define default sorting
5. **Use `param` for defaults** — set safe defaults for query parameters
6. **Override methods as needed** — customize `index`, `show`, etc.
7. **Use `renderData()`** — for consistent JSON responses
8. **Announce interception points** — for extensibility
9. **Include related entities** — use `includes` parameter for nested data
10. **Handle errors gracefully** — return proper error responses

## Engine Compatibility

This skill targets **BoxLang** engine. For CFML-specific syntax (Lucee 5+, Adobe ColdFusion 2018+), see the CFML variant of this skill.

Key BoxLang advantages:
- Cleaner script syntax without `<cfcomponent>` / `<cffunction>` tags
- No parentheses needed for zero-argument function calls
- `#{...}#` for inline expression output in `.bx` templates
- Modern syntax: `?:` null coalescing, `?.` safe navigation
- Native support for modern data structures
