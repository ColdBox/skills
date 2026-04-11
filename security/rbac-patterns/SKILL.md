---
name: coldbox-security-rbac
description: "Use this skill when implementing Role-Based Access Control (RBAC) in ColdBox, creating role and permission management services, building hierarchical role structures, assigning roles and permissions to users, checking permissions in handlers with cbsecurity, or when designing a group-based access control system."
---

# RBAC Patterns in ColdBox

## Overview

Role-Based Access Control (RBAC) assigns system access based on user roles and permissions. Roles group related permissions; users are assigned roles. CBSecurity enforces RBAC rules throughout the application.

## Database Schema

```sql
-- roles table
CREATE TABLE roles (
    id          VARCHAR(36) PRIMARY KEY,
    name        VARCHAR(50) UNIQUE NOT NULL,
    description VARCHAR(255),
    parent_id   VARCHAR(36),
    created_at  DATETIME
);

-- permissions table
CREATE TABLE permissions (
    id          VARCHAR(36) PRIMARY KEY,
    name        VARCHAR(100) UNIQUE NOT NULL,  -- e.g. "users.create"
    description VARCHAR(255)
);

-- role_permissions pivot
CREATE TABLE role_permissions (
    role_id    VARCHAR(36),
    permission VARCHAR(100),
    PRIMARY KEY (role_id, permission)
);

-- user_roles pivot
CREATE TABLE user_roles (
    user_id VARCHAR(36),
    role_id VARCHAR(36),
    PRIMARY KEY (user_id, role_id)
);
```

## Role Service

```boxlang
/**
 * models/RoleService.cfc
 */
class singleton {

    function create( required name, description = "", permissions = [] ) {
        var roleID = createUUID()

        queryExecute(
            "INSERT INTO roles (id, name, description) VALUES (:id, :name, :desc)",
            { id: roleID, name: arguments.name, desc: arguments.description }
        )

        if ( !arguments.permissions.isEmpty() ) {
            assignPermissions( roleID, arguments.permissions )
        }

        return roleID
    }

    function assignPermissions( required roleID, required array permissions ) {
        // Clear existing
        queryExecute(
            "DELETE FROM role_permissions WHERE role_id = :id",
            { id: arguments.roleID }
        )

        // Assign new
        arguments.permissions.each( ( perm ) => {
            queryExecute(
                "INSERT INTO role_permissions (role_id, permission) VALUES (:roleID, :perm)",
                { roleID: roleID, perm: perm }
            )
        } )
    }

    function getPermissions( required roleID ) {
        var qry = queryExecute(
            "SELECT permission FROM role_permissions WHERE role_id = :id",
            { id: arguments.roleID }
        )
        return listToArray( valueList( qry.permission ) )
    }

    function findAll() {
        return queryExecute( "SELECT * FROM roles ORDER BY name" )
    }
}
```

## User Role Assignment

```boxlang
/**
 * models/UserService.cfc — role assignment methods
 */
class singleton {

    function assignRole( required userID, required roleID ) {
        queryExecute(
            "INSERT IGNORE INTO user_roles (user_id, role_id) VALUES (:userId, :roleId)",
            { userId: arguments.userID, roleId: arguments.roleID }
        )
    }

    function removeRole( required userID, required roleID ) {
        queryExecute(
            "DELETE FROM user_roles WHERE user_id = :userId AND role_id = :roleId",
            { userId: arguments.userID, roleId: arguments.roleID }
        )
    }

    function getUserRoles( required userID ) {
        var qry = queryExecute(
            "SELECT r.name FROM roles r
             JOIN user_roles ur ON ur.role_id = r.id
             WHERE ur.user_id = :userId",
            { userId: arguments.userID }
        )
        return listToArray( valueList( qry.name ) )
    }

    function getUserPermissions( required userID ) {
        var qry = queryExecute(
            "SELECT DISTINCT rp.permission FROM role_permissions rp
             JOIN user_roles ur ON ur.role_id = rp.role_id
             WHERE ur.user_id = :userId",
            { userId: arguments.userID }
        )
        return listToArray( valueList( qry.permission ) )
    }
}
```

## User Entity with RBAC

```boxlang
/**
 * models/User.cfc
 */
class {

    property name="id"
    property name="name"
    property name="email"
    property name="roles"       default=""
    property name="permissions" default=""

    function hasRole( required role ) {
        // Support comma-delimited OR check
        return arguments.role.listToArray().some( r => listFindNoCase( variables.roles, r ) > 0 )
    }

    function can( required permission ) {
        return listFindNoCase( variables.permissions, arguments.permission ) > 0
    }

    function isAdmin() {
        return hasRole( "admin,superadmin" )
    }

    function getRoleList() {
        return listToArray( variables.roles )
    }
}
```

## Checking Roles in Handlers

```boxlang
class extends="coldbox.system.EventHandler" {

    property name="cbsecurity" inject="@cbsecurity"

    /**
     * Admin + moderator only
     * @secured admin,moderator
     */
    function manageUsers( event, rc, prc ) {
        // Permission check
        if ( !cbsecurity.can( "users.manage" ) ) {
            cbsecurity.block( "Requires users.manage permission" )
        }

        event.setView( "admin/users" )
    }

    function delete( event, rc, prc ) {
        // Dynamic resource + role check
        var targetUser = userService.findById( rc.id )

        // Only admin can delete other admins
        if ( targetUser.hasRole( "admin" ) && !cbsecurity.has( "superadmin" ) ) {
            cbsecurity.block( "Only superadmin can delete admin users" )
        }

        userService.delete( rc.id )
        relocate( "admin.users" )
    }
}
```

## CBSecurity Rules with Permissions

```boxlang
moduleSettings = {
    cbsecurity: {
        rules: [
            // Require specific permissions for routes
            {
                secureList: "admin.reports",
                match: "event",
                permissions: "reports.view",
                action: "block",
                statusCode: 403
            },
            {
                secureList: "admin.users.delete",
                match: "event",
                roles: "admin",
                permissions: "users.delete",
                action: "block"
            }
        ]
    }
}
```

## Hierarchical Roles

```boxlang
// Define role hierarchy — parent roles inherit child permissions
var roles = [
    { name: "viewer",    permissions: [ "read" ] },
    { name: "editor",    permissions: [ "read", "write" ], parent: "viewer" },
    { name: "admin",     permissions: [ "read", "write", "delete" ], parent: "editor" },
    { name: "superadmin", permissions: [ "read", "write", "delete", "manage" ], parent: "admin" }
]

// Check effective permissions (including inherited)
function hasEffectivePermission( required userID, required permission ) {
    var permissions = getEffectivePermissions( userID )
    return permissions.contains( arguments.permission )
}

function getEffectivePermissions( required userID ) {
    var roles = getUserRoles( arguments.userID )
    var permissions = []

    roles.each( ( role ) => {
        permissions.addAll( getInheritedPermissions( role ) )
    } )

    return permissions.toUnique()
}
```

## Common Role Sets

```boxlang
// Typical role/permission setup for a CMS
var defaultRoles = {
    guest:   [],
    member:  [ "profile.view", "profile.edit", "content.view" ],
    author:  [ "profile.view", "profile.edit", "content.view", "content.create", "content.edit" ],
    editor:  [ "profile.view", "profile.edit", "content.view", "content.create", "content.edit", "content.delete", "content.publish" ],
    admin:   [ "profile.view", "profile.edit", "content.*", "users.view", "users.create", "users.edit", "settings.*" ],
    superadmin: [ "*" ]
}
```
