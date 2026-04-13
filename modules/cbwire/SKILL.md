---
name: cbwire
description: >
  Use this skill when building dynamic, reactive UI components in ColdBox/BoxLang without writing
  JavaScript using cbwire (a Livewire-inspired library). Covers component creation, reactive data
  properties, wire:model / wire:click directives, lifecycle hooks, actions, computed properties,
  events, and production patterns for forms and live search.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# CBWire Skill

## When to Use This Skill

Load this skill when:
- Building interactive UI without writing JavaScript (live search, real-time forms, counters)
- Creating stateful server-rendered components that react to user input
- Implementing reactive data tables, modals, or multi-step wizards
- Handling form submission and validation inline without page reloads
- Emitting and listening to cross-component events

## Installation

```bash
box install cbwire
```

Include Alpine.js and CBWire scripts in your layout:

```html
<script defer src="https://cdn.jsdelivr.net/npm/alpinejs@3.x.x/dist/cdn.min.js"></script>
<script src="/cbwire/cbwire.js"></script>
```

## Language Mode Reference

| Feature | BoxLang | CFML |
|---------|---------|------|
| Component keyword | `class` | `component` |
| Data block | `data {}` or properties | `data {}` / `variables` |
| Arrow functions | Supported | Not supported |

## Creating a Wire Component

### Component Class

```js
// wires/SearchUsers.bx
class extends="cbwire.models.Component" {

    // Reactive properties — bound to wire:model in the template
    data = {
        search    : "",
        perPage   : 10,
        isLoading : false
    }

    // Computed property (cached per render cycle)
    array function getResults() {
        if ( len( trim( data.search ) ) < 2 ) return []
        return userService.search( data.search, data.perPage )
    }

    // Action called from wire:click
    void function clearSearch() {
        data.search = ""
    }
}
```

### Component Template (views/wires/SearchUsers.cfm)

```html
<div>
    <input
        type="text"
        wire:model.debounce.300ms="search"
        placeholder="Search users..."
        class="form-control"
    />

    <cfif isLoading>
        <div class="spinner-border text-primary"></div>
    </cfif>

    <cfif arrayLen( results )>
        <ul class="list-group mt-2">
            <cfloop array="#results#" index="user">
                <li class="list-group-item">#encodeForHTML( user.name )# — #encodeForHTML( user.email )#</li>
            </cfloop>
        </ul>
    <cfelseif len( trim( search ) ) gte 2>
        <p class="text-muted mt-2">No results found.</p>
    </cfif>

    <cfif len( trim( search ) )>
        <button wire:click="clearSearch" class="btn btn-secondary btn-sm mt-2">Clear</button>
    </cfif>
</div>
```

### Render in a View

```cfml
#renderWire( "SearchUsers" )#
```

## Core Directives

| Directive | Purpose |
|-----------|---------|
| `wire:model="prop"` | Two-way bind input to data property |
| `wire:model.debounce.300ms="prop"` | Debounced binding |
| `wire:model.lazy="prop"` | Bind on change/blur only |
| `wire:click="methodName"` | Call action on click |
| `wire:click="method('param')"` | Action with argument |
| `wire:submit.prevent="method"` | Call action on form submit |
| `wire:keydown.enter="method"` | Trigger on key press |
| `wire:loading` | Show while request in flight |
| `wire:dirty` | Show when data changed but unsaved |
| `wire:poll.5s="refresh"` | Poll server every N seconds |

## Lifecycle Hooks

```js
class extends="cbwire.models.Component" {

    function onMount()  { /* runs when component first renders */ }
    function onUpdate() { /* runs before every subsequent render */ }
    function onHydrate( data ) { /* runs when state is restored from client */ }
}
```

## Events

### Emit (from component)

```js
emit( "userSaved", { id: user.getId() } )
emitTo( "UserList", "refresh" )      // target specific component
emitUp( "formClosed" )               // bubble to parent
```

### Listen (in class)

```js
listeners = {
    userSaved : "handleUserSaved"
}

function handleUserSaved( id ) {
    // Refresh list or show notification
}
```

## Production Patterns

### Live Search with Pagination

```js
class extends="cbwire.models.Component" {

    property name="userService" inject="UserService";

    data = {
        search      : "",
        currentPage : 1,
        perPage     : 10
    }

    void function resetPage() {
        data.currentPage = 1
    }

    // Computed — auto-refreshes when data.search changes
    struct function getResult() {
        return userService.search(
            term   = data.search,
            page   = data.currentPage,
            max    = data.perPage
        )
    }
}
```

### Inline Form with cbvalidation

```js
class extends="cbwire.models.Component" {

    property name="validationManager" inject="ValidationManager@cbvalidation";

    data = {
        name  : "",
        email : ""
    }

    errors = {}

    void function save() {
        errors = {}

        var result = validationManager.validate(
            target      = data,
            constraints = {
                name  : { required: true, size: "1..100" },
                email : { required: true, type: "email" }
            }
        )

        if ( result.hasErrors() ) {
            for ( var error in result.getAllErrors() ) {
                errors[ error.getField() ] = error.getMessage()
            }
            return
        }

        userService.create( data )
        data.name  = ""
        data.email = ""
        emit( "userCreated" )
        messagebox.success( "User created." )
    }
}
```

## Best Practices

- **Use `wire:model.debounce`** for search inputs — avoids a server round-trip on every keystroke
- **Keep component data flat** — nested structs are supported but harder to bind
- **Use computed properties for derived data** — they are cached per render and avoid redundant queries
- **Emit events for cross-component communication** — use emitTo for targeted messaging
- **Use `wire:loading`** to provide visual feedback during server requests
- **Encode output with `encodeForHTML()`** in templates — cbwire does not sanitize output automatically
- **Avoid heavy processing in computed properties** — use service layer and cache where appropriate
- **Use `onMount()`** to initialize data requiring async service calls

## Documentation

- cbwire: https://github.com/coldbox-modules/cbwire
- cbwire docs: https://cbwire.ortusbooks.com
