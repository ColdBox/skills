---
name: cbdebugger
description: >
  Use this skill when installing, configuring, or using the CBDebugger visual debugging panel in a ColdBox
  application. Covers enabling the panel in development, performance profiling, SQL query tracking, cache
  monitoring, request inspection, and ensuring the debugger is disabled in production.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# CBDebugger Skill

## When to Use This Skill

Load this skill when:
- Installing or enabling the CBDebugger module
- Diagnosing performance bottlenecks in development
- Tracking SQL queries and their execution times
- Inspecting RC/PRC/CGI scopes during debugging
- Configuring the debugger to appear only in specific environments
- Ensuring debugger is never exposed to end-users in production

## Installation

```bash
box install cbdebugger
```

## Configuration

**Always gate the debugger behind an environment check:**

```js
// config/ColdBox.cfc
moduleSettings = {
    cbdebugger = {
        // Only enable in development — never expose in production
        enabled        = getSetting( "environment" ) == "development",
        renderDebugPanel = true,
        expanded       = false,         // collapse panel by default

        // Optional: restrict to specific IPs
        ipList         = "127.0.0.1",   // comma-separated list

        // Profiling options
        showTracers    = true,
        queryMonitor   = {
            enabled    = true,
            logQueries = false          // set true to log all queries
        }
    }
}
```

## Features

| Feature | Description |
|---------|-------------|
| **Performance Profiling** | Request timings and execution paths |
| **SQL Query Tracking** | All DB queries with execution time and bindings |
| **Cache Monitoring** | CacheBox statistics and operations |
| **Request Inspection** | RC, PRC, CGI, form, and URL scopes |
| **Tracer Messages** | Custom `tracer()` calls inline in code |
| **Exception Detail** | Full stack traces with local variable context |

## Usage in Handlers / Flow

```js
// Add a debug tracer message (only visible when debugger is enabled)
tracer( "About to call payment service", { orderId : rc.orderId } )

// Then continue your logic
paymentService.charge( rc.orderId )
```

## Production Safety Checklist

- `enabled` **must** evaluate to `false` in staging and production
- Never deploy with `ipList` empty and `enabled = true` on a public server
- Use `getSetting("environment")` or `.env` variables — not hard-coded `true`
- Panel renders at bottom of HTML responses — harmless when disabled but wasteful at scale

## Documentation

Full documentation: https://coldbox-debugger.ortusbooks.com
