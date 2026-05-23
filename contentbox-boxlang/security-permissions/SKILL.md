---
name: contentbox-boxlang-security-permissions
description: "Use this skill when implementing ContentBox security and permissions, including roles, permission modeling, cbSecurity integration, authorization checks, CSRF/rate-limiting protections, and hardening patterns."
applyTo: "**/*.{bx,bxm,cfc,cfm,cfml}"
---

# ContentBox Security & Permissions (BoxLang)

Manage authentication, authorization, roles, permissions, and security rules in ContentBox CMS using BoxLang.

## Security Architecture

ContentBox uses **cbSecurity** for its security layer with a database-driven RBAC (Role-Based Access Control) model.

### Security Entities

| Entity | File | Description |
|--------|------|-------------|
| **Author** | `models/security/Author.cfc` | User entity with password, roles, preferences, 2FA |
| **Role** | `models/security/Role.cfc` | RBAC roles with M2M to permissions |
| **Permission** | `models/security/Permission.cfc` | Individual permissions |
| **PermissionGroup** | `models/security/PermissionGroup.cfc` | Permission grouping |
| **SecurityRule** | `models/security/SecurityRule.cfc` | Firewall rules (whitelist/securelist/roles/permissions) |
| **LoginAttempt** | `models/security/LoginAttempt.cfc` | Login attempt tracking |

### Security Services

| Service | File | Description |
|---------|------|-------------|
| **SecurityService** | `models/security/SecurityService.cfc` | Authentication, session, password reset, encryption |
| **AuthorService** | `models/security/AuthorService.cfc` | Author CRUD, preferences, avatar |
| **RoleService** | `models/security/RoleService.cfc` | Role management |
| **PermissionService** | `models/security/PermissionService.cfc` | Permission management |
| **SecurityRuleService** | `models/security/SecurityRuleService.cfc` | Security rules from DB |
| **LoginTrackerService** | `models/security/LoginTrackerService.cfc` | Login attempt tracking |
| **RateLimiter** | `models/security/RateLimiter.cfc` | Rate limiting interceptor |

## Authentication

### SecurityService Methods

```boxlang
property name="securityService" inject="securityService@contentbox"

// Authentication
securityService.login( author )           // Authenticate and set session
securityService.logout()                  // Clear session
securityService.isLoggedIn()              // Check auth status
securityService.getAuthorSession()        // Get current author from session

// Password management
securityService.generateResetToken( author )  // Generate password reset token
securityService.resetPassword( author, newPassword )  // Reset password

// Session management
securityService.getKeepMeLoggedIn()       // Remember-me cookie handling
securityService.updateAuthorLoginTimestamp( author )  // Update last login
```

### Author Entity

```boxlang
property name="authorService" inject="authorService@contentbox"

// Author properties
author.getAuthorID()
author.getUsername()
author.getEmail()
author.getFirstName()
author.getLastName()
author.getFullName()          // "FirstName LastName"
author.getBiography()
author.getIsActive()
author.getLastLogin()
author.getCreatedDate()
author.getModifiedDate()

// Roles and permissions
author.getRoles()             // Array of Role entities
author.hasRole( "Admin" )     // Check if author has role
author.hasPermission( "ENTRY_EDIT" )  // Check permission

// 2FA
author.getTwoFactorEnabled()
author.getTwoFactorSecret()
author.getTwoFactorProvider()

// Preferences
author.getPreferences()       // Struct of user preferences
author.getPreference( "key" ) // Get specific preference
```

## Roles and Permissions

### Creating Roles

```boxlang
property name="roleService" inject="roleService@contentbox"

// Create a role
role = roleService.new( {
	name        : "Editor",
	description : "Can edit and publish entries"
} )
roleService.save( role )

// Add permissions to role
permission = permissionService.findByPermission( "ENTRY_EDIT" )
role.addPermission( permission )
roleService.save( role )
```

### Creating Permissions

```boxlang
property name="permissionService" inject="permissionService@contentbox"

// Create a permission
permission = permissionService.new( {
	permission  : "MYMODULE_ACCESS",
	description : "Access to my custom module"
} )
permissionService.save( permission )
```

### Permission Groups

```boxlang
property name="permissionGroupService" inject="permissionGroupService@contentbox"

// Create a permission group
group = permissionGroupService.new( {
	name        : "My Module",
	description : "Permissions for my custom module"
} )
permissionGroupService.save( group )

// Assign permission to group
permission.setPermissionGroup( group )
```

## Security Rules

Security rules are stored in the `cb_securityRule` table and loaded by `securityRuleService@contentbox`:

| Field | Description |
|-------|-------------|
| `whitelist` | Events/URLs that don't require authentication |
| `securelist` | Events/URLs that require authentication |
| `roles` | Required roles (comma-separated) |
| `permissions` | Required permissions (comma-separated) |
| `redirect` | Redirect URL on failure |
| `overrideEvent` | Override event on failure |
| `useSSL` | Force SSL for these rules |
| `action` | Action to take (redirect/override) |

### Creating Security Rules

```boxlang
property name="securityRuleService" inject="securityRuleService@contentbox"

rule = securityRuleService.new( {
	whitelist   : "cbadmin/myModule.index,cbadmin/myModule.public",
	securelist  : "cbadmin/myModule.*",
	roles       : "Admin,Editor",
	permissions : "MYMODULE_ACCESS",
	redirect    : "cbadmin/security/login",
	useSSL      : false,
	action      : "redirect"
} )
securityRuleService.save( rule )
```

## Checking Permissions in Code

```boxlang
property name="securityService" inject="securityService@contentbox"

// Check if logged in
if( securityService.isLoggedIn() ){
	author = securityService.getAuthorSession()
}

// Check role
if( author.hasRole( "Admin" ) ){
	// Admin-only logic
}

// Check permission
if( author.hasPermission( "ENTRY_EDIT" ) ){
	// Can edit entries
}

// Check multiple permissions
if( author.hasPermission( "ENTRY_EDIT,ENTRY_PUBLISH" ) ){
	// Has both permissions
}
```

## Password Security

- Passwords are hashed using **BCrypt** via `BCrypt@BCrypt`
- Password reset tokens are generated with `generateResetToken()`
- Login attempts are tracked via `LoginTrackerService`

## Rate Limiting

The `RateLimiter@contentbox` interceptor protects against brute-force attacks:

```boxlang
// Registered in core ModuleConfig.bx
interceptors = [
	{
		class : "contentbox.models.security.RateLimiter",
		name  : "RateLimiter@contentbox"
	}
]
```

## CSRF Protection

```boxlang
property name="cbcsrf" inject="cbcsrf@cbcsrf"

// Generate CSRF token
token = cbcsrf.getToken( "formName" )

// Verify CSRF token
if( cbcsrf.verify( rc.csrfToken ) ){
	// Valid token
}
```

## Best Practices

1. **Use cbSecurity rules** — define rules in the database, not in code
2. **Check permissions** — use `hasPermission()` before sensitive operations
3. **Use roles for grouping** — assign permissions to roles, roles to authors
4. **Track login attempts** — use `LoginTrackerService` for audit trails
5. **Enable 2FA** — enforce two-factor authentication for admin users
6. **Use BCrypt** — never store plain-text passwords
7. **Implement rate limiting** — protect login endpoints
8. **Use CSRF tokens** — protect all form submissions
9. **Log security events** — use `log.info()` for audit trails
10. **Test with different roles** — verify access control for each role

## Engine Compatibility

This skill targets **BoxLang** engine. For CFML-specific syntax (Lucee 5+, Adobe ColdFusion 2018+), see the CFML variant of this skill.

Key BoxLang advantages:
- Cleaner script syntax without `<cfcomponent>` / `<cffunction>` tags
- No parentheses needed for zero-argument function calls
- `#{...}#` for inline expression output in `.bx` templates
- Modern syntax: `?:` null coalescing, `?.` safe navigation
- Native support for modern data structures
