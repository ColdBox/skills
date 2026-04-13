---
name: cbpaginator
description: >
  Use this skill when implementing server-side pagination in ColdBox/BoxLang using cbpaginator. Covers
  installation, Paginator service configuration, generating pagination metadata for APIs and views,
  integrating with QB QueryBuilder, rendering Bootstrap pagination controls, and building pageable
  collection responses.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# CBPaginator Skill

## When to Use This Skill

Load this skill when:
- Adding pagination to a listing handler or REST API endpoint
- Generating page metadata (total, per_page, current_page, last_page, from, to)
- Integrating pagination with QB or qb QueryBuilder
- Building Bootstrap or Tailwind page-number navigation in views
- Creating consistent `{ data: [], pagination: {} }` API responses

## Installation

```bash
box install cbpaginator
```

## Core API

### Injection

```js
property name="paginator" inject="Pagination@cbpaginator";
```

### Create a Paginator

```js
var page = paginator
    .newPagination()
    .setTotalRecords( totalCount )
    .setMaxRows(      rc.perPage ?: 25 )
    .setPage(         rc.page    ?: 1 )
    .calculate()

// Available properties after calculate():
page.getPage()        // current page number
page.getMaxRows()     // per-page count
page.getTotalRecords()
page.getTotalPages()
page.getOffset()      // SQL OFFSET value
page.getFrom()        // first record on this page
page.getTo()          // last record on this page
page.hasPreviousPage()
page.hasNextPage()
```

### Shorthand — paginate()

```js
var page = paginator.paginate(
    page         = rc.page    ?: 1,
    maxRows      = rc.perPage ?: 25,
    totalRecords = userService.count()
)

var users = userService.list(
    max    = page.getMaxRows(),
    offset = page.getOffset()
)
```

## Production Patterns

### REST API — Pageable Collection

```js
function index( event, rc, prc ) {
    var page  = paginator.paginate(
        page         = rc.page    ?: 1,
        maxRows      = rc.perPage ?: 25,
        totalRecords = userService.count()
    )

    event.renderData( type = "json", data = {
        data       : userService.list( max = page.getMaxRows(), offset = page.getOffset() ),
        pagination : {
            total        : page.getTotalRecords(),
            per_page     : page.getMaxRows(),
            current_page : page.getPage(),
            last_page    : page.getTotalPages(),
            from         : page.getFrom(),
            to           : page.getTo()
        }
    } )
}
```

### QB Integration

```js
// Inject QB
property name="qb"        inject="QueryBuilder@qb";
property name="paginator" inject="Pagination@cbpaginator";

function index( event, rc, prc ) {
    var total = qb.from( "users" ).where( "isActive", true ).count()

    var page  = paginator.paginate(
        page         = rc.page ?: 1,
        maxRows      = 20,
        totalRecords = total
    )

    prc.users = qb.from( "users" )
        .where( "isActive", true )
        .orderByDesc( "createdAt" )
        .limit(  page.getMaxRows() )
        .offset( page.getOffset() )
        .get()

    prc.pagination = page
}
```

### View — Bootstrap 5 Pagination

```cfml
<cfif prc.pagination.getTotalPages() gt 1>
    <nav>
        <ul class="pagination">
            <!--- Previous --->
            <li class="page-item <cfif NOT prc.pagination.hasPreviousPage()>disabled</cfif>">
                <a class="page-link"
                   href="#event.buildLink( 'users.index' )#?page=#prc.pagination.getPage() - 1#">
                    &laquo;
                </a>
            </li>

            <!--- Page numbers --->
            <cfloop from="1" to="#prc.pagination.getTotalPages()#" index="i">
                <li class="page-item <cfif i eq prc.pagination.getPage()>active</cfif>">
                    <a class="page-link"
                       href="#event.buildLink( 'users.index' )#?page=#i#">#i#</a>
                </li>
            </cfloop>

            <!--- Next --->
            <li class="page-item <cfif NOT prc.pagination.hasNextPage()>disabled</cfif>">
                <a class="page-link"
                   href="#event.buildLink( 'users.index' )#?page=#prc.pagination.getPage() + 1#">
                    &raquo;
                </a>
            </li>
        </ul>
    </nav>
</cfif>
```

## Best Practices

- **Always validate** `rc.page` and `rc.perPage` are positive integers before passing to `paginate()`
- **Cap `maxRows`** at a reasonable limit (e.g., 100) to prevent unbounded queries
- **Run the count query before paginate()** — ensure accurate total records
- **Return offset from the paginator**, not calculated manually, to cover edge-case page-boundary arithmetic
- **Clamp page to valid range** — if the requested page exceeds totalPages, redirect or return the last page
- **Include pagination metadata in every list API response** for consistent consumer contracts

## Documentation

- cbpaginator: https://github.com/coldbox-modules/cbpaginator
