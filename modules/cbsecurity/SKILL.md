---
name: cbsecurity
description: >
  Use this skill when securing ColdBox/BoxLang applications with cbsecurity. Covers firewall rule
  configuration, annotation-based security on handlers/actions, JWT authentication, role and permission
  checks, route middleware (Authenticated, Authorized, JwtAuth, BasicAuth, Throttle, ApiKey, AllowedIPs, DenyIPs,
  EnsureHttps, VerifyCsrf, Honeypot, Signed), signed URLs, security context helpers,
  custom validators, interceptor events, and production hardening patterns.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# CBSecurity Skill

## When to Use This Skill

Load this skill when:
- Protecting handlers and actions with annotation-based security
- Configuring firewall rules for route-level access control
- Securing routes and route groups where they are declared with route middleware
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

## Route Middleware

*cbsecurity 3.9+ on ColdBox 8.2+.* Secure a route, or a group, where it is declared, with no firewall
rules. Denied requests go through the firewall's invalid access flow, so the `redirect`, `override` and
`block` actions, module overrides, interception points and logging behave like a rule or annotation.

| WireBox ID | Verifies |
|---|---|
| `Authenticated@cbsecurity` | The user is logged in. Route `meta` is ignored. |
| `Authorized@cbsecurity` | Logged in and satisfying the route `meta` `permissions` and/or `roles` |
| `JwtAuth@cbsecurity` | Like `Authorized`, authenticating through the JWT validator |
| `BasicAuth@cbsecurity` | Like `Authorized`, authenticating through the Basic Auth validator |

Permissions and roles live in the route `meta()`: `permissions` and `roles` take one value, a list or an
array, and `mode` is `any` (default), `all` or `none` for the permissions.

```js
route( "/account" ).middleware( "Authenticated@cbsecurity" ).to( "account.index" )

route( "/billing" )
    .middleware( "Authorized@cbsecurity" )
    .meta( { permissions: "BILLING_READ,BILLING_WRITE", mode: "all" } )
    .to( "billing.index" )

// a group shares middleware and meta (group meta needs ColdBox 8.3+)
group( { pattern: "/admin", middleware: [ "Authorized@cbsecurity" ], meta: { permissions: "ADMIN" } }, () => {
    route( "/users" ).to( "admin.users" )
    route( "/status" ).withoutMiddleware( "Authorized@cbsecurity" ).to( "admin.status" )
} )
```

Things to know:
- **Parameters go in `meta()`**, not in the middleware string. A router file loads before modules, so
  a factory such as `getInstance( "SomeFactory@cbsecurity" )` cannot be called inside `Router.configure()`.
- The **firewall interceptor must be loaded** (`firewall.autoLoadFirewall`, default `true`), otherwise
  `cbsecurity.MiddlewareRequiresFirewall` is thrown. No firewall rules are needed.
- The JWT validator checks **permissions only** (token scopes or user permissions). `roles` in the route
  meta are not evaluated for JWT requests.
- Global firewall rules and annotations run first, then the route middleware.
- Custom middleware: extend `cbsecurity.models.middleware.Guard` and call
  `super.init( permissions = "ADMIN", useMeta = false )`.
- To use a short name, alias it with ColdBox 8.3+: `registerMiddleware( "auth", "Authenticated@cbsecurity" )`.

### Abuse Protection Middleware

These do not authenticate. They answer denied requests directly with a JSON error and the right status
code (not the firewall's invalid actions). Parameters go in the route `meta()`, defaults in the
`middleware` and `signedUrls` module settings.

| WireBox ID | Does | Route `meta()` keys |
|---|---|---|
| `Throttle@cbsecurity` | Rate limit, `429` + `Retry-After` | `throttle`: a named limiter string or `{ maxAttempts, decaySeconds, cacheProvider, by, name }` |
| `ApiKey@cbsecurity` | `401` without a valid key | `apiKeys`, `apiKeyHeader` (default `x-api-key`), `apiKeyParam` (default `apiKey`) |
| `AllowedIPs@cbsecurity` | `403` unless the IP is listed | `allowedIps` (IPv4, IPv6, CIDR) |
| `DenyIPs@cbsecurity` | `403` if the IP is listed | `denyIps` |
| `EnsureHttps@cbsecurity` | `301` for GET/HEAD, `403` otherwise | `redirectToHttps` |
| `VerifyCsrf@cbsecurity` | `403` without a valid cbcsrf token on unsafe methods | `csrfKey` |
| `Honeypot@cbsecurity` | Silent `200` (or `422`) when the trap field is filled | `honeypotField`, `honeypotSilent` |
| `Signed@cbsecurity` | `403` unless the signed URL is valid | none |

```js
// config/ColdBox.bx
moduleSettings = { cbsecurity : { middleware : {
    trustedProxies : [ "10.0.0.0/8" ],
    throttle       : { limiters : { login : { maxAttempts : 5, decaySeconds : 60 } } }
} } }

// config/Router.bx: stack them, cheap checks first
route( "/login" )
    .middleware( [ "EnsureHttps@cbsecurity", "Throttle@cbsecurity" ] )
    .meta( { throttle : "login" } )
    .to( "sessions.create" )
```

- `Throttle` counts in a CacheBox cache (`cacheProvider`, default `"default"`). Use a shared cache with more
  than one server. Use the `RateLimiter@cbsecurity` model (`hit`, `tooManyAttempts`, `remaining`,
  `availableIn`, `clear`) outside routes.
- A missing list or keys throws `cbsecurity.MiddlewareMisconfigured`. `X-Forwarded-For` is only trusted from
  `middleware.trustedProxies`.

### Signed URLs

Set `signedUrls.secret` (env `CBSECURITY_SIGNING_SECRET`), protect the route with `Signed@cbsecurity`, and
create links with the mixins available in handlers, views, layouts and interceptors:

```js
route( pattern = "/invoices/:id/download", name = "invoice.download" )
    .middleware( "Signed@cbsecurity" ).to( "invoices.download" )

var link = signedRoute( "invoice.download", { id: 42 }, 3600 )  // name, params, expiresIn seconds
var link = signedUrl( "/files/9", { ref: "mail" }, 600 )
if ( hasValidSignature() ) { ... }
```

In models inject `UrlSigner@cbsecurity` (`sign`, `isValid`, `check`). Every query param is signed, so adding
one invalidates the link. Param names are case insensitive and scheme and host are ignored.

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
// config/Router.bx: secure all /api routes via JWT (see Route Middleware above)
group( { pattern: "/api", middleware: [ "JwtAuth@cbsecurity" ] }, () => {
    route( "/profile" ).to( "API.Profile.index" )
    route( "/orders/:id" ).meta( { permissions: "ORDERS_WRITE" } ).to( "API.Orders.update" )
} )
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
