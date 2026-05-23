---
name: contentbox-cfml-module-development
description: "Use this skill when building ContentBox modules, including ModuleConfig conventions, admin integration, interceptors, routes, migrations, ORM entities, dependency registration, and module lifecycle hooks."
applyTo: "**/*.{cfc,cfm,cfml}"
---

# ContentBox Module Development (CFML)

Build custom modules for ContentBox CMS using CFML. Modules are ColdBox modules that extend ContentBox functionality — adding new admin panels, API endpoints, widgets, interceptors, or content features.

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

## ModuleConfig.cfc

Every module requires a `ModuleConfig.cfc` at its root:

```cfml
<!--- modules_app/contentbox-custom/_modules/myModule/ModuleConfig.cfc --->
<cfcomponent>

	<!--- Module Properties --->
	<cfset this.title              = "My Module">
	<cfset this.author             = "Your Name">
	<cfset this.webURL             = "https://example.com">
	<cfset this.version            = "1.0.0">
	<cfset this.description        = "Description of my module">
	<cfset this.viewParentLookup   = true>
	<cfset this.layoutParentLookup = true>
	<cfset this.entryPoint         = "mymodule">
	<cfset this.modelNamespace     = "mymodule">
	<cfset this.cfmapping          = "mymodule">
	<cfset this.dependencies       = [ "contentbox" ]>

	<cffunction name="configure" access="public" returntype="void">
		<!--- Module Settings --->
		<cfset settings = {
			mySetting : "defaultValue"
		}>

		<!--- Layout Settings --->
		<cfset layoutSettings = {
			defaultLayout : "layout.cfm"
		}>

		<!--- Custom Interception Points --->
		<cfset interceptorSettings = {
			customInterceptionPoints : [
				"onMyModuleEvent"
			]
		}>

		<!--- Interceptors --->
		<cfset interceptors = [
			{
				class : "mymodule.interceptors.MyInterceptor",
				name  : "MyInterceptor@mymodule"
			}
		]>

		<!--- Routes --->
		<cfset routes = [
			{ pattern = "/mymodule/:action", handler = "home" }
		]>

		<!--- i18n --->
		<cfset cbi18n = {
			resourceBundles : {
				"mymodule" : "#moduleMapping#/i18n/mymodule"
			}
		}>
	</cffunction>

	<cffunction name="onLoad" access="public" returntype="void">
		<!--- Post-load initialization --->
		<!--- Register services, widgets, etc. --->
	</cffunction>

	<cffunction name="onUnload" access="public" returntype="void">
		<!--- Cleanup on module unload --->
	</cffunction>

</cfcomponent>
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
├── ModuleConfig.cfc       ← Module configuration
├── box.json               ← ForgeBox/CommandBox metadata
├── handlers/              ← ColdBox event handlers
│   └── Home.cfc
├── models/                ← Business logic and entities
│   ├── services/
│   │   └── MyService.cfc
│   └── entities/
│       └── MyEntity.cfc
├── views/                 ← View templates
│   └── home/
│       └── index.cfm
├── layouts/               ← Layout templates
│   └── layout.cfm
├── interceptors/          ← ColdBox interceptors
│   └── MyInterceptor.cfc
├── widgets/               ← ContentBox widgets
│   └── MyWidget.cfc
├── modules/               ← Nested sub-modules
│   └── subModule/
├── config/                ← Configuration files
│   └── Router.cfc
├── i18n/                  ← Resource bundles
│   └── mymodule.properties
├── migrations/            ← Database migrations
│   └── 001_create_my_table.cfm
└── includes/              ← Static assets, help files
    ├── css/
    ├── js/
    └── help/
```

## Creating an Admin Module

To add functionality to the ContentBox admin:

```cfml
<!--- ModuleConfig.cfc with admin integration --->
<cfcomponent>

	<cfset this.title          = "My Admin Module">
	<cfset this.entryPoint     = "cbadmin/myModule">
	<cfset this.modelNamespace = "myModule">
	<cfset this.dependencies   = [ "contentbox", "contentbox-admin" ]>

	<cffunction name="configure">
		<cfset settings = {}>

		<cfset interceptorSettings = {
			customInterceptionPoints : []
		}>

		<cfset routes = [
			{ pattern = "/cbadmin/myModule/:action", handler = "myModule" }
		]>
	</cffunction>

	<cffunction name="onLoad">
		<!--- Register with admin menu via interception --->
	</cffunction>

</cfcomponent>
```

### Admin Handler Pattern

```cfml
<!--- handlers/MyModule.cfc --->
<cfcomponent extends="contentbox.modules.contentbox-admin.handlers.baseHandler" singleton>

	<cffunction name="index" access="public" returntype="void">
		<cfset prc.pageTitle = "My Module">
		<cfset setView( "myModule/index" )>
	</cffunction>

</cfcomponent>
```

## Registering Admin Menu Items

Listen to `cbadmin_onAdminMenuLoad` to add menu items:

```cfml
<!--- interceptors/AdminMenu.cfc --->
<cfcomponent>

	<cffunction name="configure">
	</cffunction>

	<cffunction name="onAdminMenuLoad" access="public" returntype="void">
		<cfargument name="event" type="any">
		<cfargument name="data" type="struct">
		<cfargument name="buffer" type="any">
		<cfargument name="rc" type="struct">
		<cfargument name="prc" type="struct">

		<!--- Add menu item to data.menuItems --->
		<cfset arrayAppend( data.menuItems, {
			name     : "My Module",
			link     : event.buildLink( "cbadmin/myModule" ),
			icon     : "star",
			permission : "MYMODULE_ACCESS"
		} )>
	</cffunction>

</cfcomponent>
```

## Registering 2FA Providers

Register custom two-factor authentication providers in `onLoad()`:

```cfml
<cffunction name="onLoad">
	<cfset var twoFactorService = wirebox.getInstance( "twoFactorService@contentbox" )>
	<cfset twoFactorService.registerProvider(
		wirebox.getInstance( "MyTwoFactorProvider@myModule" )
	)>
</cffunction>
```

## Registering Content Helpers

Inject mixins into all content objects:

```cfml
<!--- In core ModuleConfig.cfc configure() --->
<cfset settings.contentHelpers = [
	"myModule.models.mixins.MyContentMixin"
]>
```

## Database Migrations

Use cfmigrations for schema changes:

```cfml
<!--- migrations/001_create_my_table.cfm --->
<cfscript>
	function up( schema, qb ){
		schema.create( "my_table", function( table ){
			table.increments( "id" ).primary();
			table.string( "name" ).nullable( false );
			table.timestamps();
		} );
	}

	function down( schema, qb ){
		schema.drop( "my_table" );
	}
</cfscript>
```

Run migrations:
```bash
box run-script contentbox:migrate
box run-script contentbox:migrate:up
```

## ORM Entities

Create persistent entities with ContentBox conventions:

```cfml
<!--- models/entities/MyEntity.cfc --->
<cfcomponent persistent="true" entityname="cbMyEntity" table="cb_my_table" extends="contentbox.models.BaseEntity">

	<cfproperty name="myEntityID" fieldtype="id" generator="uuid" ormtype="string">
	<cfproperty name="name" ormtype="string" length="255" notnull="true">
	<cfproperty name="description" ormtype="text">

</cfcomponent>
```

## Module Dependencies

Declare dependencies in `ModuleConfig.cfc`:

```cfml
<cfset this.dependencies = [
	"contentbox",           // Core ContentBox
	"contentbox-admin",     // Admin module
	"contentbox-api",       // API module
	"cbsecurity",           // Security module
	"cborm"                 // ORM module
]>
```

## Accessing ContentBox Services

```cfml
<!--- Inject ContentBox services --->
<cfproperty name="entryService" inject="entryService@contentbox">
<cfproperty name="pageService" inject="pageService@contentbox">
<cfproperty name="authorService" inject="authorService@contentbox">
<cfproperty name="categoryService" inject="categoryService@contentbox">
<cfproperty name="menuService" inject="menuService@contentbox">
<cfproperty name="settingService" inject="settingService@contentbox">
<cfproperty name="securityService" inject="securityService@contentbox">
<cfproperty name="widgetService" inject="widgetService@contentbox">
<cfproperty name="themeService" inject="themeService@contentbox">
<cfproperty name="cb" inject="CBHelper@contentbox">
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

This skill targets **CFML engines** (Lucee 5+, Adobe ColdFusion 2018+). For BoxLang-specific syntax and features, see the BoxLang variant of this skill.
