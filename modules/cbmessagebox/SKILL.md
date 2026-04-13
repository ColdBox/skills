---
name: cbmessagebox
description: >
  Use this skill when displaying flash-scope messages (info, success, warning, error) in ColdBox/BoxLang
  views using the cbmessagebox module. Covers installation, setting messages in handlers, rendering in
  views/layouts, checking for messages, clearing messages, and Bootstrap-compatible output patterns.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# CBMessagebox Skill

## When to Use This Skill

Load this skill when:
- Displaying success/error/warning/info notifications after form submissions or redirects
- Integrating flash-scope messaging into a ColdBox layout or view
- Checking for specific message types before rendering
- Clearing messages programmatically
- Customizing the message template to match a CSS framework

## Installation

```bash
box install cbmessagebox
```

## Core API

### Injection

```js
// cbmessagebox is available as a mixin in all ColdBox handlers and views
// Direct injection when needed in services or interceptors:
property name="messagebox" inject="messagebox@cbmessagebox";
```

### Setting Messages (in Handlers)

```js
// Types: info | success | warning | error
messagebox.info(    "Profile updated." )
messagebox.success( "Changes saved successfully!" )
messagebox.warning( "Please verify your email address." )
messagebox.error(   "An error occurred while processing your request." )
```

### Append Multiple Messages

```js
messagebox.info( "Step 1 complete." )
messagebox.info( "Step 2 complete." )
```

### Clear Messages

```js
messagebox.clearMessage( "info" )   // clear specific type
messagebox.clearAll()               // clear all types
```

## Production Patterns

### Handler — CRUD with Flash Messages

```js
function save( event, rc, prc ) {
    try {
        // Validate first
        validateOrFail( target = rc, constraints = "userConstraints" )

        userService.update( rc.id, rc )

        messagebox.success( "User updated successfully." )
        relocate( "users.index" )

    } catch ( ValidationException e ) {
        messagebox.error( "Please fix the validation errors below." )
        flash.put( "validationResult", e.getValidationResult() )
        relocate( "users.edit" )

    } catch ( any e ) {
        messagebox.error( "An unexpected error occurred: #e.message#" )
        relocate( "users.edit" )
    }
}

function delete( event, rc, prc ) {
    userService.delete( rc.id )
    messagebox.success( "User deleted." )
    relocate( "users.index" )
}
```

### Layout — Render Messages Once

```cfml
<!--- In your main layout, just before or after the body --->
<cfif messagebox.hasMessage()>
    #messagebox.renderit()#
</cfif>
```

### Render Specific Type

```cfml
<!--- Render only errors at top of a form --->
<cfif messagebox.hasMessage( "error" )>
    <div class="alert-block">
        #messagebox.renderMessage( "error" )#
    </div>
</cfif>

<!--- Render all messages --->
#messagebox.renderit()#
```

### Custom Bootstrap 5 Template

```js
// In config or an interceptor's configure method
messagebox.setTemplate(
    "<div class='alert alert-{type} alert-dismissible fade show' role='alert'>" &
    "{message}" &
    "<button type='button' class='btn-close' data-bs-dismiss='alert'></button>" &
    "</div>"
)
```

Available tokens in the template: `{type}`, `{message}`.

## Best Practices

- **Always `relocate()` after setting a message** — never render a page in the same request that sets a flash message (post-redirect-get pattern)
- **Render messages in the layout** — not in individual views — for consistent placement
- **Use `hasMessage()` before rendering** — silent when no messages exist
- **Match `{type}` values to CSS framework class names** by customizing the template
- **Prefer specific types** (`error` vs `warning`) so users understand severity

## Documentation

- cbmessagebox: https://github.com/coldbox-modules/cbmessagebox
