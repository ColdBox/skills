---
name: coldbox-security-implementation
description: "Use this skill when setting up the full CBSecurity framework in ColdBox, configuring the security firewall, creating authentication services, implementing security event handlers, configuring security rules and validators, or building a complete security layer for a ColdBox application."
---

# CBSecurity Implementation in ColdBox

## Overview

CBSecurity is ColdBox's enterprise security framework providing authentication, authorization, firewall rules, security handlers, and a security event model that protects all routes/events.

## Language Mode Reference

Examples use **BoxLang (`.bx`)** syntax by default. Adapt for your target language:

| Concept | BoxLang (`.bx`) | CFML (`.cfc`) |
|---------|-----------------|---------------|
| Class declaration | `class [extends="..."] {` | `component [extends="..."] {` |
| DI annotation | `@inject` above `property name="svc";` | `property name="svc" inject="svc";` |
| View templates | `.bxm` suffix | `.cfm` / `.cfml` suffix |
| Tag prefix | `<bx:if>`, `<bx:output>`, `<bx:set>` | `<cfif>`, `<cfoutput>`, `<cfset>` |

> **CFML Compat Mode**: With BoxLang + CFML Compat module, `.bx` and `.cfc` files coexist freely. BoxLang-native classes use `class {}` (`.bx` files); CFML-compat classes use `component {}` (`.cfc` files).

## Installation

```bash
box install cbsecurity
box install cbauth
box install bcrypt
```

## Full Configuration

```boxlang
// config/ColdBox.cfc
moduleSettings = {
    cbsecurity: {
        // Authentication service
        authenticationService: "SecurityService@models",

        // Firewall
        firewall: {
            enabled: true,
            defaultAction: "redirect",
            defaultRedirect: "/login",
            // For APIs: use "block" with 401
            statusCode: 401,
            // Override rejection targets
            invalidAuthenticationEvent: "auth.login",
            invalidAuthorizationEvent:   "main.accessDenied"
        },

        // Security rules (processed top-to-bottom, first match wins)
        rules: [
            {
                whitelist: "auth\\..*,main\\.index,main\\.about",
                match: "event"
            },
            {
                secureList: "^admin\\.",
                match: "event",
                roles: "admin,superadmin",
                action: "redirect",
                redirect: "/login"
            },
            {
                secureList: ".*",
                match: "event",
                action: "redirect",
                redirect: "/login"
            }
        ]
    }
}
```

## Authentication Service

```boxlang
/**
 * models/SecurityService.cfc
 */
class singleton {

    property name="userService"     inject="UserService"
    property name="bcrypt"          inject="@BCrypt"
    property name="sessionStorage"  inject="sessionStorage@cbstorages"

    function authenticate( required username, required password ) {
        try {
            var user = userService.findByUsername( arguments.username )

            if ( !bcrypt.checkPassword( arguments.password, user.password ) ) {
                throw( type: "InvalidCredentials" )
            }

            sessionStorage.set( "currentUser", user )
            return true

        } catch ( any e ) {
            return false
        }
    }

    function getUser() {
        return sessionStorage.get( "currentUser", {} )
    }

    function isLoggedIn() {
        return sessionStorage.exists( "currentUser" )
    }

    function logout() {
        sessionStorage.delete( "currentUser" )
    }

    function hasRole( required role ) {
        var user = getUser()
        return structIsEmpty( user ) ? false : listContains( user.roles, arguments.role )
    }

    function can( required permission ) {
        var user = getUser()
        return structIsEmpty( user ) ? false : listContains( user.permissions, arguments.permission )
    }
}
```

**CFML (`.cfc`):**

```cfml
/**
 * models/SecurityService.cfc
 */
component {

    property name="userService"     inject="UserService"
    property name="bcrypt"          inject="@BCrypt"
    property name="sessionStorage"  inject="sessionStorage@cbstorages"

    function authenticate( required username, required password ) {
        try {
            var user = userService.findByUsername( arguments.username )

            if ( !bcrypt.checkPassword( arguments.password, user.password ) ) {
                throw( type: "InvalidCredentials" )
            }

            sessionStorage.set( "currentUser", user )
            return true

        } catch ( any e ) {
            return false
        }
    }

    function getUser() {
        return sessionStorage.get( "currentUser", {} )
    }

    function isLoggedIn() {
        return sessionStorage.exists( "currentUser" )
    }

    function logout() {
        sessionStorage.delete( "currentUser" )
    }

    function hasRole( required role ) {
        var user = getUser()
        return structIsEmpty( user ) ? false : listContains( user.roles, arguments.role )
    }

    function can( required permission ) {
        var user = getUser()
        return structIsEmpty( user ) ? false : listContains( user.permissions, arguments.permission )
    }
}
```

## Security Interceptor

```boxlang
/**
 * interceptors/SecurityInterceptor.cfc
 * Runs on every request to check auth status
 */
class extends="coldbox.system.Interceptor" {

    property name="cbsecurity" inject="@cbsecurity"

    function preProcess( event, interceptData ) {
        // Skip public routes
        if ( isPublicRoute( event.getCurrentEvent() ) ) {
            return
        }

        // Check authentication
        if ( !cbsecurity.isLoggedIn() ) {
            flash.put( "redirectTo", cgi.script_name & "?" & cgi.query_string )
            relocate( "auth.login" )
        }
    }

    private function isPublicRoute( required eventName ) {
        var publicRoutes = [
            "auth.login", "auth.doLogin",
            "auth.register", "auth.doRegister",
            "main.index"
        ]
        return publicRoutes.contains( arguments.eventName )
    }
}
```

**CFML (`.cfc`):**

```cfml
/**
 * interceptors/SecurityInterceptor.cfc
 * Runs on every request to check auth status
 */
component extends="coldbox.system.Interceptor" {

    property name="cbsecurity" inject="@cbsecurity"

    function preProcess( event, interceptData ) {
        // Skip public routes
        if ( isPublicRoute( event.getCurrentEvent() ) ) {
            return
        }

        // Check authentication
        if ( !cbsecurity.isLoggedIn() ) {
            flash.put( "redirectTo", cgi.script_name & "?" & cgi.query_string )
            relocate( "auth.login" )
        }
    }

    private function isPublicRoute( required eventName ) {
        var publicRoutes = [
            "auth.login", "auth.doLogin",
            "auth.register", "auth.doRegister",
            "main.index"
        ]
        return publicRoutes.contains( arguments.eventName )
    }
}
```

## LoginHandler

```boxlang
/**
 * handlers/Auth.cfc
 */
class extends="coldbox.system.EventHandler" {

    property name="security" inject="SecurityService@models"

    function login( event, rc, prc ) {
        if ( security.isLoggedIn() ) {
            relocate( uri = "main.dashboard" )
        }
        event.setView( "auth/login" )
    }

    function doLogin( event, rc, prc ) {
        event.paramValue( "email",    "" )
        event.paramValue( "password", "" )

        if ( security.authenticate( rc.email, rc.password ) ) {
            // Redirect to originally requested URL if available
            var destination = flash.get( "redirectTo", "main.dashboard" )
            relocate( uri = destination )
        } else {
            flash.put( "error", "Invalid credentials" )
            relocate( "auth.login" )
        }
    }

    function doLogout( event, rc, prc ) {
        security.logout()
        relocate( "auth.login" )
    }
}
```

**CFML (`.cfc`):**

```cfml
/**
 * handlers/Auth.cfc
 */
component extends="coldbox.system.EventHandler" {

    property name="security" inject="SecurityService@models"

    function login( event, rc, prc ) {
        if ( security.isLoggedIn() ) {
            relocate( uri = "main.dashboard" )
        }
        event.setView( "auth/login" )
    }

    function doLogin( event, rc, prc ) {
        event.paramValue( "email",    "" )
        event.paramValue( "password", "" )

        if ( security.authenticate( rc.email, rc.password ) ) {
            // Redirect to originally requested URL if available
            var destination = flash.get( "redirectTo", "main.dashboard" )
            relocate( uri = destination )
        } else {
            flash.put( "error", "Invalid credentials" )
            relocate( "auth.login" )
        }
    }

    function doLogout( event, rc, prc ) {
        security.logout()
        relocate( "auth.login" )
    }
}
```

## Security Events (Interceptors)

```boxlang
/**
 * interceptors/SecurityEvents.cfc
 * Log security events for audit trail
 */
class extends="coldbox.system.Interceptor" {

    property name="logger" inject="logbox:logger:{this}"

    function CBSecurity_onInvalidAuthentication( event, interceptData ) {
        logger.warn( "Authentication failure: event=#interceptData.event#, ip=#cgi.remote_addr#" )
    }

    function CBSecurity_onInvalidAuthorization( event, interceptData ) {
        logger.warn( "Authorization failure: event=#interceptData.event#, user=#interceptData.userId#" )
    }

    function CBSecurity_onAuthentication( event, interceptData ) {
        logger.info( "User logged in: #interceptData.userId#" )
    }

    function CBSecurity_onLogout( event, interceptData ) {
        logger.info( "User logged out: #interceptData.userId#" )
    }
}
```

**CFML (`.cfc`):**

```cfml
/**
 * interceptors/SecurityEvents.cfc
 * Log security events for audit trail
 */
component extends="coldbox.system.Interceptor" {

    property name="logger" inject="logbox:logger:{this}"

    function CBSecurity_onInvalidAuthentication( event, interceptData ) {
        logger.warn( "Authentication failure: event=#interceptData.event#, ip=#cgi.remote_addr#" )
    }

    function CBSecurity_onInvalidAuthorization( event, interceptData ) {
        logger.warn( "Authorization failure: event=#interceptData.event#, user=#interceptData.userId#" )
    }

    function CBSecurity_onAuthentication( event, interceptData ) {
        logger.info( "User logged in: #interceptData.userId#" )
    }

    function CBSecurity_onLogout( event, interceptData ) {
        logger.info( "User logged out: #interceptData.userId#" )
    }
}
```

## Access Denied Handler

```boxlang
/**
 * handlers/Main.cfc
 */
function accessDenied( event, rc, prc ) {
    event.setHTTPHeader( statusCode: 403 )
    event.setView( "errors/403" )
}
```

## Security Checklist

- [x] Install cbsecurity, cbauth, bcrypt
- [x] Configure `authenticationService` in moduleSettings
- [x] Define firewall rules (whitelist public routes, secure the rest)
- [x] Use BCrypt for password hashing — never store plain text passwords
- [x] Implement role/permission checking in handlers with `@secured`
- [x] Log security events with LogBox
- [x] Use HTTPS in production
- [x] Set `JWT_SECRET` and other secrets via environment variables
- [x] Validate all user input at system boundaries
