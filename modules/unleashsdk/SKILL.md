---
name: unleashsdk
description: >
  Use this skill when implementing feature flags and gradual rollouts in ColdBox/BoxLang using the
  unleashsdk module. Covers UnleashClient configuration, isEnabled() checks, getVariant() for A/B
  testing, custom context, gradual rollout strategies, and production patterns for safe feature
  releases.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# UnleashSDK Skill

## When to Use This Skill

Load this skill when:
- Enabling/disabling features at runtime without redeployment
- Rolling out features gradually to a percentage of users
- Running A/B tests or controlled experiments
- Creating killswitches for risky features
- Targeting feature flags to specific user segments

## Installation

```bash
box install unleashsdk
```

## Configuration

### config/modules/unleashsdk.cfc

```js
function configure() {
    return {
        // Unleash server URL
        url        : getSystemSetting( "UNLEASH_URL",    "https://unleash.example.com/api" ),
        // Client API token
        apiToken   : getSystemSetting( "UNLEASH_TOKEN",  "" ),
        // Application identifier — shown in Unleash UI
        appName    : getSystemSetting( "APP_NAME",       "myapp" ),
        // Environment (matches Unleash environment activation)
        environment: getSystemSetting( "APP_ENV",        "production" ),
        // How often to sync flags from server (seconds)
        refreshInterval : 15,
        // Cache TTL (seconds) — use cached values when server unreachable
        cacheTTL        : 60
    }
}
```

## Core API

### Injection

```js
property name="unleash" inject="UnleashClient@unleashsdk";
```

### Check if a Feature is Enabled

```js
// Simple check
if ( unleash.isEnabled( "new-dashboard" ) ) {
    // render new dashboard
}

// With context (user ID, session ID, properties)
if ( unleash.isEnabled( "beta-checkout", {
    userId    : security.getCurrentUser().getId(),
    sessionId : session.id,
    properties: {
        role  : security.getCurrentUser().getRole(),
        plan  : security.getCurrentUser().getPlan()
    }
} ) ) {
    // show beta checkout flow
}
```

### Get a Variant (A/B Testing)

```js
var variant = unleash.getVariant( "checkout-button-color", {
    userId : security.getCurrentUser().getId()
} )

// variant.name   = "blue" | "green" | "disabled"
// variant.enabled = true/false
// variant.payload = { type: "string", value: "#0070f3" }

if ( variant.enabled ) {
    prc.buttonColor = variant.payload.value
} else {
    prc.buttonColor = "default"
}
```

## Production Patterns

### Feature Flag in a Handler

```js
function checkout( event, rc, prc ) {
    var userId = security.getCurrentUser().getId()

    // Feature flag controls which checkout version to render
    if ( unleash.isEnabled( "new-checkout-v2", { userId: userId } ) ) {
        prc.layout = "v2"
        event.setView( "checkout/v2/index" )
    } else {
        prc.layout = "v1"
        event.setView( "checkout/v1/index" )
    }
}
```

### Gradual Rollout Service

```js
class FeatureService {

    property name="unleash" inject="UnleashClient@unleashsdk";

    boolean function useNewSearch() {
        return unleash.isEnabled( "new-search-engine", {
            userId    : security.getCurrentUser().getId(),
            sessionId : session.id
        } )
    }

    struct function getCheckoutVariant() {
        var variant = unleash.getVariant( "checkout-experiment", {
            userId : security.getCurrentUser().getId()
        } )
        return {
            name   : variant.enabled ? variant.name : "control",
            config : variant.enabled ? variant.payload : {}
        }
    }
}
```

### Killswitch / Emergency Disable

```js
function run( event, rc, prc ) {
    if ( !unleash.isEnabled( "data-export-feature" ) ) {
        event.renderData(
            type       = "json",
            statusCode = 503,
            data       = { error: "This feature is temporarily disabled." }
        )
        return
    }

    // ... proceed with export
}
```

### Automated A/B Test Tracking

```js
function trackAndRoute( event, rc, prc ) {
    var variant = unleash.getVariant( "pricing-page", {
        userId : security.getCurrentUser().getId()
    } )

    // Record which variant the user saw
    analyticsService.track( "pricing_variant_shown", {
        userId  : security.getCurrentUser().getId(),
        variant : variant.name
    } )

    if ( variant.name == "v2" ) {
        event.setView( "pricing/v2" )
    } else {
        event.setView( "pricing/v1" )
    }
}
```

## Best Practices

- **Always provide a default** — `isEnabled()` returns `false` if the server is unreachable; design safe defaults
- **Use user context** for consistent per-user bucketing in gradual rollouts
- **Use `getVariant()`** for experiments, not just `isEnabled()` — variants carry payload and support multi-arm tests
- **Store the API token in an environment variable** — never commit it to source control
- **Use feature flags as killswitches** for risky features — wrap expensive or experimental code paths
- **Clean up stale flags** — remove flags from code once a rollout is complete (100% or 0%)
- **Test the `false` path** — ensure the application works correctly when a flag is disabled

## Documentation

- unleashsdk: https://github.com/coldbox-modules/unleashsdk
- Unleash feature flags: https://docs.getunleash.io
