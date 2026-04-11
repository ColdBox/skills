---
name: coldbox-security-passkeys
description: "Use this skill when implementing passkeys (WebAuthn/FIDO2) passwordless authentication in ColdBox with cbsecurity-passkeys, configuring relying party settings, building passkey registration and authentication flows, managing passkey device storage, or adding biometric and hardware security key login support."
---

# Passkeys (WebAuthn/FIDO2) in ColdBox

## Overview

Passkeys provide phishing-resistant passwordless authentication using public-key cryptography. The browser/OS manages private keys (stored in secure enclave or hardware token); the server stores only public keys. ColdBox uses `cbsecurity-passkeys` to implement the WebAuthn ceremony.

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
install cbsecurity
install cbsecurity-passkeys
```

## Module Configuration

```boxlang
// config/ColdBox.cfc
moduleSettings = {
    "cbsecurity-passkeys": {
        rpName:                  "My Application",      // Shown during registration prompt
        rpID:                    "app.example.com",     // Must match domain (no port)
        rpOrigin:                "https://app.example.com",
        attestation:             "none",                // none | direct | indirect
        authenticatorAttachment: "platform",            // platform (biometric) | cross-platform (USB key) | "" (both)
        userVerification:        "preferred",           // required | preferred | discouraged
        storageService:          "PasskeyStorageService"  // Your custom storage impl
    }
}
```

## Passkey Storage Service

```boxlang
/**
 * models/PasskeyStorageService.cfc
 * Implements the IPasskeyStorage interface from cbsecurity-passkeys.
 */
class singleton {

    // Store a newly registered passkey credential
    function storeCredential( required struct credential ) {
        queryExecute(
            "INSERT INTO passkeys
                (id, user_id, credential_id, public_key, sign_count, device_name, created_at)
             VALUES
                (:id, :userId, :credentialId, :publicKey, :signCount, :deviceName, NOW())",
            {
                id:           createUUID(),
                userId:       arguments.credential.userID,
                credentialId: arguments.credential.id,
                publicKey:    arguments.credential.publicKey,
                signCount:    arguments.credential.signCount ?: 0,
                deviceName:   arguments.credential.deviceName ?: "Unknown Device"
            }
        )
    }

    // Retrieve a credential by its WebAuthn credential ID
    function getCredential( required credentialId ) {
        var qry = queryExecute(
            "SELECT * FROM passkeys WHERE credential_id = :id",
            { id: arguments.credentialId }
        )
        if ( !qry.recordCount ) return javaCast( "null", "" )
        return queryRowToStruct( qry, 1 )
    }

    // Update sign count after successful authentication (replay attack prevention)
    function updateSignCount( required credentialId, required signCount ) {
        queryExecute(
            "UPDATE passkeys SET sign_count = :count, last_used = NOW()
             WHERE credential_id = :id",
            { count: arguments.signCount, id: arguments.credentialId }
        )
    }

    // List all passkeys for a given user
    function getCredentialsForUser( required userID ) {
        return queryExecute(
            "SELECT * FROM passkeys WHERE user_id = :userId AND revoked = 0
             ORDER BY created_at DESC",
            { userId: arguments.userID }
        )
    }

    // Revoke a specific passkey
    function revokeCredential( required credentialId ) {
        queryExecute(
            "UPDATE passkeys SET revoked = 1 WHERE credential_id = :id",
            { id: arguments.credentialId }
        )
    }
}
```

**CFML (`.cfc`):**

```cfml
/**
 * models/PasskeyStorageService.cfc
 * Implements the IPasskeyStorage interface from cbsecurity-passkeys.
 */
component {

    // Store a newly registered passkey credential
    function storeCredential( required struct credential ) {
        queryExecute(
            "INSERT INTO passkeys
                (id, user_id, credential_id, public_key, sign_count, device_name, created_at)
             VALUES
                (:id, :userId, :credentialId, :publicKey, :signCount, :deviceName, NOW())",
            {
                id:           createUUID(),
                userId:       arguments.credential.userID,
                credentialId: arguments.credential.id,
                publicKey:    arguments.credential.publicKey,
                signCount:    arguments.credential.signCount ?: 0,
                deviceName:   arguments.credential.deviceName ?: "Unknown Device"
            }
        )
    }

    // Retrieve a credential by its WebAuthn credential ID
    function getCredential( required credentialId ) {
        var qry = queryExecute(
            "SELECT * FROM passkeys WHERE credential_id = :id",
            { id: arguments.credentialId }
        )
        if ( !qry.recordCount ) return javaCast( "null", "" )
        return queryRowToStruct( qry, 1 )
    }

    // Update sign count after successful authentication (replay attack prevention)
    function updateSignCount( required credentialId, required signCount ) {
        queryExecute(
            "UPDATE passkeys SET sign_count = :count, last_used = NOW()
             WHERE credential_id = :id",
            { count: arguments.signCount, id: arguments.credentialId }
        )
    }

    // List all passkeys for a given user
    function getCredentialsForUser( required userID ) {
        return queryExecute(
            "SELECT * FROM passkeys WHERE user_id = :userId AND revoked = 0
             ORDER BY created_at DESC",
            { userId: arguments.userID }
        )
    }

    // Revoke a specific passkey
    function revokeCredential( required credentialId ) {
        queryExecute(
            "UPDATE passkeys SET revoked = 1 WHERE credential_id = :id",
            { id: arguments.credentialId }
        )
    }
}
```

## Passkeys Handler

```boxlang
/**
 * handlers/Passkeys.cfc
 */
class extends="coldbox.system.EventHandler" {

    property name="passkeyService" inject="@cbsecurity-passkeys"
    property name="cbauth"         inject="@cbauth"
    property name="userService"    inject="UserService"

    // ─── Registration ───────────────────────────────────────────────────────

    /**
     * GET /passkeys/register/options
     * Returns challenge options for the browser's navigator.credentials.create()
     */
    function registrationOptions( event, rc, prc ) {
        secured()

        var options = passkeyService.generateRegistrationOptions(
            user: prc.currentUser
        )
        return event.renderData( type = "json", data = options )
    }

    /**
     * POST /passkeys/register/verify
     * Accepts the browser's credential and stores the passkey.
     */
    function registrationVerify( event, rc, prc ) {
        secured()

        try {
            passkeyService.verifyRegistration(
                user:       prc.currentUser,
                credential: rc.credential  // JSON from browser
            )
            return event.renderData( type = "json", data = { success: true } )
        } catch ( any e ) {
            return event.renderData(
                type       = "json",
                data       = { success: false, error: e.message },
                statusCode = 400
            )
        }
    }

    // ─── Authentication ──────────────────────────────────────────────────────

    /**
     * POST /passkeys/auth/options
     * Returns the authentication challenge for navigator.credentials.get()
     * Body may optionally contain { email } to filter by user.
     */
    function authOptions( event, rc, prc ) {
        var options = passkeyService.generateAuthenticationOptions(
            email: rc.email ?: ""
        )
        return event.renderData( type = "json", data = options )
    }

    /**
     * POST /passkeys/auth/verify
     * Verifies the signed assertion from the browser; logs user in.
     */
    function authVerify( event, rc, prc ) {
        try {
            var result = passkeyService.verifyAuthentication( rc.credential )
            var user   = userService.findById( result.userID )

            cbauth.login( user )

            return event.renderData( type = "json", data = {
                success:  true,
                redirect: event.buildLink( "dashboard" )
            } )
        } catch ( any e ) {
            return event.renderData(
                type       = "json",
                data       = { success: false, error: "Authentication failed" },
                statusCode = 401
            )
        }
    }

    // ─── Device Management ───────────────────────────────────────────────────

    /**
     * GET /passkeys
     * List current user's registered passkeys.
     */
    function list( event, rc, prc ) {
        secured()
        var keys = passkeyService.listForUser( prc.currentUser.getID() )
        return event.renderData( type = "json", data = keys )
    }

    /**
     * DELETE /passkeys/:credentialID
     * Revoke a specific passkey.
     */
    function revoke( event, rc, prc ) {
        secured()
        passkeyService.revokeCredential( rc.credentialID )
        return event.renderData( type = "json", data = { message: "Passkey removed" } )
    }
}
```

**CFML (`.cfc`):**

```cfml
/**
 * handlers/Passkeys.cfc
 */
component extends="coldbox.system.EventHandler" {

    property name="passkeyService" inject="@cbsecurity-passkeys"
    property name="cbauth"         inject="@cbauth"
    property name="userService"    inject="UserService"

    // ─── Registration ───────────────────────────────────────────────────────

    /**
     * GET /passkeys/register/options
     * Returns challenge options for the browser's navigator.credentials.create()
     */
    function registrationOptions( event, rc, prc ) {
        secured()

        var options = passkeyService.generateRegistrationOptions(
            user: prc.currentUser
        )
        return event.renderData( type = "json", data = options )
    }

    /**
     * POST /passkeys/register/verify
     * Accepts the browser's credential and stores the passkey.
     */
    function registrationVerify( event, rc, prc ) {
        secured()

        try {
            passkeyService.verifyRegistration(
                user:       prc.currentUser,
                credential: rc.credential  // JSON from browser
            )
            return event.renderData( type = "json", data = { success: true } )
        } catch ( any e ) {
            return event.renderData(
                type       = "json",
                data       = { success: false, error: e.message },
                statusCode = 400
            )
        }
    }

    // ─── Authentication ──────────────────────────────────────────────────────

    /**
     * POST /passkeys/auth/options
     * Returns the authentication challenge for navigator.credentials.get()
     * Body may optionally contain { email } to filter by user.
     */
    function authOptions( event, rc, prc ) {
        var options = passkeyService.generateAuthenticationOptions(
            email: rc.email ?: ""
        )
        return event.renderData( type = "json", data = options )
    }

    /**
     * POST /passkeys/auth/verify
     * Verifies the signed assertion from the browser; logs user in.
     */
    function authVerify( event, rc, prc ) {
        try {
            var result = passkeyService.verifyAuthentication( rc.credential )
            var user   = userService.findById( result.userID )

            cbauth.login( user )

            return event.renderData( type = "json", data = {
                success:  true,
                redirect: event.buildLink( "dashboard" )
            } )
        } catch ( any e ) {
            return event.renderData(
                type       = "json",
                data       = { success: false, error: "Authentication failed" },
                statusCode = 401
            )
        }
    }

    // ─── Device Management ───────────────────────────────────────────────────

    /**
     * GET /passkeys
     * List current user's registered passkeys.
     */
    function list( event, rc, prc ) {
        secured()
        var keys = passkeyService.listForUser( prc.currentUser.getID() )
        return event.renderData( type = "json", data = keys )
    }

    /**
     * DELETE /passkeys/:credentialID
     * Revoke a specific passkey.
     */
    function revoke( event, rc, prc ) {
        secured()
        passkeyService.revokeCredential( rc.credentialID )
        return event.renderData( type = "json", data = { message: "Passkey removed" } )
    }
}
```

## Route Configuration

```boxlang
// config/Router.cfc
route( "/passkeys/register/options" ).withAction( { GET: "registrationOptions" } ).to( "passkeys" )
route( "/passkeys/register/verify"  ).withAction( { POST: "registrationVerify" } ).to( "passkeys" )
route( "/passkeys/auth/options"     ).withAction( { POST: "authOptions" } ).to( "passkeys" )
route( "/passkeys/auth/verify"      ).withAction( { POST: "authVerify" } ).to( "passkeys" )
route( "/passkeys"                  ).withAction( { GET: "list" } ).to( "passkeys" )
route( "/passkeys/:credentialID"    ).withAction( { DELETE: "revoke" } ).to( "passkeys" )
```

## Browser JavaScript (Registration)

```javascript
// Register a new passkey
async function registerPasskey() {
    // 1. Get options from server
    const optRes  = await fetch( "/passkeys/register/options" )
    const options = await optRes.json()

    // 2. Create credential in browser
    options.challenge = base64urlDecode( options.challenge )
    options.user.id   = base64urlDecode( options.user.id )

    const credential = await navigator.credentials.create( { publicKey: options } )

    // 3. Send to server for verification
    const verifyRes = await fetch( "/passkeys/register/verify", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify( serializeCredential( credential ) )
    } )

    const { success } = await verifyRes.json()
    if ( success ) alert( "Passkey registered!" )
}
```

## Browser JavaScript (Authentication)

```javascript
// Authenticate with passkey
async function loginWithPasskey() {
    // 1. Get challenge from server (optional email for filtering)
    const optRes  = await fetch( "/passkeys/auth/options", {
        method: "POST",
        body: JSON.stringify( { email: emailInput.value } )
    } )
    const options = await optRes.json()

    // 2. Prompt user for biometric/key
    options.challenge = base64urlDecode( options.challenge )
    const assertion   = await navigator.credentials.get( { publicKey: options } )

    // 3. Verify on server
    const verifyRes = await fetch( "/passkeys/auth/verify", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify( serializeCredential( assertion ) )
    } )

    const { success, redirect } = await verifyRes.json()
    if ( success ) window.location.href = redirect
}
```

## WebAuthn Concepts Reference

| Term | Meaning |
|------|---------|
| Relying Party (RP) | Your web server/application |
| rpID | Domain of your app (no port) |
| Challenge | One-time random bytes to prevent replay attacks |
| Credential | Public key + credential ID stored on server |
| Authenticator | Device storing private key (Touch ID, Face ID, YubiKey) |
| Attestation | Proof of authenticator model (typically `none` for most apps) |
| Sign Count | Monotonic counter; increases each use; detects credential cloning |

## Security Checklist

- [ ] `rpID` matches the exact domain (no port, no subdomain unless intended)
- [ ] `rpOrigin` uses HTTPS in production
- [ ] Challenge is cryptographically random and discarded after single use
- [ ] Sign count validated and updated on every successful auth
- [ ] Revoked credentials cannot authenticate
- [ ] Users can list and revoke their own passkeys
- [ ] Passkeys complement (not replace) existing auth during rollout
