---
name: contentbox-cfml-admin-extension
description: "Use this skill when extending the ContentBox admin UI with custom menus, views, interception points, editor integrations, workflows, and secure admin-only module behaviors."
applyTo: "**/*.{cfc,cfm,cfml}"
---

# ContentBox Admin Extension (CFML)

Extend the ContentBox admin interface using CFML. Add custom panels, modify existing screens, inject HTML into admin layouts, hook into content lifecycle events, and register new admin functionality.

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

```cfml
<!--- interceptors/AdminAssets.cfc --->
<cfcomponent>

	<cffunction name="configure">
	</cffunction>

	<cffunction name="onCbadmin_beforeHeadEnd" access="public" returntype="void">
		<cfargument name="event" type="any">
		<cfargument name="data" type="struct">
		<cfargument name="buffer" type="any">
		<cfargument name="rc" type="struct">
		<cfargument name="prc" type="struct">

		<cfset buffer.append( '<link rel="stylesheet" href="/modules/mymodule/includes/css/admin.css">' )>
	</cffunction>

	<cffunction name="onCbadmin_beforeBodyEnd" access="public" returntype="void">
		<cfargument name="event" type="any">
		<cfargument name="data" type="struct">
		<cfargument name="buffer" type="any">
		<cfargument name="rc" type="struct">
		<cfargument name="prc" type="struct">

		<cfset buffer.append( '<script src="/modules/mymodule/includes/js/admin.js"></script>' )>
	</cffunction>

</cfcomponent>
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

```cfml
<!--- interceptors/EditorSidebar.cfc --->
<cfcomponent>

	<cffunction name="configure">
	</cffunction>

	<cffunction name="onCbadmin_contentEditorSidebarAccordion" access="public" returntype="void">
		<cfargument name="event" type="any">
		<cfargument name="data" type="struct">
		<cfargument name="buffer" type="any">
		<cfargument name="rc" type="struct">
		<cfargument name="prc" type="struct">

		<cfset var content = "">
		<cfset saveContent variable="content">
			<div class="accordion-section">
				<h4>My Custom Panel</h4>
				<div class="accordion-content">
					<!--- Custom fields or content --->
					<p>Custom panel content here</p>
				</div>
			</div>
		</cfset>
		<cfset buffer.append( content )>
	</cffunction>

</cfcomponent>
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

```cfml
<!--- interceptors/EntryAudit.cfc --->
<cfcomponent>

	<cffunction name="configure">
	</cffunction>

	<cffunction name="onCbadmin_postEntrySave" access="public" returntype="void">
		<cfargument name="event" type="any">
		<cfargument name="data" type="struct">
		<cfargument name="buffer" type="any">
		<cfargument name="rc" type="struct">
		<cfargument name="prc" type="struct">

		<!--- data.entry contains the saved entry --->
		<cfset var entry = data.entry>
		<cfset var log = wirebox.getInstance( "logbox:logger:EntryAudit" )>
		<cfset log.info( "Entry saved: #entry.getTitle()# by #entry.getAuthor().getUsername()#" )>

		<!--- Could also trigger external API, send notifications, etc. --->
	</cffunction>

</cfcomponent>
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

```cfml
<!--- interceptors/AuthorExtension.cfc --->
<cfcomponent>

	<cffunction name="configure">
	</cffunction>

	<cffunction name="onCbadmin_onAuthorEditorSidebar" access="public" returntype="void">
		<cfargument name="event" type="any">
		<cfargument name="data" type="struct">
		<cfargument name="buffer" type="any">
		<cfargument name="rc" type="struct">
		<cfargument name="prc" type="struct">

		<cfset var content = "">
		<cfset saveContent variable="content">
			<div class="form-group">
				<label>Department</label>
				<input type="text" name="department"
					value="#prc.author.getDepartment() ?: ''#"
					class="form-control">
			</div>
		</cfset>
		<cfset buffer.append( content )>
	</cffunction>

	<cffunction name="onCbadmin_postAuthorSave" access="public" returntype="void">
		<cfargument name="event" type="any">
		<cfargument name="data" type="struct">
		<cfargument name="buffer" type="any">
		<cfargument name="rc" type="struct">
		<cfargument name="prc" type="struct">

		<!--- Save custom field --->
		<cfif structKeyExists( rc, "department" )>
			<cfset data.author.setDepartment( rc.department )>
			<cfset authorService = wirebox.getInstance( "authorService@contentbox" )>
			<cfset authorService.save( data.author )>
		</cfif>
	</cffunction>

</cfcomponent>
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

```cfml
<!--- interceptors/AdminMenu.cfc --->
<cfcomponent>

	<cffunction name="configure">
	</cffunction>

	<cffunction name="onCbadmin_onAdminMenuLoad" access="public" returntype="void">
		<cfargument name="event" type="any">
		<cfargument name="data" type="struct">
		<cfargument name="buffer" type="any">
		<cfargument name="rc" type="struct">
		<cfargument name="prc" type="struct">

		<!--- data.menuItems is the array of menu items --->
		<cfset arrayAppend( data.menuItems, {
			name       : "My Module",
			link       : event.buildLink( "cbadmin/myModule" ),
			icon       : "star",
			permission : "MYMODULE_ACCESS",
			order      : 50
		} )>
	</cffunction>

</cfcomponent>
```

## Registering Admin Interceptors

Register your interceptors in your module's `ModuleConfig.cfc`:

```cfml
<cfcomponent>

	<cffunction name="configure">
		<cfset interceptors = [
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
		]>
	</cffunction>

</cfcomponent>
```

## Creating Custom Admin Handlers

Extend `baseHandler` for admin handlers:

```cfml
<!--- handlers/MyModule.cfc --->
<cfcomponent extends="contentbox.modules.contentbox-admin.handlers.baseHandler" singleton>

	<cffunction name="index" access="public" returntype="void">
		<cfset prc.pageTitle = "My Module">
		<cfset setView( "myModule/index" )>
	</cffunction>

	<cffunction name="settings" access="public" returntype="void">
		<cfset prc.pageTitle = "My Module Settings">
		<cfset prc.settings = settingService.getSettings()>
		<cfset setView( "myModule/settings" )>
	</cffunction>

</cfcomponent>
```

## Admin Routes

Define routes in your module's `ModuleConfig.cfc`:

```cfml
<cfset routes = [
	{ pattern = "/cbadmin/myModule", handler = "myModule", action = "index" },
	{ pattern = "/cbadmin/myModule/settings", handler = "myModule", action = "settings" },
	{ pattern = "/cbadmin/myModule/:action", handler = "myModule" }
]>
```

## Admin Security

Admin handlers inherit security from the admin firewall. Configure permissions:

```cfml
<!--- In your module's ModuleConfig.cfc --->
<cfset settings.cbsecurity = {
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
}>
```

## Best Practices

1. **Use interception points** — don't modify core admin files
2. **Register interceptors** in your module's `ModuleConfig.cfc`
3. **Extend `baseHandler`** for admin handlers
4. **Use `prc` scope** for passing data to admin views
5. **Check permissions** before rendering admin content
6. **Use `buffer.append()`** for HTML injection interceptors
7. **Follow admin UI patterns** — match existing styling and structure
8. **Test with all engines** — Lucee, Adobe CF, BoxLang
9. **Use `provider:` injection** to avoid circular dependencies
10. **Document custom interception points** in your module

## Engine Compatibility

This skill targets **CFML engines** (Lucee 5+, Adobe ColdFusion 2018+). For BoxLang-specific syntax and features, see the BoxLang variant of this skill.
