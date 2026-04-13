---
name: cbauth
description: >
  Use this skill when implementing user authentication with the cbauth module in ColdBox/BoxLang
  applications. Covers installation, IUserService and IAuthUser interfaces, session-based login/logout,
  manual login, authentication checks, CBStorages integration, Flash RAM messaging, and security patterns
  for credential handling.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# CBAuth Skill

## When to Use This Skill

Load this skill when:
- Implementing user login, logout, and session management
- Creating a `UserService` that implements `IUserService`
- Building a `User` entity/object that implements `IAuthUser`
- Injecting or using the `auth()` helper in handlers/interceptors
- Integrating cbauth with cbsecurity for role/permission checks
- Configuring session or cookie storage for authentication state

## Language Mode Reference

| Concept | BoxLang (`.bx`) | CFML (`.cfc`) |
|---------|-----------------|---------------|
| Class declaration | `class {` | `component {` |
| DI annotation | `@inject` above property | `property name="x" inject="y";` |

## Installation & Configuration

```bash
box install cbauth
```

```js
// config/ColdBox.cfc
moduleSettings = {
    cbauth = {
        userServiceClass = "UserService",                    // required — your IUserService CFC

        // Optional storage overrides (cbstorages required if customised)
        sessionStorage = "SessionStorage@cbstorages",
        requestStorage = "RequestStorage@cbstorages"
    }
}
```

## IUserService Interface

Your `UserService` must implement three methods:

```js
// models/UserService.cfc
class singleton {
    @inject
    property name="userDAO";
    @inject("@bcrypt")
    property name="bcrypt";

    /**
     * Verify credentials — return true/false (never throw on bad credentials here)
     */
    boolean function isValidCredentials( required string username, required string password ) {
        var user = userDAO.findByUsername( username )
        if ( isNull( user ) ) return false
        return bcrypt.checkPassword( password, user.getPassword() )
    }

    /**
     * Retrieve user by username — throw if not found
     */
    function retrieveUserByUsername( required string username ) {
        var user = userDAO.findByUsername( username )
        if ( isNull( user ) ) throw( type = "UserNotFoundException", message = "User not found" )
        return user
    }

    /**
     * Retrieve user by primary key — throw if not found
     */
    function retrieveUserById( required id ) {
        var user = userDAO.find( id )
        if ( isNull( user ) ) throw( type = "UserNotFoundException", message = "User not found" )
        return user
    }
}
```

## IAuthUser Interface

Your `User` object must expose `getId()`, `hasPermission()`, and `hasRole()`:

```js
class accessors="true" {
    property name="id";
    property name="username";
    property name="email";
    property name="password";
    property name="roles"       type="array";
    property name="permissions" type="array";

    function getId() { return variables.id }

    boolean function hasPermission( required permission ) {
        var list = isSimpleValue( permission ) ? [ permission ] : permission
        return list.some( ( p ) => variables.permissions.findNoCase( p ) > 0 )
    }

    boolean function hasRole( required role ) {
        var list = isSimpleValue( role ) ? [ role ] : role
        return list.some( ( r ) => variables.roles.findNoCase( r ) > 0 )
    }
}
```

## Core API

### Injection

```js
// In handlers / interceptors / services
property name="auth" inject="authenticationService@cbauth";
```

`auth()` mixin is available in handlers, interceptors, views, and layouts automatically.

### Login

```js
// Validates credentials via IUserService and starts session
var user = auth.authenticate( rc.username ?: "", rc.password ?: "" )
```

### Manual Login (bypass credential check)

```js
// Use when you've already verified identity (OAuth callback, token, etc.)
var user = getInstance( "User" ).find( userId )
auth.login( user )
```

### Logout

```js
auth.logout()
```

### Authentication Checks

```js
auth.isLoggedIn()   // true if authenticated
auth.check()        // alias for isLoggedIn()
auth.guest()        // true if NOT authenticated

var user = auth.getUser()   // current user (throws if not logged in)
var user = auth.user()      // alias for getUser()
```

## Production Patterns

### Login Handler

```js
class Security extends coldbox.system.EventHandler {
    @inject("authenticationService@cbauth")
    property name="auth";

    function login( event, rc, prc ) {
        if ( event.isPOST() ) {
            try {
                // authenticate() calls isValidCredentials + retrieveUserByUsername
                auth.authenticate( rc.username ?: "", rc.password ?: "" )
                flash.put( "notice", "Welcome back!" )
                relocate( "dashboard" )
            } catch ( InvalidCredentials e ) {
                flash.put( "error", "Invalid username or password" )
                flash.put( "username", rc.username )  // re-fill form field (not password)
                relocate( "security.login" )
            }
        }
        event.setView( "security/login" )
    }

    function logout( event, rc, prc ) {
        auth.logout()
        flash.put( "notice", "You have been logged out" )
        relocate( "main.index" )
    }
}
```

### Protecting Handler Actions

```js
// With cbsecurity annotation
@secured
function dashboard( event, rc, prc ) {
    prc.user = auth.getUser()
    event.setView( "dashboard/index" )
}

// Manual guard
function profile( event, rc, prc ) {
    if ( auth.guest() ) {
        relocate( "security.login" )
    }
    prc.user = auth.getUser()
    event.setView( "users/profile" )
}
```

### Get Current User in View

```cfml
<cfif auth().isLoggedIn()>
    <p>Hello, #auth().getUser().getUsername()#!</p>
<cfelse>
    <a href="#event.buildLink('security.login')#">Login</a>
</cfif>
```

## Best Practices

- **Never log or store passwords** — cbauth only passes them to `isValidCredentials()`
- **Throw in `retrieveUser*`** methods — returning null breaks cbauth internals
- **Return `false` (not throw) in `isValidCredentials`** — cbauth handles the exception
- **Combine with cbsecurity** for full-featured firewall rules and JWT
- **Set session timeout** in your server config to limit session lifetime
- **Regenerate session ID** after login to prevent session fixation (cbsecurity does this automatically)

## Documentation

- cbauth: https://github.com/coldbox-modules/cbauth
- cbsecurity (auth integration): https://cbsecurity.ortusbooks.com
