---
name: coldbox-security-sso
description: "Use this skill when implementing Single Sign-On (SSO) in ColdBox with the cbsso module, configuring OAuth2 providers like Google, Azure AD, or Okta, handling OAuth2 callback flows, mapping SSO identity to local user accounts, or implementing social login buttons with OpenID Connect."
---

# SSO Integration in ColdBox

## Overview

Single Sign-On (SSO) lets users authenticate via external identity providers (Google, Azure AD, Okta, GitHub, etc.) using OAuth2 / OpenID Connect. The `cbsso` module provides the OAuth2 flow and provider configuration.

## Installation

```bash
install cbsso
install cbauth
```

## Module Configuration

```boxlang
// config/ColdBox.cfc
moduleSettings = {
    cbsso: {
        defaultProvider: "google",
        providers: {
            google: {
                clientID:     getSystemSetting( "GOOGLE_CLIENT_ID", "" ),
                clientSecret: getSystemSetting( "GOOGLE_CLIENT_SECRET", "" ),
                redirectURI:  "https://app.example.com/sso/callback/google",
                scope:        "openid email profile"
            },
            azure: {
                clientID:     getSystemSetting( "AZURE_CLIENT_ID", "" ),
                clientSecret: getSystemSetting( "AZURE_CLIENT_SECRET", "" ),
                tenantID:     getSystemSetting( "AZURE_TENANT_ID", "common" ),
                redirectURI:  "https://app.example.com/sso/callback/azure",
                scope:        "openid email profile"
            },
            okta: {
                clientID:     getSystemSetting( "OKTA_CLIENT_ID", "" ),
                clientSecret: getSystemSetting( "OKTA_CLIENT_SECRET", "" ),
                domain:       getSystemSetting( "OKTA_DOMAIN", "" ),
                redirectURI:  "https://app.example.com/sso/callback/okta",
                scope:        "openid email profile"
            }
        }
    }
}
```

## SSO Handler

```boxlang
/**
 * handlers/SSO.cfc
 */
class extends="coldbox.system.EventHandler" {

    property name="ssoService"  inject="SSOService"    // cbsso
    property name="authService" inject="AuthService"
    property name="userService" inject="UserService"
    property name="cbauth"      inject="@cbauth"

    /**
     * Redirect user to provider's login page.
     * GET /sso/login/:provider
     */
    function login( event, rc, prc ) {
        var provider = rc.provider ?: "google"
        var authURL  = ssoService.buildAuthorizationURL( provider )
        relocate( authURL )
    }

    /**
     * OAuth2 callback — exchange code, load/create user, log them in.
     * GET /sso/callback/:provider
     */
    function callback( event, rc, prc ) {
        var provider = rc.provider ?: "google"

        // Exchange authorization code for tokens + identity
        try {
            var identity = ssoService.exchangeCode(
                provider = provider,
                code     = rc.code,
                state    = rc.state ?: ""
            )
        } catch ( any e ) {
            flash.put( "error", "SSO authentication failed: " & e.message )
            return relocate( "auth.login" )
        }

        // Find or create local user
        var user = userService.findByEmail( identity.email )

        if ( isNull( user ) ) {
            user = userService.createFromSSO( {
                email:    identity.email,
                name:     identity.name,
                avatar:   identity.picture ?: "",
                provider: provider,
                sub:      identity.sub       // external user ID
            } )
        } else {
            // Update SSO metadata
            userService.updateSSOMetadata( user.getID(), provider, identity.sub )
        }

        // Log in via cbauth
        cbauth.login( user )

        flash.put( "success", "Welcome, " & user.getName() & "!" )
        relocate( "dashboard" )
    }

    /**
     * SSO logout — local + optional provider logout.
     * GET /sso/logout
     */
    function logout( event, rc, prc ) {
        cbauth.logout()
        relocate( "auth.login" )
    }
}
```

## User Service SSO Methods

```boxlang
/**
 * Additional methods on UserService for SSO account creation and linking.
 */
function createFromSSO( required struct identity ) {
    var userID = createUUID()
    queryExecute(
        "INSERT INTO users (id, email, name, avatar, provider, external_id, email_verified)
         VALUES (:id, :email, :name, :avatar, :provider, :sub, 1)",
        {
            id:       userID,
            email:    arguments.identity.email,
            name:     arguments.identity.name,
            avatar:   arguments.identity.avatar,
            provider: arguments.identity.provider,
            sub:      arguments.identity.sub
        }
    )

    // Auto-assign default role
    roleService.assignRole( userID, roleService.findByName( "member" ).id )

    return findById( userID )
}

function updateSSOMetadata( required userID, required provider, required sub ) {
    queryExecute(
        "UPDATE users SET provider = :provider, external_id = :sub, last_login = NOW()
         WHERE id = :id",
        { provider: arguments.provider, sub: arguments.sub, id: arguments.userID }
    )
}

function findByExternalID( required provider, required sub ) {
    var qry = queryExecute(
        "SELECT * FROM users WHERE provider = :provider AND external_id = :sub",
        { provider: arguments.provider, sub: arguments.sub }
    )
    if ( !qry.recordCount ) return javaCast( "null", "" )
    return entityToUser( qry )
}
```

## View — Social Login Buttons

```html
<!-- Login page with SSO buttons -->
<div class="social-login">
    <a href="#event.buildLink('sso.login')#?provider=google"
       class="btn btn-google">
        <img src="/assets/img/google-icon.svg" alt="Google"> Sign in with Google
    </a>

    <a href="#event.buildLink('sso.login')#?provider=azure"
       class="btn btn-azure">
        <img src="/assets/img/microsoft-icon.svg" alt="Microsoft"> Sign in with Microsoft
    </a>

    <a href="#event.buildLink('sso.login')#?provider=okta"
       class="btn btn-okta">
        Sign in with Okta
    </a>
</div>
```

## Route Configuration

```boxlang
// config/Router.cfc
route( "/sso/login/:provider" ).to( "sso.login" )
route( "/sso/callback/:provider" ).to( "sso.callback" )
route( "/sso/logout" ).to( "sso.logout" )
```

## SSO Provider Reference

| Provider | Required Config | Notes |
|----------|----------------|-------|
| Google | clientID, clientSecret | Create at console.cloud.google.com |
| Azure AD | clientID, clientSecret, tenantID | tenantID = your-tenant or "common" |
| Okta | clientID, clientSecret, domain | domain = yourorg.okta.com |
| GitHub | clientID, clientSecret | Scope: "read:user user:email" |
| Auth0 | clientID, clientSecret, domain | Most flexible option |

## Security Notes

- Store OAuth2 secrets in environment variables, not source code
- Validate `state` parameter to prevent CSRF on callback
- Always verify email_verified claim from provider
- Use HTTPS for all redirect URIs
- Log SSO login events for audit trail
