---
name: cbsecurity
description: >
  Use this skill when securing ColdBox/BoxLang applications with cbsecurity. Covers firewall rule
  configuration, annotation-based security on handlers/actions, JWT authentication, role and permission
  checks, security context helpers, custom validators, interceptor events, and production hardening patterns.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# CBSecurity Skill

## When to Use This Skill

Load this skill when:
- Protecting handlers and actions with annotation-based security
- Configuring firewall rules for route-level access control
- Implementing JWT-based API authentication
- Checking roles, permissions, or authentication status in code
- Writing custom security validators or user services
- Handling unauthorized/unauthenticated redirects and error responses

## Installation

```bash
box install cbsecurity
```

## Configuration

### config/modules/cbsecurity.cfc

```js
function configure() {
    return {
        // The WireBox ID of the User service implementing IUserService
        userService : "UserService",

        // Authentication handler
        authentication : {
            // WireBox ID of provider: cfauth, jwt, etc.
            provider : "AuthenticationService@cbauth"
        },

        firewall : {
            // No default for all requests? set to true to force all routes to be secured
            autoSecureAll : false,

            // Redirect unauthenticated requests (HTML apps)
            invalidAuthenticationEvent  : "security.login",
            defaultAuthenticationAction : "redirect",

            // What happens when authenticated but not authorized
            invalidAuthorizationEvent   : "security.unauthorized",
            defaultAuthorizationAction  : "redirect",

            // Rules evaluated in order — first match wins
            rules : [
                // Bypass public assets
                { secureList: "^/assets", whiteList: true },
                // Require authentication for everything under /admin
                { secureList: "^/admin",  authenticate: true },
                // Require 'admin' role for user management
                { secureList: "^/admin/users", roles: "admin" }
            ]
        },

        jwt : {
            secretKey        : getSystemSetting( "JWT_SECRET", "" ),
            expiration       : 60,                  // minutes
            algorithm        : "HS512",
            tokenStorage     : {
                // Store issued tokens for revocation support
                enabled    : true,
                driver     : "cachebox",
                properties : { cacheName: "default" }
            }
        }
    }
}
```

## Annotation-Based Security

```js
// On the entire handler
@secured
class UsersHandler extends coldbox.system.EventHandler {

    // Public action in a secured handler — override with no security
    @unsecured
    function login( event, rc, prc ) {}

    // Require specific role
    @secured( "admin" )
    function delete( event, rc, prc ) {}

    // Require specific permission
    @secured( permissions="users:delete" )
    function destroy( event, rc, prc ) {}

    // Multiple roles
    @secured( "admin,manager" )
    function index( event, rc, prc ) {}
}
```

## Security Context (In Code)

```js
property name="security" inject="SecurityService@cbsecurity";

// Check authentication
if ( security.isLoggedIn() ) { ... }

// Get current user
var user = security.getCurrentUser()

// Check role
if ( security.hasRole( "admin" ) ) { ... }

// Check permission
if ( security.hasPermission( "reports:view" ) ) { ... }

// Require auth or throw exception
security.secure()             // throws NotAuthenticatedException
security.secure( "admin" )    // throws NotAuthorizedException if not admin
```

## JWT API Authentication

### Login Endpoint

```js
// handlers/API/Auth.bx
@unsecured
class {

    property name="jwtService" inject="JwtService@cbsecurity";
    property name="userService" inject="UserService";
    property name="bcrypt"     inject="BCryptService@bcrypt";

    function login( event, rc, prc ) {
        var user = userService.findByEmail( rc.email ?: "" )

        if ( isNull( user ) || !bcrypt.checkPassword( rc.password ?: "", user.getPassword() ) ) {
            return event.renderData(
                type       = "json",
                statusCode = 401,
                data       = { error: "Invalid credentials" }
            )
        }

        event.renderData( type = "json", data = {
            token : jwtService.attempt( rc.email, rc.password ),
            user  : user.getMemento()
        } )
    }

    function logout( event, rc, prc ) {
        jwtService.logout()
        event.renderData( type = "json", data = { message: "Logged out" } )
    }

    // Refresh token
    function refresh( event, rc, prc ) {
        event.renderData( type = "json", data = {
            token: jwtService.refreshToken( rc.token ?: "" )
        } )
    }
}
```

### Securing API Routes

```js
// router.bx — secure all /api routes via JWT
route( "/api/profile" )
    .withMiddleware( "JwtAuthFilter@cbsecurity" )
    .toAction( "API/Profile/index" )
```

## IUserService Required Methods

```js
// services/UserService.bx
class implements="cbsecurity.interfaces.IUserService" {

    User function loadUserByUsername( username ) {
        return queryExecute( "SELECT * FROM users WHERE email = :email",
            { email: { value: username, cfsqltype: "cf_sql_varchar" } }
        )
    }

    boolean function isValidCredentials( username, password ) {
        var user = loadUserByUsername( username )
        return !isNull( user ) && bcrypt.checkPassword( password, user.password )
    }
}
```

## Production Patterns

### Handler — Admin Area

```js
@secured( "admin" )
class AdminDashboardHandler extends coldbox.system.EventHandler {

    function index( event, rc, prc ) {
        prc.stats = adminService.getDashboardStats()
        event.setView( "admin/dashboard" )
    }

    @secured( "superadmin" )
    function purgeCache( event, rc, prc ) {
        cacheBox.clearAll()
        messagebox.success( "Cache cleared." )
        relocate( "admin.dashboard" )
    }
}
```

### REST Permission Guard

```js
function update( event, rc, prc ) {
    var resource = resourceService.getOrFail( rc.id )

    // Only owner or admin may edit
    if ( resource.getUserId() != security.getCurrentUser().getId() ) {
        security.secure( "admin" )   // throws if not admin
    }

    resourceService.update( resource, rc )
    event.renderData( type = "json", data = resource.getMemento() )
}
```

## Best Practices

- **Store JWT secret in environment variable** — never hardcode it in config files
- **Enable token storage** for JWT revocation support (logout/invalidation)
- **Use annotation-based security** for handler-level control — keeps authorization close to the code
- **Use firewall rules** for broad, route-level access patterns (e.g., entire `/admin` prefix)
- **Return 401 vs 403 correctly**: 401 = not authenticated, 403 = authenticated but not authorized
- **Never trust `rc` data for permission logic** — always derive from the authenticated session/JWT
- **Implement `isValidCredentials` securely** — always use constant-time comparison (bcrypt)
- **Scope JWT expiration** tightly — short-lived tokens (≤60 min) with refresh tokens for APIs

## Documentation

- cbsecurity: https://github.com/coldbox-modules/cbsecurity
- cbsecurity docs: https://coldbox-security.ortusbooks.com
