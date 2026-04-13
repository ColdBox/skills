---
name: cbantisamy
description: >
  Use this skill when sanitizing HTML/XML user input to prevent XSS attacks using the OWASP AntiSamy library
  via the cbantisamy module. Covers installation, policy selection (strict/relaxed/slashdot/ebay/myspace),
  custom XML policy files, fluent sanitization API, ColdBox validation integration, and safe rendering of
  rich user-generated HTML.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# CBAntiSamy Skill

## When to Use This Skill

Load this skill when:
- Accepting HTML input from users (rich text editors, CMS content, comments)
- Sanitizing output to prevent stored or reflected XSS
- Choosing between AntiSamy policies (strict vs relaxed)
- Defining a custom AntiSamy policy XML file
- Integrating HTML sanitization with CBValidation constraints
- Rendering user-generated content safely in views

## Language Mode Reference

| Concept | BoxLang (`.bx`) | CFML (`.cfc`) |
|---------|-----------------|---------------|
| Class declaration | `class {` | `component {` |
| DI annotation | `@inject` above property | `property name="x" inject="y";` |

## Installation & Configuration

```bash
box install cbantisamy
```

```js
// config/ColdBox.cfc
moduleSettings = {
    cbantisamy = {
        defaultPolicy           = "relaxed", // strict | relaxed | slashdot | ebay | myspace
        policyPath              = "/config/antisamy/",
        validationIntegration   = true
    }
}
```

## Policy Reference

| Policy | Use Case |
|--------|----------|
| `strict` | Comments, form fields — minimal formatting only |
| `relaxed` | Blog posts, rich text — moderate HTML allowed |
| `slashdot` | Slashdot-style user content |
| `ebay` | Product descriptions with rich markup |
| `myspace` | Profile bios with extended HTML |

> **Rule of thumb**: `strict` for user-supplied data in forms, `relaxed` for CMS/blog bodies managed by trusted users.

## Core API

### Injection

```js
property name="antiSamy" inject="AntiSamy@cbantisamy";
```

### Sanitize with Default Policy

```js
var cleanHtml = antiSamy.clean( rc.content ?: "" )
```

### Sanitize with Specific Policy

```js
var cleanComment = antiSamy.withPolicy( "strict" ).clean( rc.comment ?: "" )
var cleanPost    = antiSamy.withPolicy( "relaxed" ).clean( rc.body ?: "" )
```

### Custom Policy File

Create `/config/antisamy/custom-policy.xml`:

```xml
<?xml version="1.0"?>
<anti-samy-rules xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
    <directives>
        <directive name="omitXmlDeclaration"      value="true"/>
        <directive name="omitDoctypeDeclaration"  value="true"/>
        <directive name="maxInputSize"            value="200000"/>
    </directives>
    <common-attributes>
        <attribute name="id">
            <regexp-list><regexp name="anything"/></regexp-list>
        </attribute>
        <attribute name="class">
            <regexp-list><regexp name="anything"/></regexp-list>
        </attribute>
    </common-attributes>
    <tag-rules>
        <tag name="p"    action="validate"/>
        <tag name="div"  action="validate"/>
        <tag name="span" action="validate"/>
        <tag name="a" action="validate">
            <attribute name="href">
                <regexp-list>
                    <regexp name="onsiteURL"/>
                    <regexp name="offsiteURL"/>
                </regexp-list>
            </attribute>
        </tag>
        <tag name="img" action="validate">
            <attribute name="src">
                <regexp-list>
                    <regexp name="onsiteURL"/>
                    <regexp name="offsiteURL"/>
                </regexp-list>
            </attribute>
            <attribute name="alt"/>
        </tag>
    </tag-rules>
    <common-regexps>
        <regexp name="anything"   value=".*"/>
        <regexp name="onsiteURL"  value="^/.*"/>
        <regexp name="offsiteURL" value="^https?://.*"/>
    </common-regexps>
</anti-samy-rules>
```

```js
var clean = antiSamy.withPolicy( "custom-policy" ).clean( dirtyHtml )
```

## Production Patterns

### Handler — Sanitize Before Save

```js
class Posts extends coldbox.system.EventHandler {
    @inject("AntiSamy@cbantisamy")
    property name="antiSamy";

    function save( event, rc, prc ) {
        // Sanitize rich body; never trust raw user HTML
        rc.body    = antiSamy.withPolicy( "relaxed" ).clean( rc.body    ?: "" )
        rc.excerpt = antiSamy.withPolicy( "strict"  ).clean( rc.excerpt ?: "" )

        postService.save( rc )
        relocate( "posts.index" )
    }
}
```

### CBValidation Integration

```js
component accessors="true" {
    // Automatically sanitizes on validation
    property name="content" validates="antiSamy";
    property name="comment" validates="antiSamy:strict";
    property name="bio"     validates="antiSamy:relaxed"
        validationMessage="Bio contains unsafe HTML content";
}
```

### Safe Output in Views

```cfml
<!--- Always render sanitized content — NEVER output raw user HTML --->
<div class="post-body">
    #antiSamy.withPolicy( "relaxed" ).clean( prc.post.getBody() )#
</div>
```

## Best Practices

- **Sanitize on input** (before persisting) AND trust that stored content was already sanitized
- **Choose the most restrictive policy** that meets your UX requirements
- **Never bypass sanitization** for admin-authored content without a clear business reason
- **Log or count sanitization hits** to detect injection attempts
- **Combine with CSP headers** for defense in depth — AntiSamy + Content-Security-Policy

## Documentation

- cbantisamy: https://github.com/coldbox-modules/cbantisamy
- OWASP AntiSamy: https://owasp.org/www-project-antisamy/
