---
name: bcrypt
description: >
  Use this skill when working with BCrypt password hashing in ColdBox/BoxLang applications. Covers
  installation, work-factor configuration, hashing passwords, verifying credentials, generating salts,
  mixin helpers in handlers/interceptors, user registration, authentication, password change/reset, and
  security best practices for credential handling.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# BCrypt Skill

## When to Use This Skill

Load this skill when:
- Hashing passwords before storing in the database
- Verifying user credentials during authentication
- Implementing user registration or password change flows
- Configuring work factor / cost factor for security tuning
- Using BCrypt mixin helpers in handlers and views
- Migrating plain-text or MD5/SHA passwords to BCrypt

## Language Mode Reference

Examples use **BoxLang (`.bx`)** syntax. Adapt as needed:

| Concept | BoxLang (`.bx`) | CFML (`.cfc`) |
|---------|-----------------|---------------|
| Class declaration | `class {` | `component {` |
| DI annotation | `@inject` above property | `property name="x" inject="y";` |

## Installation & Configuration

```bash
box install bcrypt
```

```js
// config/ColdBox.cfc — optional, default workFactor = 12
moduleSettings = {
    bcrypt = {
        workFactor = 12  // range 4–31; higher = slower but more secure
    }
}
```

> **Security note**: BCrypt cost factor 12 takes ~250 ms per hash on modern hardware. Increase to 14–15 for high-value accounts. Never go below 10 in production.

## Core API

### Injection

```js
// BoxLang / CFML script
property name="bcrypt" inject="@bcrypt";
```

Mixin helpers are available automatically in ColdBox handlers, interceptors, layouts, and views:
- `bcryptHash( password [, workFactor] )`
- `bcryptCheck( candidate, hash )`
- `bcryptSalt( [workFactor] )`

### Hash a Password

```js
// Default work factor
var hash = bcrypt.hashPassword( "SecurePass123!" )

// Custom work factor
var hash = bcrypt.hashPassword(
    password   = "SecurePass123!",
    workFactor = 14
)
// Result: $2a$14$... (60-character BCrypt hash)
```

### Verify a Password

```js
var isValid = bcrypt.checkPassword(
    candidate  = plaintextPassword,
    bCryptHash = storedHash
)
// Always compare in constant time — never compare hashes directly with ==
```

### Generate Salt

```js
var salt = bcrypt.generateSalt( 12 )  // explicit work factor
```

## Production Patterns

### User Registration

```js
class UserService {
    @inject
    property name="userDAO";
    @inject("@bcrypt")
    property name="bcrypt";

    function register( required struct data ) {
        // NEVER store plaintext passwords
        data.password = bcrypt.hashPassword( data.password )
        return userDAO.create( data )
    }
}
```

### Authentication

```js
class AuthService {
    @inject
    property name="userDAO";
    @inject("@bcrypt")
    property name="bcrypt";

    function authenticate( required string username, required string password ) {
        var user = userDAO.findByUsername( username )

        // Use same error message for missing vs wrong password — no user enumeration
        if ( isNull( user ) || !bcrypt.checkPassword( password, user.getPassword() ) ) {
            throw( type = "InvalidCredentials", message = "Invalid username or password" )
        }

        return user
    }
}
```

### Password Change

```js
function changePassword(
    required numeric userId,
    required string  currentPassword,
    required string  newPassword
) {
    var user = userDAO.findOrFail( userId )

    if ( !bcrypt.checkPassword( currentPassword, user.getPassword() ) ) {
        throw( type = "InvalidPassword", message = "Current password is incorrect" )
    }

    user.setPassword( bcrypt.hashPassword( newPassword ) )
    user.save()
}
```

### Password Reset (Token-Based)

```js
function resetPassword( required string token, required string newPassword ) {
    var record = tokenDAO.findValidToken( token )

    if ( isNull( record ) || record.getExpiresAt() < now() ) {
        throw( type = "InvalidToken", message = "Reset token is invalid or expired" )
    }

    var user = userDAO.find( record.getUserId() )
    user.setPassword( bcrypt.hashPassword( newPassword ) )
    user.save()

    tokenDAO.invalidate( token )  // one-time use
}
```

## Best Practices

- **Never store or log plaintext passwords** — only store the BCrypt hash
- **Use work factor 12+** in production; re-evaluate every 1–2 years
- **Avoid user enumeration** — return the same error for wrong username AND wrong password
- **Rate-limit login attempts** — BCrypt slows individual hashes but not brute-force at scale
- **Upgrade legacy hashes on login** — if a user logs in with an old MD5 hash, re-hash with BCrypt before saving
- **Do not use `==` to compare hashes** — always use `bcrypt.checkPassword()`

## Documentation

- BCrypt module: https://github.com/coldbox-modules/bcrypt
