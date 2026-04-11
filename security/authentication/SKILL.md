---
name: coldbox-security-authentication
description: "Use this skill when implementing user authentication in ColdBox with CBAuth, creating user services with retrieveUserById/retrieveUserByUsername, hashing passwords with BCrypt, managing login/logout sessions, implementing remember me functionality, or setting up the CBAuth module configuration."
---

# Authentication Patterns in ColdBox

## Overview

CBAuth provides user authentication for ColdBox applications with session management, password hashing (BCrypt), remember me functionality, and user session storage.

## Installation

```bash
box install cbauth
box install bcrypt
```

## Configuration

```boxlang
// config/ColdBox.cfc
moduleSettings = {
    cbauth: {
        userServiceClass: "UserService@models",
        userModel: "User@models",
        rememberMe: {
            enabled: true,
            cookieName: "remember_me",
            days: 30
        }
    }
}
```

## User Service (Required by CBAuth)

```boxlang
/**
 * models/UserService.cfc
 * Must implement three CBAuth-required methods
 */
class singleton {

    property name="bcrypt" inject="@BCrypt"

    // Required: Retrieve user by ID
    function retrieveUserById( required id ) {
        return queryExecute(
            "SELECT * FROM users WHERE id = :id",
            { id: arguments.id }
        ).reduce( ( acc, row ) => row )
    }

    // Required: Retrieve user by username/email
    function retrieveUserByUsername( required username ) {
        return queryExecute(
            "SELECT * FROM users WHERE email = :email",
            { email: arguments.username }
        ).reduce( ( acc, row ) => row )
    }

    // Required: Validate credentials
    function isValidCredentials( required username, required password ) {
        try {
            var user = retrieveUserByUsername( arguments.username )
            return bcrypt.checkPassword( arguments.password, user.password )
        } catch ( any e ) {
            return false
        }
    }

    function create( required data ) {
        var hashedPwd = bcrypt.hashPassword( arguments.data.password )

        var qry = queryExecute(
            "INSERT INTO users (name, email, password) VALUES (:name, :email, :pwd)",
            {
                name:  arguments.data.name,
                email: arguments.data.email,
                pwd:   hashedPwd
            }
        )

        return retrieveUserById( qry.generatedKey )
    }
}
```

## Login Handler

```boxlang
/**
 * handlers/Auth.cfc
 */
class extends="coldbox.system.EventHandler" {

    property name="auth" inject="authenticationService@cbauth"

    // GET /login
    function login( event, rc, prc ) {
        if ( auth.isLoggedIn() ) {
            relocate( "main.index" )
        }
        event.setView( "auth/login" )
    }

    // POST /login
    function doLogin( event, rc, prc ) {
        event.paramValue( "email",       "" )
        event.paramValue( "password",    "" )
        event.paramValue( "rememberMe",  false )

        try {
            auth.authenticate( rc.email, rc.password )

            if ( rc.rememberMe ) {
                auth.setRememberMe( 30 )
            }

            flash.put( "success", "Welcome back!" )
            relocate( "main.index" )

        } catch ( InvalidCredentials e ) {
            flash.put( "error", "Invalid email or password" )
            relocate( "auth.login" )
        }
    }

    // POST /logout
    function doLogout( event, rc, prc ) {
        auth.logout()
        flash.put( "info", "You have been logged out" )
        relocate( "auth.login" )
    }

    // GET /register
    function register( event, rc, prc ) {
        event.setView( "auth/register" )
    }

    // POST /register
    function doRegister( event, rc, prc ) {
        event.paramValue( "name",     "" )
        event.paramValue( "email",    "" )
        event.paramValue( "password", "" )

        userService.create( {
            name:     rc.name,
            email:    rc.email,
            password: rc.password
        } )

        // Auto-login after registration
        auth.authenticate( rc.email, rc.password )

        relocate( "main.index" )
    }
}
```

## Protecting Routes (CBSecurity)

```boxlang
// config/ColdBox.cfc — protect routes requiring auth
moduleSettings = {
    cbsecurity: {
        firewall: {
            enabled: true,
            defaultAction: "redirect",
            defaultRedirect: "/login"
        },
        rules: [
            // Public routes
            {
                whitelist: "auth\\..*,main\\.index",
                match: "event"
            },
            // Protected dashboard
            {
                secureList: "^dashboard\\.",
                match: "event",
                action: "redirect",
                redirect: "/login"
            }
        ]
    }
}
```

## Accessing the Logged-In User

```boxlang
class extends="coldbox.system.EventHandler" {

    property name="auth" inject="authenticationService@cbauth"

    function dashboard( event, rc, prc ) {
        // Check authentication
        if ( !auth.isLoggedIn() ) {
            relocate( "auth.login" )
        }

        // Get current user
        prc.user = auth.getUser()

        event.setView( "dashboard/index" )
    }
}
```

## Remember Me View Integration

```html
<!-- views/auth/login.cfm -->
<form action="#event.buildLink( 'auth.doLogin' )#" method="post">
    #csrf()#
    <input type="email" name="email" required />
    <input type="password" name="password" required />
    <label>
        <input type="checkbox" name="rememberMe" value="true" /> Remember me
    </label>
    <button type="submit">Login</button>
</form>
```

## CBAuth Methods Quick Reference

| Method | Description |
|--------|-------------|
| `auth.authenticate( username, password )` | Validate and log in user |
| `auth.isLoggedIn()` | Check if user is authenticated |
| `auth.getUser()` | Get the current authenticated user |
| `auth.logout()` | Log out current user |
| `auth.setRememberMe( days )` | Enable remember me cookie |
| `auth.getUserId()` | Get authenticated user's ID |
