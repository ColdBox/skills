---
name: coldbox-security-authorization
description: "Use this skill when implementing authorization in ColdBox with CBSecurity, defining security rules for events and URLs, checking user roles with cbsecurity.has(), checking permissions with cbsecurity.can(), implementing role-based route protection, using @secured annotations on handler actions, or blocking unauthorized access."
---

# Authorization Patterns in ColdBox

## Overview

Authorization in ColdBox is handled by CBSecurity through security rules, role checks, permission checks, and handler-level annotations. It determines what authenticated users can access.

## CBSecurity Rules

### Event-Based Rules

```boxlang
// config/ColdBox.cfc
moduleSettings = {
    cbsecurity: {
        rules: [
            // Secure specific event
            {
                secureList: "admin.index",
                match: "event",
                roles: "admin",
                action: "redirect",
                redirect: "/login"
            },
            // Secure multiple events
            {
                secureList: "users.edit,users.delete",
                match: "event",
                roles: "admin",
                action: "block"
            },
            // Regex pattern for all admin events
            {
                secureList: "^admin\\.",
                match: "event",
                roles: "admin,superadmin",
                action: "redirect",
                redirect: "/login"
            },
            // Public whitelist
            {
                whitelist: "main\\.index,auth\\..*",
                match: "event"
            }
        ]
    }
}
```

### URL-Based Rules

```boxlang
rules: [
    {
        secureList: "/admin",
        match: "url",
        roles: "admin",
        action: "redirect"
    },
    {
        secureList: "^/api/.*",
        match: "url",
        roles: "api_user",
        action: "block",
        statusCode: 401
    }
]
```

## Role and Permission Checking in Handlers

```boxlang
class extends="coldbox.system.EventHandler" {

    property name="cbsecurity" inject="@cbsecurity"

    function edit( event, rc, prc ) {
        // Single role check
        if ( !cbsecurity.has( "admin" ) ) {
            cbsecurity.block( "Admin access required" )
        }

        // OR logic — any of these roles passes
        if ( !cbsecurity.has( "admin,manager" ) ) {
            cbsecurity.block( "Insufficient privileges" )
        }

        // AND logic — user must have all these roles
        if ( !cbsecurity.all( "admin,moderator" ) ) {
            cbsecurity.block( "Need both admin and moderator roles" )
        }

        // Permission check
        if ( !cbsecurity.can( "users.edit" ) ) {
            cbsecurity.block( "Missing permission: users.edit" )
        }

        event.setView( "users/edit" )
    }
}
```

## @secured Annotation (Handler-Level)

```boxlang
/**
 * Secure entire handler (all actions require login)
 * @secured
 */
class extends="coldbox.system.EventHandler" {

    /**
     * Requires admin role
     * @secured admin
     */
    function deleteUser( event, rc, prc ) {
        // Only admin role gets here
    }

    /**
     * Requires specific permission
     * @secured users.create
     */
    function create( event, rc, prc ) {
        // Only users with users.create permission get here
    }

    /**
     * Requires role AND permission
     * @secured admin,users.delete
     */
    function destroy( event, rc, prc ) {
        // Requires admin role AND users.delete permission
    }
}
```

## Resource-Based Authorization

```boxlang
class extends="coldbox.system.EventHandler" {

    property name="cbsecurity"  inject="@cbsecurity"
    property name="postService" inject="PostService"
    property name="auth"        inject="authenticationService@cbauth"

    function edit( event, rc, prc ) {
        event.paramValue( "id", 0 )
        prc.post = postService.findById( rc.id )

        // Resource-owner check (user owns this resource)
        if ( prc.post.authorId != auth.getUserId() && !cbsecurity.has( "admin" ) ) {
            cbsecurity.block( "You can only edit your own posts" )
        }

        event.setView( "posts/edit" )
    }
}
```

## Permission-Based Authorization

```boxlang
// models/User.cfc — user entity with roles/permissions
class {

    property name="id"
    property name="name"
    property name="roles" default=""
    property name="permissions" default=""

    function hasRole( required role ) {
        return listContains( variables.roles, arguments.role )
    }

    function can( required permission ) {
        return listContains( variables.permissions, arguments.permission )
    }

    function getRolesList() {
        return listToArray( variables.roles )
    }
}
```

## Firewall Configuration

```boxlang
moduleSettings = {
    cbsecurity: {
        firewall: {
            enabled: true,
            // "redirect" for web apps, "block" for APIs
            defaultAction: "redirect",
            defaultRedirect: "/login",
            // Status code when action = "block"
            statusCode: 401,
            // Override rejection handler
            invalidAuthenticationEvent: "auth.login",
            invalidAuthorizationEvent: "main.accessDenied"
        }
    }
}
```

## View-Level Authorization

```html
<!-- Show/hide UI based on role -->
<cfif cbsecurity.has( "admin" )>
    <a href="#event.buildLink( 'admin.index' )#">Admin Panel</a>
</cfif>

<cfif cbsecurity.can( "users.create" )>
    <a href="#event.buildLink( 'users.create' )#">Add User</a>
</cfif>

<cfif auth.isLoggedIn()>
    <span>Welcome, #auth.getUser().name#</span>
</cfif>
```

## CBSecurity Method Reference

| Method | Description |
|--------|-------------|
| `cbsecurity.has( "role" )` | Check if user has role (OR for comma list) |
| `cbsecurity.all( "role1,role2" )` | Check if user has ALL roles (AND) |
| `cbsecurity.can( "permission" )` | Check if user has specific permission |
| `cbsecurity.block( message )` | Block/reject current request |
| `cbsecurity.isLoggedIn()` | Check authentication status |
| `cbsecurity.getUser()` | Get authenticated user |
| `cbsecurity.isAdmin()` | Check if user is admin |
