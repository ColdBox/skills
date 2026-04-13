---
name: cbvalidation
description: >
  Use this skill when validating user input in ColdBox/BoxLang using cbvalidation. Covers constraint
  definitions, validate()/validateOrFail() calls, built-in validators, custom validators, shared
  constraint files, error handling in handlers, and integrating validation results with views and
  REST APIs.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# CBValidation Skill

## When to Use This Skill

Load this skill when:
- Validating form input or API payloads in handlers
- Defining reusable constraint sets or per-field rules
- Catching `ValidationException` and returning structured error responses
- Adding custom validators for domain-specific rules
- Integrating validation errors with views or JSON API responses

## Installation

```bash
box install cbvalidation
```

## Defining Constraints

### Inline in a Handler

```js
var constraints = {
    name : {
        required   : true,
        type       : "string",
        size       : "1..100",
        validator  : "NotNullValidator"
    },
    email : {
        required   : true,
        type       : "email"
    },
    age : {
        required   : false,
        type       : "numeric",
        min        : 0,
        max        : 150
    },
    password : {
        required   : true,
        size       : "8..255",
        regex      : "^(?=.*[A-Z])(?=.*[0-9]).{8,}$"
    }
}
```

### In a Shared Model (config/validation/UserConstraints.cfc)

```js
class {
    this.constraints = {
        email : {
            required : true,
            type     : "email",
            size     : "5..255",
            unique   : { table: "users", column: "email" }
        },
        name : {
            required : true,
            size     : "1..100"
        },
        password : {
            required : true,
            size     : "8..255"
        }
    }
}
```

## Validating in Handlers

### validate() — Manual Error Check

```js
property name="validationManager" inject="ValidationManager@cbvalidation";

function save( event, rc, prc ) {
    var result = validationManager.validate(
        target      = rc,
        constraints = {
            name  : { required: true, size: "1..100" },
            email : { required: true, type: "email" }
        }
    )

    if ( result.hasErrors() ) {
        prc.validationResult = result
        return event.setView( "users/form" )
    }

    userService.save( rc )
    messagebox.success( "User saved." )
    relocate( "users.index" )
}
```

### validateOrFail() — Exception-Based

```js
function create( event, rc, prc ) {
    // Throws ValidationException automatically
    validateOrFail(
        target      = rc,
        constraints = "UserConstraints"   // or inline struct
    )

    var user = userService.create( rc )
    event.renderData( type = "json", statusCode = 201, data = user.getMemento() )
}
```

### Catch ValidationException in REST APIs

```js
function create( event, rc, prc ) {
    try {
        validateOrFail( target = rc, constraints = "UserConstraints" )
        event.renderData( type = "json", statusCode = 201, data = userService.create( rc ) )

    } catch ( ValidationException e ) {
        event.renderData(
            type       = "json",
            statusCode = 422,
            data       = {
                message : "Validation failed",
                errors  : e.getValidationResult().getAllErrors().map( ( error ) => {
                    return {
                        field   : error.getField(),
                        message : error.getMessage()
                    }
                } )
            }
        )
    }
}
```

## Built-in Validators

| Validator | Usage |
|-----------|-------|
| `required: true` | Field must be present and non-empty |
| `type: "email"` | Valid email address |
| `type: "numeric"` | Numeric value |
| `type: "string"` | String type |
| `type: "boolean"` | Boolean |
| `type: "date"` | Valid date |
| `type: "url"` | Valid URL |
| `type: "uuid"` | Valid UUID |
| `size: "1..100"` | String/array length range |
| `min: 0` | Minimum numeric value |
| `max: 100` | Maximum numeric value |
| `regex: "pattern"` | Regular expression match |
| `sameAs: "fieldName"` | Must equal another field |
| `inList: "a,b,c"` | Value in allowed list |
| `unique` | DB uniqueness check (cborm integration) |

## Custom Validators

```js
// models/validators/StrongPasswordValidator.bx
class implements="cbvalidation.models.validators.IValidator" {

    string this.name = "StrongPassword"

    boolean function isValid( value, validationData, targetValue, target ) {
        if ( !len( value ) ) return true   // delegate to 'required'

        return (
            reFind( "[A-Z]", value ) &&    // uppercase
            reFind( "[0-9]", value ) &&    // digit
            reFind( "[!@#$%]", value ) &&  // special char
            len( value ) >= 8
        )
    }

    string function getDefaultMessage( validationData ) {
        return "Password must be at least 8 characters with an uppercase letter, number, and special character."
    }
}
```

```js
// Register in config/WireBox.cfc
binder.map( "StrongPasswordValidator@cbvalidation" )
      .to( "models.validators.StrongPasswordValidator" )
```

```js
// Use in constraint
password : {
    required  : true,
    validator : "StrongPassword"
}
```

## Rendering Errors in Views

```cfml
<!--- views/users/_form.cfm --->
<cfif isDefined( "prc.validationResult" ) AND prc.validationResult.hasErrors()>
    <div class="alert alert-danger">
        <ul>
        <cfloop array="#prc.validationResult.getAllErrors()#" index="error">
            <li>#encodeForHTML( error.getMessage() )#</li>
        </cfloop>
        </ul>
    </div>
</cfif>

<!--- Per-field error --->
<div class="mb-3 <cfif prc.validationResult.hasFieldErrors('email')>has-error</cfif>">
    <label>Email</label>
    <input type="email" name="email" value="#encodeForHTMLAttribute( rc.email ?: '' )#">
    <cfif prc.validationResult.hasFieldErrors( "email" )>
        <span class="text-danger">#prc.validationResult.getFieldErrors( "email" )[1].getMessage()#</span>
    </cfif>
</div>
```

## Best Practices

- **Always validate at the handler boundary** — never trust `rc` data passed to services
- **Use `validateOrFail()` in REST APIs** for concise exception-based flow
- **Catch `ValidationException`** explicitly and return 422 with structured error detail
- **Use shared constraint files** for constraints reused across handlers
- **Never return raw exception messages** to clients — always shape the error response
- **Encode error messages** in HTML views to prevent reflected XSS
- **Custom validators should be side-effect-free** — no writes, no session access

## Documentation

- cbvalidation: https://github.com/coldbox-modules/cbvalidation
- cbvalidation docs: https://coldbox-validation.ortusbooks.com
