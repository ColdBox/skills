---
name: contentbox-boxlang-module-development
description: "Use this skill when building ContentBox modules, including ModuleConfig conventions, admin integration, interceptors, routes, migrations, ORM entities, dependency registration, and module lifecycle hooks."
applyTo: "**/*.{bx,bxm,cfc,cfm,cfml}"
---

# ContentBox Module Development (BoxLang)

Build custom modules for ContentBox CMS using BoxLang. Modules are ColdBox modules that extend ContentBox functionality — adding new admin panels, API endpoints, widgets, interceptors, or content features.

## Module Architecture

ContentBox uses a layered module architecture:

| Layer | Path | Purpose |
|-------|------|---------|
| **Core** | `modules/contentbox/` | ContentBox engine (do not modify) |
| **Sub-modules** | `modules/contentbox/modules/` | Admin, API, UI, deps |
| **Custom** | `modules_app/contentbox-custom/` | User-owned customizations |
| **Standalone** | `modules/` | Independent ColdBox modules |

### Module Locations for Custom Code

| Type | Location |
|------|----------|
| Custom modules | `modules_app/contentbox-custom/_modules/` |
| Custom widgets | `modules_app/contentbox-custom/_widgets/` |
| Custom themes | `modules_app/contentbox-custom/_themes/` |
| Custom content | `modules_app/contentbox-custom/_content/` |
| Standalone modules | `modules/{moduleName}/` |

## ModuleConfig.bx

Every module requires a `ModuleConfig.bx` at its root:

```boxlang
// modules_app/contentbox-custom/_modules/myModule/ModuleConfig.bx

// Module Properties
this.title              = "My Module"
this.author             = "Your Name"
this.webURL             = "https://example.com"
this.version            = "1.0.0"
this.description        = "Description of my module"
this.viewParentLookup   = true
this.layoutParentLookup = true
this.entryPoint         = "mymodule"
this.modelNamespace     = "mymodule"
this.cfmapping          = "mymodule"
this.dependencies       = [ "contentbox" ]

function configure(){
	// Module Settings
	settings = {
		mySetting : "defaultValue"
	}

	// Layout Settings
	layoutSettings = {
		defaultLayout : "layout.bx"
	}

	// Custom Interception Points
	interceptorSettings = {
		customInterceptionPoints : [
			"onMyModuleEvent"
		]
	}

	// Interceptors
	interceptors = [
		{
			class : "mymodule.interceptors.MyInterceptor",
			name  : "MyInterceptor@mymodule"
		}
	]

	// Routes
	routes = [
		{ pattern : "/mymodule/:action", handler : "home" }
	]

	// i18n
	cbi18n = {
		resourceBundles : {
			"mymodule" : "#moduleMapping#/i18n/mymodule"
		}
	}
}

function onLoad(){
	// Post-load initialization
	// Register services, widgets, etc.
}

function onUnload(){
	// Cleanup on module unload
}
```

### ModuleConfig Properties

| Property | Description |
|----------|-------------|
| `title` | Module display name |
| `author` | Author name |
| `webURL` | Author/project website |
| `version` | Module version |
| `description` | Module description |
| `viewParentLookup` | Inherit views from parent modules (`true`/`false`) |
| `layoutParentLookup` | Inherit layouts from parent modules (`true`/`false`) |
| `entryPoint` | SES URL prefix for module routes |
| `modelNamespace` | WireBox DI namespace (e.g., `@mymodule`) |
| `cfmapping` | ColdFusion mapping for the module |
| `dependencies` | Array of required module names |

### ModuleConfig Methods

| Method | When Called | Purpose |
|--------|-------------|---------|
| `configure()` | Module registration | Define settings, routes, interceptors |
| `onLoad()` | After module loaded | Post-load initialization |
| `onUnload()` | Module unloaded | Cleanup resources |
| `onMissingMethod()` | Missing method calls | Dynamic method handling |

## Module Directory Structure

```
myModule/
├── ModuleConfig.bx        ← Module configuration
├── box.json               ← ForgeBox/CommandBox metadata
├── handlers/              ← ColdBox event handlers
│   └── Home.bx
├── models/                ← Business logic and entities
│   ├── services/
│   │   └── MyService.bx
│   └── entities/
│       └── MyEntity.bx
├── views/                 ← View templates
│   └── home/
│       └── index.bx
├── layouts/               ← Layout templates
│   └── layout.bx
├── interceptors/          ← ColdBox interceptors
│   └── MyInterceptor.bx
├── widgets/               ← ContentBox widgets
│   └── MyWidget.bx
├── modules/               ← Nested sub-modules
│   └── subModule/
├── config/                ← Configuration files
│   └── Router.bx
├── i18n/                  ← Resource bundles
│   └── mymodule.properties
├── migrations/            ← Database migrations
│   └── 001_create_my_table.bx
└── includes/              ← Static assets, help files
    ├── css/
    ├── js/
    └── help/
```

## Creating an Admin Module

To add functionality to the ContentBox admin:

```boxlang
// ModuleConfig.bx with admin integration

this.title          = "My Admin Module"
this.entryPoint     = "cbadmin/myModule"
this.modelNamespace = "myModule"
this.dependencies   = [ "contentbox", "contentbox-admin" ]

function configure(){
	settings = {}

	interceptorSettings = {
		customInterceptionPoints : []
	}

	routes = [
		{ pattern : "/cbadmin/myModule/:action", handler : "myModule" }
	]
}

function onLoad(){
	// Register with admin menu via interception
}
```

### Admin Handler Pattern

```boxlang
// handlers/MyModule.bx
component extends="contentbox.modules.contentbox-admin.handlers.baseHandler" singleton {

	function index(){
		prc.pageTitle = "My Module"
		setView( "myModule/index" )
	}

}
```

## Registering Admin Menu Items

Listen to `cbadmin_onAdminMenuLoad` to add menu items:

```boxlang
// interceptors/AdminMenu.bx
component {

	function configure(){
	}

	function onAdminMenuLoad( event, data, buffer, rc, prc ){
		// Add menu item to data.menuItems
		arrayAppend( data.menuItems, {
			name       : "My Module",
			link       : event.buildLink( "cbadmin/myModule" ),
			icon       : "star",
			permission : "MYMODULE_ACCESS"
		} )
	}

}
```

## Registering 2FA Providers

Register custom two-factor authentication providers in `onLoad()`:

```boxlang
function onLoad(){
	twoFactorService = wirebox.getInstance( "twoFactorService@contentbox" )
	twoFactorService.registerProvider(
		wirebox.getInstance( "MyTwoFactorProvider@myModule" )
	)
}
```

## Registering Content Helpers

Inject mixins into all content objects:

```boxlang
// In core ModuleConfig.bx configure()
settings.contentHelpers = [
	"myModule.models.mixins.MyContentMixin"
]
```

## Database Migrations

Use cfmigrations for schema changes:

```boxlang
// migrations/001_create_my_table.bx
function up( schema, qb ){
	schema.create( "my_table", function( table ){
		table.increments( "id" ).primary()
		table.string( "name" ).nullable( false )
		table.timestamps()
	} )
}

function down( schema, qb ){
	schema.drop( "my_table" )
}
```

Run migrations:
```bash
box run-script contentbox:migrate
box run-script contentbox:migrate:up
```

## ORM Entities

Create persistent entities with ContentBox conventions:

```boxlang
// models/entities/MyEntity.bx
component persistent="true" entityname="cbMyEntity" table="cb_my_table" extends="contentbox.models.BaseEntity" {

	property name="myEntityID" fieldtype="id" generator="uuid" ormtype="string"
	property name="name" ormtype="string" length="255" notnull="true"
	property name="description" ormtype="text"

}
```

## Module Dependencies

Declare dependencies in `ModuleConfig.bx`:

```boxlang
this.dependencies = [
	"contentbox",           // Core ContentBox
	"contentbox-admin",     // Admin module
	"contentbox-api",       // API module
	"cbsecurity",           // Security module
	"cborm"                 // ORM module
]
```

## Accessing ContentBox Services

```boxlang
// Inject ContentBox services
property name="entryService" inject="entryService@contentbox"
property name="pageService" inject="pageService@contentbox"
property name="authorService" inject="authorService@contentbox"
property name="categoryService" inject="categoryService@contentbox"
property name="menuService" inject="menuService@contentbox"
property name="settingService" inject="settingService@contentbox"
property name="securityService" inject="securityService@contentbox"
property name="widgetService" inject="widgetService@contentbox"
property name="themeService" inject="themeService@contentbox"
property name="cb" inject="CBHelper@contentbox"
```

## Interception Points

### Core ContentBox Interception Points

| Point | When Fired |
|-------|------------|
| `cb_onContentRendering` | Before content is rendered |
| `cb_onContentStoreRendering` | Before ContentStore content is rendered |

### Admin Interception Points

See the admin extension skill for the full list of `cbadmin_*` interception points.

## Best Practices

1. **Use `modules_app/contentbox-custom/_modules/`** for user-owned modules
2. **Declare dependencies** in `this.dependencies`
3. **Use namespace injection** — `@moduleName` for DI
4. **Follow ColdBox conventions** — handlers, models, views, layouts
5. **Use migrations** for database changes — never modify schema directly
6. **Extend base classes** — `BaseEntity`, `baseHandler`, `BaseWidget`
7. **Register interceptors** for admin menu, content lifecycle, etc.
8. **Use `provider:` injection** to avoid circular dependencies
9. **Include `box.json`** for ForgeBox distribution
10. **Test with all three engines** — Lucee, Adobe CF, BoxLang

## Engine Compatibility

This skill targets **BoxLang** engine. For CFML-specific syntax (Lucee 5+, Adobe ColdFusion 2018+), see the CFML variant of this skill.

Key BoxLang advantages:
- Cleaner script syntax without `<cfcomponent>` / `<cffunction>` tags
- No parentheses needed for zero-argument function calls
- `#{...}#` for inline expression output in `.bx` templates
- Modern syntax: `?:` null coalescing, `?.` safe navigation
- Native support for modern data structures
