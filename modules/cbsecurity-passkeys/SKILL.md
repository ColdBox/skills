---
name: cbsecurity-passkeys
description: >
  Use this skill when implementing WebAuthn/Passkeys authentication in ColdBox/BoxLang using the
  cbsecurity-passkeys module. Covers the credential repository interface, registration ceremony,
  authentication ceremony, JavaScript integration, and production hardening for passwordless login.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# CBSecurity Passkeys Skill

## When to Use This Skill

Load this skill when:
- Adding WebAuthn / Passkeys (passwordless) authentication to a ColdBox application
- Implementing the `ICredentialRepository` interface for custom credential storage
- Building the registration and authentication ceremony endpoints
- Connecting the WebAuthn JS client to ColdBox handlers
- Hardening passkey flows for production security

## Installation

```bash
box install cbsecurity-passkeys
```

Requires cbsecurity to be installed and configured:

```bash
box install cbsecurity
```

## Configuration

### config/modules/cbsecurity-passkeys.cfc

```js
function configure() {
    return {
        // WireBox ID of your ICredentialRepository implementation
        credentialRepository : "PasskeyCredentialRepository",

        // Relying Party name shown to users during registration
        rpName : "My Application",

        // Relying Party ID — must match the effective domain of your app
        rpId   : getSystemSetting( "APP_DOMAIN", "localhost" ),

        // Allowed origins (must include https:// + rpId)
        origins : [
            "https://" & getSystemSetting( "APP_DOMAIN", "localhost" )
        ],

        // Timeout for ceremony challenges (milliseconds)
        timeout : 60000
    }
}
```

## ICredentialRepository Interface

```js
// models/PasskeyCredentialRepository.bx
class implements="cbsecurity.modules.cbsecurity-passkeys.interfaces.ICredentialRepository" {

    property name="qb" inject="QueryBuilder@qb";

    /**
     * Find all credentials for a given user handle (user ID)
     */
    array function findByUserHandle( userHandle ) {
        return qb.from( "passkey_credentials" )
            .where( "userId", userHandle )
            .get()
    }

    /**
     * Find a single credential by its credential ID (base64url)
     */
    struct function findById( credentialId ) {
        return qb.from( "passkey_credentials" )
            .where( "credentialId", credentialId )
            .first()
    }

    /**
     * Persist a newly registered credential
     */
    void function save( credential ) {
        qb.table( "passkey_credentials" ).insert( {
            id                   : createUUID(),
            userId               : credential.userHandle,
            credentialId         : credential.credentialId,
            publicKey            : credential.publicKey,
            signCount            : credential.signCount,
            transports           : serializeJSON( credential.transports ?: [] ),
            aaguid               : credential.aaguid ?: "",
            createdAt            : now()
        } )
    }

    /**
     * Update the sign counter after successful authentication
     */
    void function updateSignCount( credentialId, signCount ) {
        qb.table( "passkey_credentials" )
            .where( "credentialId", credentialId )
            .update( { signCount: signCount, lastUsedAt: now() } )
    }

    /**
     * Delete a credential
     */
    void function delete( credentialId ) {
        qb.table( "passkey_credentials" )
            .where( "credentialId", credentialId )
            .delete()
    }
}
```

### Database Migration for Credentials

```js
function up( schema ) {
    schema.create( "passkey_credentials", ( table ) => {
        table.uuid( "id" ).primary()
        table.string( "userId",       36 )
        table.string( "credentialId", 1000 )
        table.longText( "publicKey" )
        table.bigInteger( "signCount" ).default( 0 )
        table.string( "transports", 500 ).nullable()
        table.string( "aaguid", 36 ).nullable()
        table.dateTime( "createdAt" )
        table.dateTime( "lastUsedAt" ).nullable()
        table.index( "userId" )
        table.unique( "credentialId" )
    } )
}
```

## Registration Ceremony

```js
// handlers/Passkeys.bx
@secured
class {

    property name="passkeyService" inject="PasskeyService@cbsecurity-passkeys";
    property name="security"       inject="SecurityService@cbsecurity";

    // Step 1: Generate registration options and store challenge in session
    function registrationOptions( event, rc, prc ) {
        var user    = security.getCurrentUser()
        var options = passkeyService.generateRegistrationOptions( {
            userId      : user.getId(),
            username    : user.getEmail(),
            displayName : user.getFullName()
        } )

        // Store challenge server-side for verification
        sessionStorage.set( "passkeyChallenge", options.challenge )

        event.renderData( type = "json", data = options )
    }

    // Step 2: Verify and persist the credential
    function registerCredential( event, rc, prc ) {
        try {
            var challenge = sessionStorage.get( "passkeyChallenge" )
            sessionStorage.delete( "passkeyChallenge" )

            passkeyService.verifyRegistration(
                credential : rc.credential,
                challenge  : challenge,
                userHandle : security.getCurrentUser().getId()
            )

            event.renderData( type = "json", data = { success: true } )

        } catch ( any e ) {
            event.renderData(
                type       = "json",
                statusCode = 400,
                data       = { error: "Registration failed: #e.message#" }
            )
        }
    }
}
```

## Authentication Ceremony

```js
// handlers/Auth.bx
@unsecured
class {

    property name="passkeyService" inject="PasskeyService@cbsecurity-passkeys";
    property name="jwtService"     inject="JwtService@cbsecurity";

    // Step 1: Generate authentication options
    function authOptions( event, rc, prc ) {
        var options = passkeyService.generateAuthenticationOptions()
        sessionStorage.set( "passkeyChallenge", options.challenge )
        event.renderData( type = "json", data = options )
    }

    // Step 2: Verify assertion and issue token
    function authenticate( event, rc, prc ) {
        try {
            var challenge = sessionStorage.get( "passkeyChallenge" )
            sessionStorage.delete( "passkeyChallenge" )

            var result = passkeyService.verifyAuthentication(
                credential : rc.credential,
                challenge  : challenge
            )

            // Issue JWT for the verified user
            var token = jwtService.fromUser( userService.get( result.userHandle ) )

            event.renderData( type = "json", data = { token: token } )

        } catch ( any e ) {
            event.renderData(
                type       = "json",
                statusCode = 401,
                data       = { error: "Authentication failed" }
            )
        }
    }
}
```

## Best Practices

- **Always use HTTPS** — WebAuthn requires a secure context; passkeys will not work over HTTP
- **Set `rpId` to the effective domain only** — do not include protocol or port; must match browser origin
- **Delete the challenge from session immediately** after use — prevents replay attacks
- **Validate `signCount`** on each use — a decreasing count indicates a cloned authenticator
- **Never log credential material** — specifically `publicKey` and raw assertion bytes
- **Offer passkeys as an additional method** alongside passwords (progressive rollout)
- **Verify all ceremony options** — never skip verifying origin, rpId, and challenge on the server
- **Return generic error messages** on failure — avoid leaking which step failed

## Documentation

- cbsecurity-passkeys: https://github.com/coldbox-modules/cbsecurity-passkeys
- WebAuthn spec: https://www.w3.org/TR/webauthn/
- Passkeys overview: https://passkeys.dev
