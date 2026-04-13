---
name: cbi18n
description: >
  Use this skill when adding internationalization (i18n) and localization (l10n) to a ColdBox/BoxLang
  application with the cbi18n module. Covers installation, resource bundle formats (.properties and JSON),
  locale management, translation helpers ($r / getResource), positional and named substitutions, multiple
  bundles, locale switching, and date/number formatting.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# CBi18n Skill

## When to Use This Skill

Load this skill when:
- Adding multi-language support to a ColdBox application
- Creating or reading `.properties` / JSON resource bundle files
- Getting and setting the active locale
- Using `getResource()` / `$r()` helpers for translations
- Passing value substitutions (positional `{1}` or named `{name}`)
- Switching locale based on user preference or URL parameter
- Formatting dates, numbers, and currency per locale

## Language Mode Reference

| Concept | BoxLang (`.bx`) | CFML (`.cfc`) |
|---------|-----------------|---------------|
| Mixin helpers | `getResource()`, `$r()`, `getFWLocale()`, `setFWLocale()` | same — available in handlers/views |

## Installation & Configuration

```bash
box install cbi18n
```

```js
// config/ColdBox.cfc
moduleSettings = {
    cbi18n = {
        defaultResourceBundle  = "includes/i18n/main",   // path WITHOUT locale/extension
        defaultLocale          = "en_US",
        localeStorage          = "cookieStorage@cbstorages",
        unknownTranslation     = "**NOT FOUND**",
        logUnknownTranslation  = true,

        resourceBundles = {
            admin  = "includes/i18n/admin",
            emails = "includes/i18n/emails"
        }
    }
}
```

## Resource Bundle Files

### Java `.properties` Format (recommended)

```properties
# includes/i18n/main_en_US.properties
welcome.message=Welcome, {1}!
user.login=Login
user.logout=Logout
btn.submit=Submit
btn.cancel=Cancel
error.required=The field {1} is required
error.email=Please provide a valid email address
```

```properties
# includes/i18n/main_es_ES.properties
welcome.message=¡Bienvenido, {1}!
user.login=Iniciar Sesión
user.logout=Cerrar Sesión
btn.submit=Enviar
btn.cancel=Cancelar
error.required=El campo {1} es requerido
error.email=Por favor proporcione un email válido
```

### JSON Format

```json
{
    "welcome" : { "message" : "Welcome, {1}!" },
    "btn"     : { "submit"  : "Submit", "cancel" : "Cancel" }
}
```

## Core API

### Mixin Helpers (handlers, views, layouts)

```cfml
<!--- Simple translation --->
#getResource( "btn.submit" )#

<!--- Short alias --->
#$r( "btn.submit" )#

<!--- Positional substitution --->
#getResource( resource = "welcome.message", values = [ "John" ] )#

<!--- Named substitution (if using {name} tokens) --->
#getResource( resource = "user.greeting", values = { name : "John", age : 30 } )#

<!--- From a non-default bundle --->
#getResource( resource = "dashboard.title", bundle = "admin" )#
```

### Locale Management

```js
// Get current locale
var locale = getFWLocale()          // returns "en_US"

// Set locale (persisted in configured storage)
setFWLocale( "es_ES" )

// Set with timezone
setFWLocale( "es_ES", "Europe/Madrid" )
```

## Production Patterns

### Language Switch Handler

```js
class Language extends coldbox.system.EventHandler {
    function changeLocale( event, rc, prc ) {
        // Validate locale before setting — prevent arbitrary values
        var supported = [ "en_US", "es_ES", "fr_FR", "de_DE" ]
        var requested = rc.locale ?: "en_US"

        if ( !supported.findNoCase( requested ) ) {
            requested = "en_US"
        }

        setFWLocale( requested )

        // Redirect back to referring page
        var referer = event.getHTTPHeader( "Referer", event.buildLink( "main.index" ) )
        relocate( url = referer )
    }
}
```

### Locale Selector in View

```cfml
<form action="#event.buildLink('language.changeLocale')#" method="POST">
    #csrf()#
    <select name="locale" onchange="this.form.submit()">
        <option value="en_US" #getFWLocale() eq "en_US" ? "selected" : ""#>English</option>
        <option value="es_ES" #getFWLocale() eq "es_ES" ? "selected" : ""#>Español</option>
        <option value="fr_FR" #getFWLocale() eq "fr_FR" ? "selected" : ""#>Français</option>
    </select>
</form>
```

### Programmatic Injection

```js
class UserProfileHandler extends coldbox.system.EventHandler {
    @inject("i18n@cbi18n")
    property name="i18n";

    function show( event, rc, prc ) {
        // Use i18n service directly when view helpers are unavailable
        prc.welcomeTitle = i18n.getResource(
            resource = "welcome.message",
            values   = [ prc.user.getFirstName() ]
        )
        event.setView( "users/show" )
    }
}
```

### Formatted Dates / Numbers per Locale

```js
// Use Java locale-aware formatting
var jLocale = createObject( "java", "java.util.Locale" )
    .init( getFWLocale().listFirst( "_" ), getFWLocale().listLast( "_" ) )

var formatter = createObject( "java", "java.text.DateFormat" )
    .getDateInstance( createObject( "java", "java.text.DateFormat" ).SHORT, jLocale )
var formatted = formatter.format( now().getTime() )
```

## Best Practices

- **Never hardcode user-facing strings** — always use resource bundles
- **Use positional substitutions** for dynamic content: `{1}`, `{2}` (1-indexed)
- **Validate locale on input** — restrict to a known supported list
- **Keep bundle keys hierarchical** — `error.required` not `errRequired` — for readability
- **Log missing translations** (`logUnknownTranslation = true`) in development to catch gaps
- **Test all supported locales** before deploying multilingual routes

## Documentation

- cbi18n: https://github.com/coldbox-modules/cbi18n
