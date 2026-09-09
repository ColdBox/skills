---
name: cbwire
description: >
  Use this skill when building reactive UI components in ColdBox/BoxLang with cbwire, a
  Livewire-inspired library. Covers wire components and templates, wire:model and wire:click
  directives, lifecycle hooks, computed properties, events, validation, file uploads, lazy
  loading, JavaScript integration, and the behavior traps that are hard to diagnose.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# CBWire Skill

## When to Use This Skill

Load this skill when:

- Building interactive UI without writing JavaScript: live search, forms, counters, data tables
- Creating stateful server-rendered components that react to user input
- Handling form submission and validation inline, without page reloads
- Dispatching and listening to cross-component events, or wiring a third-party JavaScript widget
- Debugging a morph, a dirty-state indicator, or a file upload that misbehaves

## Installation

```bash
box install cbwire
```

**No script tags are required.** CBWIRE injects its own assets. Do not add an Alpine.js CDN tag,
because Alpine ships inside the bundled Livewire build and a second instance conflicts. Do not
hand-write a path to a cbwire script: the module emits the bundled `livewire.js` with the CSRF
token and update endpoint already attached.

## Configuration

Override in `config/ColdBox.bx` or `config/ColdBox.cfc` under `moduleSettings.cbwire`. Every
setting has a working default, so only change one for a reason.

```boxlang
// config/ColdBox.bx. Both values here are non-defaults, shown to illustrate the shape.
moduleSettings = {
    "cbwire" : {
        "moduleRootURL" : "/assets/cbwire",
        "wiresLocation" : "components"
    }
};
```

| Setting | Default | Change it when |
|---------|---------|----------------|
| `autoInjectAssets` | `true` | You need control over where the tags land. See below |
| `moduleRootURL` | `/modules/cbwire` | The module is not web-accessible at that path, such as behind a CDN, a reverse proxy, or the newer ColdBox layout that puts modules under `./lib/modules/`. A 404 on `livewire.js` is the symptom |
| `wiresLocation` | `wires` | Your components live in a differently named folder |
| `updateEndpoint` | `/cbwire/update` | URL rewriting is off. Use `/index.cfm/cbwire/update` |
| `uploadsStoragePath` | system temp `/cbwire` | Temp is not writable, or not shared across nodes |
| `storagePath` | the module's `models/tmp` | You compile single-file components and need them elsewhere |
| `trimStringValues` | `false` | You want every bound string trimmed on the way in |
| `throwOnMissingSetterMethod` | `false` | You want a loud error instead of a silently ignored update |
| `showProgressBar` | `true` | The `wire:navigate` bar clashes with your own loading UI |
| `progressBarColor` | `##2299dd` | Brand match |
| `secret` | derived from the module path | You run several servers and need a stable checksum key |
| `checksumValidation` | `true` | Effectively never. Leave it on |
| `csrfEnabled` | `true` | Effectively never. Leave it on |
| `csrfStorage` | `SessionCSRFStorage@cbwire` | Clustered or sessionless. Use `CacheCSRFStorage@cbwire` |

`progressBarColor` holds a literal `#`, so it must be doubled to `##` inside a CFML or BoxLang
string. A single `#` starts an interpolation and throws.

### Turning off automatic asset injection

With `autoInjectAssets` on, an interceptor inserts the CSS before the literal `</head>` and the JS
before the literal `</body>`. Set it to `false` and place them yourself, with `#wireStyles()#` in
the head and `#wireScripts()#` at the end of the body.

Turn it off for three reasons. Your own bundle must load after CBWIRE, because it expects
`Livewire` to exist or hooks `livewire:init`. A CSP needs a nonce on the tag. Or the layout has no
literal `</head>` or `</body>`. That last case is the one that bites. The interceptor runs a plain
string replace, once, so a fragment layout silently gets no assets at all. Turning it off is all or
nothing. A layout that forgets `wireScripts()` renders components that look fine and do nothing on
click.

## Language Mode Reference

Every example below is BoxLang. Read it as CFML with this table.

| Feature | BoxLang | CFML |
|---------|---------|------|
| Class file | `.bx` | `.cfc` |
| Template file | `.bxm` | `.cfm` |
| Component keyword | `class` | `component` |
| Struct literal | `{ "key" : value }` | `{ "key" = value }` |
| Output block | `<bx:output>` | `<cfoutput>` |
| Conditional | `<bx:if>` / `<bx:elseif>` / `<bx:else>` | `<cfif>` / `<cfelseif>` / `<cfelse>` |
| Loop | `<bx:loop>` | `<cfloop>` |
| Script block | `<bx:script>` | `<cfscript>` |
| Component-level `secured` | `@secured` above `class` | `secured` attribute on `component` |
| Method-level `secured` | `@secured("admin")` above the function | `function x() secured="admin"` |
| Arrow functions | Supported | Not supported |

## Creating a Wire Component

A component is a class plus a template. **Both live in `./wires/`, side by side.** The template is
not a ColdBox view and does not belong in `views/`. A wrong path throws "A .bxm or .cfm template
could not be found". Name the template exactly like the class: on a case-sensitive file system
such as Linux, `SearchUsers.bx` needs `SearchUsers.bxm`, not `searchUsers.bxm`.

```boxlang
// wires/SearchUsers.bx
class extends="cbwire.models.Component" {

    property name="userService" inject="UserService";

    // Reactive properties, bound to wire:model in the template
    data = {
        "search"  : "",
        "perPage" : 10
    };

    // Computed. Cached for the length of one render.
    function getResults() computed {
        if ( len( trim( data.search ) ) < 2 ) {
            return [];
        }
        return userService.search( data.search, data.perPage );
    }

    // Action, called from wire:click
    function clearSearch() {
        data.search = "";
    }
}
```

The `computed` attribute on the function is what enables the caching. A computed keeps its own
name, so `getResults()` stays `getResults()` and never becomes a `results` property. Call it as a
function in the template, and pass `false` to bypass the cache once, as in `getResults( false )`.

```html
<!--- wires/SearchUsers.bxm --->
<bx:output>
<div>
    <input type="text" wire:model.live.debounce.300ms="search" class="form-control">

    <bx:if arrayLen( getResults() )>
        <ul class="list-group mt-2">
            <bx:loop array="#getResults()#" index="user">
                <li class="list-group-item">#encodeForHTML( user.name )#</li>
            </bx:loop>
        </ul>
    <bx:elseif len( trim( search ) ) gte 2>
        <p class="text-muted mt-2">No results found.</p>
    </bx:if>

    <button wire:click="clearSearch" class="btn btn-secondary btn-sm">Clear</button>
</div>
</bx:output>
```

A component must render **exactly one root element**. See Behavior and Gotchas for why breaking
that now fails quietly instead of throwing.

### Render in a layout or view

```cfml
#wire( "SearchUsers" )#
#wire( name = "SearchUsers", params = { "teamId" : 4 } )#
#wire( "SearchUsers@myModule" )#
#wire( "admin.reports.SalesReport" )#
```

`wire()` takes `name`, `params`, `key`, `lazy`, and `lazyIsolated`. Call it from a ColdBox layout
or view, or another component template. Where `wire()` is not in scope, such as a model or an
interceptor, inject `CBWIREController@cbwire` and call
`CBWIREController.wire( "EditablePage", { "contentKey" : "hero" } )`.

## Core Directives

| Directive | Purpose |
|-----------|---------|
| `wire:model="prop"` | Bind an input to a data property, synced on the next action |
| `wire:click="method"` | Call an action on click |
| `wire:click="method('param')"` | Action with an argument |
| `wire:submit.prevent="method"` | Call an action on form submit |
| `wire:keydown.enter="method"` | Trigger on a key press |
| `wire:init="method"` | Run an action once, right after the first render in the browser |
| `wire:loading` | Show while a request is in flight |
| `wire:dirty` | Show while data differs from the last server state |
| `wire:confirm="text"` | Browser confirm before the action runs |
| `wire:poll.5s="refresh"` | Poll the server every N seconds |
| `wire:navigate` | SPA-style navigation without a full page load |
| `wire:current` | Style the link matching the current URL |
| `wire:key="id"` | Identity for a looped or nested element |
| `wire:ignore` | Exclude a subtree from the morph |
| `wire:ignore.self` | Exclude only this element's own attributes |
| `wire:replace` | Rebuild the element from scratch instead of diffing it |
| `wire:show` | Toggle visibility with CSS, keeping the element in the DOM |
| `wire:text` | Update text content with no network round trip |
| `wire:cloak` | Hide an element until CBWIRE has initialized |
| `wire:transition` | Animate an element in and out |
| `wire:offline` | React to the browser going offline |
| `wire:stream` | Receive partial output pushed by `stream()` |

`wire:loading` takes a large modifier family: `.delay.shortest` through `.delay.longest`, `.attr`,
`.class`, `.remove`, and the display modifiers `.inline`, `.block`, `.flex`, `.grid`, `.table`.

## Data Binding

### wire:model modifiers

| Form | When it syncs |
|------|---------------|
| `wire:model="prop"` | On the next action. Nothing is sent while typing |
| `wire:model.live="prop"` | As the user types, debounced 150ms on text inputs |
| `wire:model.live.debounce.300ms="prop"` | As the user types, with your own debounce |
| `wire:model.blur="prop"` | When the input loses focus |
| `wire:model.change="prop"` | On the DOM change event |
| `wire:model.lazy="prop"` | Identical to `.change` in CBWIRE 5 |
| `wire:model.throttle.500ms="prop"` | Rate limited rather than debounced |
| `wire:model.number="prop"` | Cast to a number on the server |
| `wire:model.boolean="prop"` | Cast to a boolean on the server |
| `wire:model.fill="prop"` | Seed from the element's `value` attribute on load |

**A bare `wire:model` does not talk to the server as you type.** It holds the value on the client
and sends it with the next action. This is the single most common surprise for anyone arriving
from CBWIRE 2 or Livewire 2. `.lazy` and `.change` resolve to the same code path, and neither one
defers until an action despite what the name suggests. If you want the update held until an
action, use a bare `wire:model`.

Do not seed a textarea from its property. `wire:model` fills it for you, and a seeded value fights
the morph. Write `<textarea wire:model="content"></textarea>`, empty, never
`<textarea wire:model="content">#content#</textarea>`.

### Nested data binding

New in CBWIRE 5. Bind straight into a nested struct with a dot-separated path, so your data keeps
the shape it already has instead of one flat key per field.

```boxlang
data = { "user" : { "name" : { "first" : "", "last" : "" }, "contact" : { "email" : "" } } };
```

Bind with `wire:model="user.name.first"` and `wire:model="user.contact.email"`.

- **Depth is not limited.** CBWIRE walks the struct and builds a path for every leaf.
- **Missing keys are created** on update rather than throwing.
- **Array elements bind by zero-based index**, so `wire:model="items.0"` is the first element.
- **Lifecycle hooks swap dots for underscores.** `user.contact.email` fires
  `onUpdateUser_contact_email( value, oldValue )`.

### Locking properties

Nested binding widens what the client can write, so pair it with `locked` for anything the browser
must not change. An update to a locked property throws `CBWIREException` rather than applying.

```boxlang
locked = [ "user.id", "user.createdAt" ];
```

**Use the array form.** `locked` also accepts a comma list, but the array is matched
case-insensitively and the list is not. A list whose casing does not match the incoming key
protects nothing and reports nothing.

## Lifecycle Hooks

### Order of operations

| First render | Every later AJAX request |
|---|---|
| `onBoot()` | `onBoot()` |
| `onSecure()` | `onSecure()` |
| `onMount()` | `onHydrate<Property>()`, then `onHydrate()` |
| `onRender()` | `onUpdate<Property>()`, then `onUpdate()` |
| | The requested actions |
| | `onRender()` |

### The hooks

| Hook | Signature | When it fires |
|---|---|---|
| `onBoot` | `()` | First on every request, right after dependencies are injected |
| `onSecure` | `( event, prc, isInitial, params )` | Before every other hook. Return `false` to halt and render an empty div |
| `onMount` | `( event, rc, prc, params )` | First render only. Params passed through `wire()` arrive here |
| `onHydrate` | `()` | State restored from the client, on every later request |
| `onHydrate<Property>` | `()` | The same, for one property |
| `onUpdate` | `( newValues, oldValues )` | After any data property changes |
| `onUpdate<Property>` | `( value, oldValue )` | After that one property changes |
| `onRender` | `()` | Wrap or replace the rendered output |
| `onUploadError` | `( property, errors, multiple )` | A file upload returned a non-2xx status |

**Never override `onDIComplete()`.** CBWIRE uses it to build the component: the id, the data
property list, the computed properties, the listeners, the metadata cache. An override that does
not call `super.onDIComplete()` breaks the component in ways that are hard to trace. Use
`onBoot()`, which CBWIRE calls at the end of its own setup for exactly this purpose.

The `onHydrate` hooks are real, but the documentation and the module disagree about what arguments
they receive. Declare them with no parameters and read component state directly.

### onSecure

Returns `false` to stop the component rendering. It fires on the initial render **and** on every
AJAX request, so the check cannot be skipped after mount.

```boxlang
// wires/AdminDashboard.bx, inside the class
secureMountFailMessage = "<div class='alert'>Please log in.</div>";

function onSecure( event, prc, isInitial, params ) {
    return authService.isLoggedIn() ? true : false;
}
```

`secureMountFailMessage` also works as a `wire()` param and as a module setting. The component
value wins. The documented parameter list includes `rc`, but the module does not pass it, so
declaring `rc` leaves it null. Read the request collection through `event.getCollection()`.

### onMount and parameter auto-population

**A CBWIRE 5 breaking change.** When a component defines `onMount()`, parameters passed through
`wire()` are no longer assigned to matching data properties for you. Assign them yourself:

```boxlang
// wires/ShowPost.bx, inside the class
data = { "title" : "", "author" : "" };

function onMount( event, rc, prc, params ) {
    data.title  = params.title;
    data.author = params.author;
}
```

Drop `onMount()` entirely and the auto-population comes back. `onMount()` runs once, so anything
you read from `rc` or `prc` there is gone on the next request. Copy it into a data property.

### Per-property hooks

`onUpdate<Property>` fires only when that property changes and receives both values, as in
`function onUpdateSearch( value, oldValue )`. For a nested path, replace the dots with
underscores, so `address.city` becomes `onUpdateAddress_city`.

## Events

CBWIRE 5 uses `dispatch`. Components have no `emit()`, `emitTo()`, or `emitUp()`. The chainable
test API does have an `emit()`, which is a different thing. See Testing.

```boxlang
dispatch( "userSaved", { "id" : user.getId() } );   // to everyone
dispatchTo( "UserList", "refresh", {} );            // to a named component
dispatchSelf( "reload", {} );                       // to this component only
```

From a template use `$dispatch()`, as in `wire:click="$dispatch( 'post-added' )"`, and from
JavaScript use `Livewire.dispatch()`.

Listen in the class with a `listeners` struct. A listed method must exist, or CBWIRE throws at
load time. Quote the keys, because JavaScript event names are case-sensitive and unquoted CFML
struct keys are upper-cased.

```boxlang
// wires/Posts.bx, inside the class
listeners = { "userSaved" : "handleUserSaved" };

function handleUserSaved( id ) {
    // refresh a list, show a notification
}
```

`dispatch()` also reaches plain JavaScript, which makes it the bridge to any widget with no CBWIRE
integration. Listen for the event name on `window` and read the payload from `event.detail`.

## Validation

CBWIRE wraps cbvalidation. Declare `constraints` and call `validate()`. Do not hand-roll a
`ValidationManager` call. The `cbvalidation` skill covers the constraint syntax itself.

```boxlang
// wires/UserRegistration.bx, inside the class
data = { "name" : "", "email" : "" };

constraints = {
    "name"  : { "required" : true, "size" : "1..100" },
    "email" : { "required" : true, "type" : "email" }
};

function save() {
    validate();
    if ( hasErrors() ) {
        return;
    }
    userService.create( data );
    reset();
    dispatch( "userCreated", {} );
}
```

Show the messages in the template with `hasError()` and `getError()`, inside a
`<form wire:submit.prevent="save">`:

```html
<input type="text" wire:model.blur="name">
<bx:if hasError( "name" )>
    <span class="invalid-feedback">#getError( "name" )#</span>
</bx:if>
```

| Method | Purpose |
|--------|---------|
| `validate()` | Run the constraints, store and return the result |
| `validateOrFail()` | The same, but raises `ValidationException` |
| `hasErrors()` | True when the stored result has any error |
| `hasError( field )` | True when that field has an error |
| `getError( field )` | The first message for that field, as a string |
| `getErrors()` | The full error set, as objects rather than strings |

Three things about validation that are easy to get wrong:

- **CBWIRE validates on its own.** When cbvalidation is installed, `validate()` runs before every
  template render, whether or not you call it. That is why errors appear without an explicit call.
  It runs at render time, which is **after** your actions, not before.
- **`validateOrFail()` raises `ValidationException`, but an action invoked from the browser
  swallows it** so the component can still render. Do not wrap it in a try/catch expecting to
  catch anything. It also stores no result, so a later `hasErrors()` reads whatever the last
  `validate()` left behind.
- **`validates( field )` does not do what its name says.** It reports whether the component has
  any errors at all, not whether that one field passed. Use `hasError( field )`.

Prefer `getError( field )` over `getErrors()`. `getError()` returns a printable string, while
`getErrors()` returns error objects that print as object output.

## File Uploads

Bind a file input with `wire:model`, as in
`<input type="file" wire:model="avatar" accept="image/*">`, and pair it with
`<div wire:loading wire:target="avatar">`. The data property becomes a `FileUpload` object held in
temporary storage. It stays there until an action commits it, which is what makes an upload behave
like any other staged edit.

```boxlang
// wires/PhotoUpload.bx, inside the class
function save() {
    if ( isSimpleValue( data.avatar ) ) {
        return;
    }
    if ( data.avatar.isImage() ) {
        var storedPath = data.avatar.store( expandPath( "./uploads" ) );
    }
    data.avatar = "";
}
```

| `FileUpload` method | Purpose |
|---------------------|---------|
| `store( path )` | Move the file to permanent storage. Returns the absolute stored path |
| `isImage()` | Type test before processing |
| `getSize()` / `getMIMEType()` / `getMeta()` | Inspect without reading the bytes |
| `get()` | The binary content |
| `getBase64()` / `getBase64Src()` | Encoded content, and a `data:` URL for an `<img>` tag |
| `getPreviewURL()` | Preview an image before committing |
| `getTemporaryStoragePath()` / `getMetaPath()` | The staged locations |
| `destroy()` | Discard the staged file |

`store()` takes **one** argument. Pass a directory and the original filename is kept. Pass a full
file path and the file is renamed. Missing directories are created either way.

### Do not call destroy() after store()

The CBWIRE documentation shows `store()` followed by `destroy()`. **Do not copy that pattern.**
`store()` repoints the object's internal path at the file it just wrote. `destroy()` then deletes
whatever that path points at, so the pair deletes the file you just saved.

```boxlang
var storedPath = data.avatar.store( expandPath( "./uploads" ) );
data.avatar.destroy();   // WRONG, deletes the file store() just wrote
data.avatar = "";        // RIGHT, store() already moved it
```

`destroy()` is still the right call when you are **discarding** an upload the user never committed.

### Upload errors

`onUploadError( property, errors, multiple )` fires on any non-2xx response. In 4.x an upload error
threw instead. `errors` is null unless the status was 422, in which case it holds the response
body.

## JavaScript Integration

### cbwire:script and cbwire:assets

These are the supported way to attach JavaScript to a component. Both sit **outside** the root
element, as siblings. CBWIRE lifts them out of the template before rendering, so they never become
a second root.

```html
<!--- wires/DatePicker.bxm --->
<bx:output>
<div>
    <input type="text" data-picker>
</div>

<cbwire:assets>
    <script src="https://cdn.jsdelivr.net/npm/pikaday/pikaday.js" defer></script>
</cbwire:assets>

<cbwire:script>
    <script>
        new Pikaday( { field: $wire.$el.querySelector( '[data-picker]' ) } );
    </script>
</cbwire:script>
</bx:output>
```

Why use these over a hand-rolled script tag: `$wire` is already in scope, so no lookup, no id
matching, no init event. Assets load once per page while scripts run once per component instance.
And Livewire guarantees assets are on the page before scripts evaluate. A plain `<script>` tag has
none of that and **must** live inside the root element, or the component gets two roots.

### The $wire object

Inside a component's scripts, `$wire` is a JavaScript view of the server component.

```javascript
$wire.counter                    // read a data property, or $wire.$get( 'counter' )
$wire.$set( 'counter', 5 )       // write one. Also $toggle, $call, $refresh
$wire.$dispatch( 'saved', { id: 2 } )   // also $dispatchTo( 'UserList', 'refresh', {} )
$wire.$watch( 'counter', ( value, old ) => {} )
$wire.$upload( 'avatar', file, finish, error, progress )
$wire.$el                        // root DOM element. Also $id, $parent
```

### Holding a reference from an external script

When the code lives outside the component, in a bundled file for example, register on
`livewire:init` and match by id.

```html
<script>
document.addEventListener( "livewire:init", () => {
    Livewire.hook( "component.init", ( { component, cleanup } ) => {
        if ( component.id === "#_id#" ) {
            window._myWidget = component.$wire;
            // all initialization for this component goes here
        }
    } );
} );
</script>
```

**Never use `DOMContentLoaded`.** It fires before the component registers, so `Livewire.find()`
returns `undefined` there. The id check is required, because `component.init` fires for every
component on the page. `#_id#` is the current component's id, pre-loaded into the template scope.
Keep the global, so later code calls `window._myWidget.someAction()` instead of finding the
component again.

CBWIRE re-fires the Livewire events under its own names, so `cbwire:init`, `cbwire:initialized`,
and `cbwire:navigated` also work, and `window.cbwire` aliases `window.Livewire`. Other hooks:
`element.init`, the `morph.*` family, `commit`, `commit.prepare`, and `request`. The `request` hook
is how you replace the default "page expired" dialog on a 419.

### wire:ignore versus wire:ignore.self

- **`wire:ignore`** protects an entire subtree. Use it for anything holding browser state the
  morph would destroy: a date picker, a rich text editor, a drop zone with pending files.
- **`wire:ignore.self`** protects only the element's own attributes and still updates its
  children. Use it when a client library owns a class on the container, such as an active or
  collapsed state. The inputs inside still belong to the component.

## Advanced API

### js( expression )

Queues JavaScript to run in the browser after the response lands, as in
`js( "myToast.show('Saved')" )`. It is the shortest path from an action to a client side
side-effect, with no template plumbing.

**It must be a single expression.** Alpine splices the queued string into an assignment, so
anything that cannot legally follow an `=` is a syntax error. That rules out a string starting with
`for`, `while`, `var`, `function`, `return`, `throw`, or `try`. A leading `if (...)`, `let`, or
`const` is special-cased and does work. Semicolon-separated statements parse, but only the first
value is captured. Build a helper function and call that instead.

`js()` is documented only in the 4.0 release notes, not the CBWIRE 5 docs. If a call produces a
browser console `TypeError` instead of running, the server and the bundled Livewire build disagree
about the payload shape. Check your installed version before debugging the expression.

### reset( property ) and resetExcept( property )

Restore data properties to the values they were declared with. Both accept a name, a list, an
array, or nothing at all to mean every property: `reset()`, `reset( "search,page" )`,
`resetExcept( "perPage" )`.

### Lazy loading and placeholder()

Defer a component until it scrolls into view with `#wire( name = "SalesReport", lazy = true )#`.
Useful for anything expensive that is not needed for first contentful paint. A component can opt
into this itself with `lazyLoad = true`, and into render isolation with `isolate = true`.

```boxlang
function placeholder() {
    return "<div class='skeleton'>Loading report...</div>";
}
```

**The placeholder must use the same root element type as the component.** A placeholder that
returns a `<div>` in front of a component whose root is a `<span>` renders wrong. `placeholder()`
can also return a ColdBox view through `view( "spinner" )`.

### Nested components

Call `wire()` from a parent template. Give every child in a loop a `wire:key` so the morph matches
children by identity rather than position:
`#wire( name = "RowEditor", params = { "id" : row.id }, key = "row-#row.id#" )#`.

### stream( target, content, replace )

Push partial output to the page during a long-running action, paired with `wire:stream` in the
template. All three arguments are required. `replace` has no default, and calling `stream()` with
two arguments errors.

### The secured annotation

Component-level and method-level authorization, evaluated through cbSecurity. The annotation is
only active when cbSecurity is installed. Without it, use `onSecure()`. This is the one place where
the two language modes differ by more than a mechanical substitution, so both are shown.

```boxlang
// wires/AdminPanel.bx
@secured
class extends="cbwire.models.Component" {
    @secured( "admin" )
    function deleteUser( userId ) {}
}
```

```cfml
// wires/AdminPanel.cfc
component extends="cbwire.models.Component" secured {
    function deleteUser( userId ) secured="admin" {}
}
```

A bare `secured` means "must be logged in". A string is passed to cbSecurity as a permission check,
and a comma list passes when the user holds any one of them. `@secured(false)` allows everyone. The
`cbsecurity` skill covers what a permission string resolves to.

### redirect( url, useNavigate )

Navigate away from an action. Pass `true` for the second argument to use SPA navigation instead of
a full page load.

### Interception points

Six ColdBox interception points, so cross-cutting concerns need not live in the components:
`onCBWIREMount`, `preCBWIRERender`, `onCBWIRERender`, `preCBWIREUpdate`, `onCBWIREUpdate`, and
`onCBWIRESecureFail`. Each carries the wire instance, its name, and its data.

## Behavior and Gotchas

Each of these is hard to diagnose from the symptom alone.

### wire:dirty compares snapshots, not history

It measures the canonical snapshot against the reactive one. **Every commit makes the current state
canonical.** Dirty silently resets after any server round trip, including one the user never
thought of as a save. Do not use it as a running "has this form been touched" flag across requests.

### wire:target on a wire:dirty element must name a property

`wire:loading` and `wire:dirty` both read `wire:target`, but differently. `wire:dirty` resolves the
target against the component's data, so it must be a **data property path**. Point it at an action
name and it silently never fires, with no error.

```html
<div wire:dirty wire:target="save">Unsaved</div>   <!--- WRONG, "save" is an action --->
<div wire:dirty wire:target="name">Unsaved</div>   <!--- RIGHT, a data property --->
```

A `wire:model` on the same element is added as a target automatically, and `wire:target` accepts a
comma-separated list of paths.

### Livewire wipes any class a wire:dirty directive names

A reset runs after the morph and returns every `wire:dirty` element to its clean state. So a class
the server renders **must not** be a class the directive also toggles, or it vanishes a moment
after it appears. Give each signal its own class and combine them in CSS.

```html
<div class="note is-dirty" wire:dirty.class="is-dirty">Unsaved</div>
<!--- WRONG, the post-commit reset wipes the server's copy of is-dirty --->

<div class="note is-dirty-server" wire:dirty.class="is-dirty-wire">Unsaved</div>
<!--- RIGHT, each signal owns its own class --->
```

### A file upload is itself a commit

The upload round trip makes the new value canonical, so `wire:dirty` reports **clean** while the
file is still uncommitted to storage. Any "you have a staged file" indicator has to come from the
server, which can see the `FileUpload` object.

### The template scope is pre-loaded

CBWIRE injects a lot into the template scope before including it:

- every data key as a bare variable, plus a `get<Key>()` for each
- the `data` and `args` structs
- every public method and every computed function
- `event`, the ColdBox request context
- `view()`, `_id`, and `validation` when cbvalidation is installed

An unprefixed local variable in a wire template can silently overwrite one of them. Prefix locals,
or derive the value from a component method instead. Params passed through `wire()` are applied
**last** and overwrite same-named data properties. The one exception is `event`, assigned after the
params, which therefore wins over a param of that name.

### Components are transients

A fresh instance per request, so memoizing in the `variables` scope is safe within a render and
cannot leak across requests. This matters when a template asks for the same derived value many
times. Key the memo on whatever it derives from if an action can change that value mid-request.

```boxlang
function getSettings() {
    if ( !variables.keyExists( "_settingsMemo" ) ) {
        variables._settingsMemo = settingsService.load( data.recordId );
    }
    return variables._settingsMemo;
}
```

### The single root element is no longer enforced

CBWIRE used to throw a clear error for a template with two root elements. That check now returns
immediately without validating. The rule still holds, because CBWIRE injects the wire attributes
into the first tag it finds. Breaking it now gives a confusing morph failure instead of a helpful
message.

### wire:model on a file input already has a native listener

Livewire binds to the input's own `change` event, so a browse pick already reaches Livewire with no
help. Code that sets `input.files` and dispatches a synthetic `change` fires twice on a browse and
uploads the same file twice. A drag and drop never touches the input, so that path does need the
synthetic event. Fire only when the file set actually changed, and guard against re-entry, because
your own listener will hear the event you dispatch.

```javascript
const changed = !sameFiles( input.files, dt.files );
input.files = dt.files;
if ( changed ) input.dispatchEvent( new Event( "change", { bubbles: true } ) );
```

### Alpine x-if destroys nested components

A wire component inside an `x-if` template is torn out of the DOM when the condition flips, and
Livewire then reports "unable to find component". Use `x-show`, or `wire:show`, which keeps the
element in the DOM.

## Testing

Extend `cbwire.models.BaseWireTest` and call `wire()` for a chainable test instance. The
`testbox/bdd` skill covers the `describe` and `it` structure itself.

```boxlang
// tests/specs/TaskListTest.bx
class extends="cbwire.models.BaseWireTest" {
    function run() {
        describe( "TaskList", function() {
            beforeEach( function( currentSpec ) { setup(); } );

            it( "clears the tasks", function() {
                wire( "TaskList" )
                    .data( "tasks", [ "task1" ] )
                    .see( "task1" )
                    .call( "clearTasks" )
                    .dontSee( "task1" );
            } );
        } );
    }
}
```

| Method | Purpose |
|--------|---------|
| `data( key, value )` | Set a data property, or pass a struct of them |
| `computed( key, closure )` | Stub a computed property |
| `toggle( key )` | Flip a boolean data property |
| `call( method, params )` | Run an action |
| `emit( event, params )` | Fire an event and run the component's listeners |
| `see( text )` / `dontSee( text )` | Assert on the rendered HTML |
| `seeData( key, value )` / `dontSeeData( key, value )` | Assert on a data property |

`setup()` and `execute()` come from ColdBox's `BaseTestCase`, which `BaseWireTest` extends. Call
`setup()` in `beforeEach()` so each example starts from a clean request context.

### The render cache makes repeated renders return stale HTML

CBWIRE caches rendered content on the component instance and never clears it. The framework relies
on this, because a parent renders each non-lazy child twice. For tests, a second render of the same
instance returns the **first** render's HTML, whatever changed in between. That produces both false
passes and false regressions. Keep one render per test block. To assert on state after an action,
use `seeData()` rather than a second `see()`.

### Test rendering is HTML only

No JavaScript runs in a test render. Directives, morph behavior, `wire:dirty`, uploads, and any
client integration are all **unverifiable** by the suite. A green suite says the markup rendered,
not that the component behaves. Say so when reporting results, and verify client-side work in a
browser.

## Best Practices

- **Fetch large reference lists in the template** through `getInstance()`, never in `data`.
  Everything in `data` travels in every payload, so keep it flat and small
- **Encode output with `encodeForHTML()`.** CBWIRE does not sanitize for you
- **Use `wire:loading`** for feedback on slow actions, `dispatchTo()` for targeted messaging, and
  keep heavy work in the service layer where it can be cached
- **For tabs, accordions, and wizards, consider rendering every pane** and toggling visibility on
  the client. Server-side switching sends a whole subtree back and forces the morph to rebuild it,
  the case morphdom handles worst. The tradeoff is a larger first render, so weigh it
- **Reach for plain JavaScript** when a component would manage hundreds of nodes. CBWIRE is the
  wrong tool for a large tree

## Documentation

- cbwire: https://github.com/coldbox-modules/cbwire
- cbwire docs: https://cbwire.ortusbooks.com
