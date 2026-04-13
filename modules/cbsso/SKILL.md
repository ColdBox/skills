---
name: cbsso
description: >
  Use this skill when implementing Single Sign-On (SSO) in ColdBox/BoxLang using the cbsso module.
  Covers SAML2 and OAuth2/OIDC configuration, the SSOService API, redirect-to-provider flow,
  callback processing, token validation, user provisioning on first login, and session management.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# CBSSO Skill

## When to Use This Skill

Load this skill when:
- Integrating ColdBox applications with enterprise SSO providers (Okta, Azure AD, Auth0, Google)
- Implementing SAML2 or OAuth2/OIDC authentication flows
- Processing SSO callbacks and provisioning users on first login
- Managing federated identity sessions alongside cbsecurity
- Handling SSO logout (single logout)

## Installation

```bash
box install cbsso
```

## Configuration

### config/modules/cbsso.cfc

```js
function configure() {
    return {
        // SSO provider type: "saml2" | "oauth2" | "oidc"
        provider : "oauth2",

        oauth2 : {
            clientId     : getSystemSetting( "SSO_CLIENT_ID",     "" ),
            clientSecret : getSystemSetting( "SSO_CLIENT_SECRET", "" ),
            authURL      : getSystemSetting( "SSO_AUTH_URL",      "" ),
            tokenURL     : getSystemSetting( "SSO_TOKEN_URL",     "" ),
            userInfoURL  : getSystemSetting( "SSO_USERINFO_URL",  "" ),
            redirectURI  : getSystemSetting( "SSO_REDIRECT_URI",  "" ),
            scopes       : "openid profile email",
            pkce         : true     // enable PKCE for public clients
        },

        // Where to send users after login
        successRedirect : "dashboard",

        // Where to send on SSO failure
        failureRedirect : "security.login"
    }
}
```

### SAML2 Provider Configuration

```js
saml2 : {
    idpEntityId      : getSystemSetting( "SAML_IDP_ENTITY_ID" ),
    idpSSOURL        : getSystemSetting( "SAML_IDP_SSO_URL" ),
    idpCertificate   : getSystemSetting( "SAML_IDP_CERT" ),
    spEntityId       : getSystemSetting( "SAML_SP_ENTITY_ID" ),
    spACSURL         : getSystemSetting( "SAML_SP_ACS_URL" ),
    spPrivateKey     : getSystemSetting( "SAML_SP_PRIVATE_KEY" ),
    spCertificate    : getSystemSetting( "SAML_SP_CERT" ),
    nameIdFormat     : "urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress",
    signRequests     : true,
    wantAssertionsSigned : true
}
```

## Core API

### Injection

```js
property name="ssoService" inject="SSOService@cbsso";
```

### Initiate SSO Login

```js
// Redirect to identity provider
function ssoLogin( event, rc, prc ) {
    var redirectURL = ssoService.getAuthorizationURL( {
        state : createUUID()   // store in session for CSRF check
    } )

    sessionStorage.set( "ssoState", state )
    relocate( redirectURL )
}
```

### Process Callback

```js
function callback( event, rc, prc ) {
    try {
        // Validate state to prevent CSRF
        if ( rc.state != sessionStorage.get( "ssoState" ) ) {
            throw( type = "CSRFException", message = "Invalid SSO state" )
        }
        sessionStorage.delete( "ssoState" )

        // Exchange code for tokens and get user claims
        var claims = ssoService.processCallback( rc.code ?: "", rc.state ?: "" )

        // Provision or retrieve user
        var user = provisionUser( claims )

        // Log user in via cbauth/cbsecurity
        authService.login( user )

        messagebox.success( "Signed in successfully via SSO." )
        relocate( "dashboard" )

    } catch ( CSRFException e ) {
        messagebox.error( "Invalid SSO request." )
        relocate( "security.login" )

    } catch ( SSOException e ) {
        log.error( "SSO callback failed: #e.message#" )
        messagebox.error( "SSO authentication failed. Please try again." )
        relocate( "security.login" )
    }
}
```

### User Provisioning (JIT Provisioning)

```js
private function provisionUser( claims ) {
    // Try to find existing user by SSO subject or email
    var user = userService.findByEmail( claims.email ?: "" )

    if ( isNull( user ) ) {
        // Just-in-time provisioning — create user on first SSO login
        user = userService.create( {
            email       : claims.email,
            firstName   : claims.given_name  ?: "",
            lastName    : claims.family_name ?: "",
            ssoProvider : "okta",
            ssoSubject  : claims.sub,
            role        : mapSSOGroupsToRole( claims.groups ?: [] ),
            isActive    : true
        } )
    } else {
        // Update attributes that may have changed in the IdP
        user.update( {
            firstName : claims.given_name  ?: user.getFirstName(),
            lastName  : claims.family_name ?: user.getLastName()
        } )
    }

    return user
}
```

### SSO Logout (Single Logout)

```js
function logout( event, rc, prc ) {
    authService.logout()

    // Initiate SSO single logout if provider supports it
    if ( ssoService.supportsLogout() ) {
        relocate( ssoService.getLogoutURL() )
    } else {
        relocate( "security.login" )
    }
}
```

## Best Practices

- **Store SSO credentials in environment variables** — never commit `clientSecret` or private keys
- **Validate OAuth state parameter** on every callback — prevents CSRF attacks against the OAuth flow
- **Enable PKCE** for OAuth2 flows — mitigates authorization code interception attacks
- **Use HTTPS for all redirect URIs** — IdPs reject HTTP callback URLs in production
- **Validate all received claims** — do not trust unverified `email` or `sub` from untrusted sources
- **Provision users conservatively** — only assign elevated roles after verifying group membership claims
- **Store SSO subject alongside email** — allows login after an email address change on the IdP
- **Implement single logout** where the provider supports it — prevents stale sessions on shared devices

## Documentation

- cbsso: https://github.com/coldbox-modules/cbsso
- OAuth 2.0 RFC: https://datatracker.ietf.org/doc/html/rfc6749
- SAML2 overview: https://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0.html
