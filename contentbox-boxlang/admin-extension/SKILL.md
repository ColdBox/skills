# ContentBox Admin Extension (BoxLang)

Extend the ContentBox admin interface using BoxLang. Add custom panels, modify existing screens, inject HTML into admin layouts, hook into content lifecycle events, and register new admin functionality.

## Admin Module Overview

The admin module lives at `modules/contentbox/modules/contentbox-admin/` with entry point `cbadmin`. It provides:

- **30+ handlers** for managing content, authors, settings, security, etc.
- **Admin layouts** with consistent navigation and UI
- **Interceptors** for request processing, 2FA enforcement, menu management
- **Interception points** for extending every part of the admin UI

## Admin Extension Points

### 1. Layout HTML Injection

Inject custom HTML into the admin layout at specific points:

| Interception Point | Location |
|-------------------|----------|
| `cbadmin_beforeHeadEnd` | Before `</head>` tag |
| `cbadmin_afterBodyStart` | After `<body>` tag |
| `cbadmin_beforeBodyEnd` | Before `</body>` tag |
| `cbadmin_footer` | Admin footer area |
| `cbadmin_beforeContent` | Before main content area |
| `cbadmin_afterContent` | After main content area |
| `cbadmin_onTagLine` | Tag line area |
| `cbadmin_onTopBar` | Top navigation bar |

#### Example: Inject Custom CSS/JS

```boxlang
// interceptors/AdminAssets.bx
component {

	function configure(){
	}

	function onCbadmin_beforeHeadEnd( event, data, buffer, rc, prc ){
		buffer.append( '<link rel="stylesheet" href="/modules/mymodule/includes/css/admin.css">' )
	}

	function onCbadmin_beforeBodyEnd( event, data, buffer, rc, prc ){
		buffer.append( '<script src="/modules/mymodule/includes/js/admin.js"></script>' )
	}

}
```

### 2. Content Editor Extension

Extend the content editor (entry/page editing screens):

| Interception Point | Location |
|-------------------|----------|
| `cbadmin_contentEditorSidebar` | Sidebar area |
| `cbadmin_contentEditorSidebarAccordion` | Sidebar accordion sections |
| `cbadmin_contentEditorSidebarFooter` | Bottom of sidebar |
| `cbadmin_contentEditorFooter` | Editor footer |
| `cbadmin_contentEditorInBody` | Inside editor body |
| `cbadmin_contentEditorNav` | Editor navigation |
| `cbadmin_contentEditorNavContent` | Editor nav content |

#### Example: Add Custom Sidebar Panel

```boxlang
// interceptors/EditorSidebar.bx
component {

	function configure(){
	}

	function onCbadmin_contentEditorSidebarAccordion( event, data, buffer, rc, prc ){
		content = '
			<div class="accordion-section">
				<h4>My Custom Panel</h4>
				<div class="accordion-content">
					<p>Custom panel content here</p>
				</div>
			</div>
		'
		buffer.append( content )
	}

}
```

### 3. Content Lifecycle Events

Hook into content save, remove, and status change events:

#### Entry Events

| Point | When |
|-------|------|
| `cbadmin_preEntrySave` | Before entry is saved |
| `cbadmin_postEntrySave` | After entry is saved |
| `cbadmin_preEntryRemove` | Before entry is deleted |
| `cbadmin_postEntryRemove` | After entry is deleted |
| `cbadmin_onEntryStatusUpdate` | When entry status changes |

#### Page Events

| Point | When |
|-------|------|
| `cbadmin_prePageSave` | Before page is saved |
| `cbadmin_postPageSave` | After page is saved |
| `cbadmin_prePageRemove` | Before page is deleted |
| `cbadmin_postPageRemove` | After page is deleted |
| `cbadmin_onPageStatusUpdate` | When page status changes |

#### Example: Post-Save Hook

```boxlang
// interceptors/EntryAudit.bx
component {

	property name="log" inject="logbox:logger:EntryAudit"

	function configure(){
	}

	function onCbadmin_postEntrySave( event, data, buffer, rc, prc ){
		// data.entry contains the saved entry
		entry = data.entry
		log.info( "Entry saved: #{entry.getTitle()}# by #{entry.getAuthor().getUsername()}#" )

		// Could also trigger external API, send notifications, etc.
	}

}
```

### 4. Author Extension

Extend author management screens:

| Point | Location |
|-------|----------|
| `cbadmin_UserPreferencePanel` | User preferences panel |
| `cbadmin_onAuthorEditorNav` | Author editor navigation |
| `cbadmin_onAuthorEditorContent` | Author editor content area |
| `cbadmin_onAuthorEditorSidebar` | Author editor sidebar |
| `cbadmin_onAuthorEditorActions` | Author editor action buttons |
| `cbadmin_onNewAuthorForm` | New author form |
| `cbadmin_onNewAuthorActions` | New author form actions |
| `cbadmin_preNewAuthorSave` | Before new author saved |
| `cbadmin_postNewAuthorSave` | After new author saved |
| `cbadmin_onAuthorPasswordChange` | When author password changes |
| `cbadmin_preAuthorPreferencesSave` | Before preferences saved |
| `cbadmin_postAuthorPreferencesSave` | After preferences saved |

#### Example: Add Custom Author Field

```boxlang
// interceptors/AuthorExtension.bx
component {

	property name="authorService" inject="authorService@contentbox"

	function configure(){
	}

	function onCbadmin_onAuthorEditorSidebar( event, data, buffer, rc, prc ){
		content = '
			<div class="form-group">
				<label>Department</label>
				<input type="text" name="department"
					value="#{prc.author?.getDepartment() ?: ''}#"
					class="form-control">
			</div>
		'
		buffer.append( content )
	}

	function onCbadmin_postAuthorSave( event, data, buffer, rc, prc ){
		// Save custom field
		if( structKeyExists( rc, "department" ) ){
			data.author.setDepartment( rc.department )
			authorService.save( data.author )
		}
	}

}
```

### 5. Dashboard Extension

Add content to the admin dashboard:

| Point | Location |
|-------|----------|
| `cbadmin_onDashboard` | Dashboard main area |
| `cbadmin_preDashboardContent` | Before dashboard content |
| `cbadmin_postDashboardContent` | After dashboard content |
| `cbadmin_preDashboardSideBar` | Before dashboard sidebar |
| `cbadmin_postDashboardSideBar` | After dashboard sidebar |
| `cbadmin_onDashboardTabNav` | Dashboard tab navigation |
| `cbadmin_preDashboardTabContent` | Before tab content |
| `cbadmin_postDashboardTabContent` | After tab content |

### 6. Admin Menu Registration

Register custom menu items via `cbadmin_onAdminMenuLoad`:

```boxlang
// interceptors/AdminMenu.bx
component {

	function configure(){
	}

	function onCbadmin_onAdminMenuLoad( event, data, buffer, rc, prc ){
		// data.menuItems is the array of menu items
		arrayAppend( data.menuItems, {
			name       : "My Module",
			link       : event.buildLink( "cbadmin/myModule" ),
			icon       : "star",
			permission : "MYMODULE_ACCESS",
			order      : 50
		} )
	}

}
```

## Registering Admin Interceptors

Register your interceptors in your module's `ModuleConfig.bx`:

```boxlang
// ModuleConfig.bx
interceptors = [
	{
		class : "mymodule.interceptors.AdminAssets",
		name  : "AdminAssets@mymodule"
	},
	{
		class : "mymodule.interceptors.EditorSidebar",
		name  : "EditorSidebar@mymodule"
	},
	{
		class : "mymodule.interceptors.EntryAudit",
		name  : "EntryAudit@mymodule"
	}
]
```

## Creating Custom Admin Handlers

Extend `baseHandler` for admin handlers:

```boxlang
// handlers/MyModule.bx
component extends="contentbox.modules.contentbox-admin.handlers.baseHandler" singleton {

	function index(){
		prc.pageTitle = "My Module"
		setView( "myModule/index" )
	}

	function settings(){
		prc.pageTitle = "My Module Settings"
		prc.settings  = settingService.getSettings()
		setView( "myModule/settings" )
	}

}
```

## Admin Routes

Define routes in your module's `ModuleConfig.bx`:

```boxlang
routes = [
	{ pattern : "/cbadmin/myModule", handler : "myModule", action : "index" },
	{ pattern : "/cbadmin/myModule/settings", handler : "myModule", action : "settings" },
	{ pattern : "/cbadmin/myModule/:action", handler : "myModule" }
]
```

## Admin Security

Admin handlers inherit security from the admin firewall. Configure permissions:

```boxlang
// In your module's ModuleConfig.bx
settings.cbsecurity = {
	firewall : {
		rules : {
			provider : {
				source     : "model",
				properties : {
					model  : "securityRuleService@contentbox",
					method : "getSecurityRules"
				}
			}
		}
	}
}
```

## Best Practices

1. **Use interception points** — don't modify core admin files
2. **Register interceptors** in your module's `ModuleConfig.bx`
3. **Extend `baseHandler`** for admin handlers
4. **Use `prc` scope** for passing data to admin views
5. **Check permissions** before rendering admin content
6. **Use `buffer.append()`** for HTML injection interceptors
7. **Follow admin UI patterns** — match existing styling and structure
8. **Test with all engines** — Lucee, Adobe CF, BoxLang
9. **Use `provider:` injection** to avoid circular dependencies
10. **Document custom interception points** in your module

## Engine Compatibility

This skill targets **BoxLang** engine. For CFML-specific syntax (Lucee 5+, Adobe ColdFusion 2018+), see the CFML variant of this skill.

Key BoxLang advantages:
- Cleaner script syntax without `<cfcomponent>` / `<cffunction>` tags
- No parentheses needed for zero-argument function calls
- `#{...}#` for inline expression output in `.bx` templates
- Modern syntax: `?:` null coalescing, `?.` safe navigation
- Native support for modern data structures
