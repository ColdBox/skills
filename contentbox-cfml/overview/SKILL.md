---
name: contentbox-cfml-overview
description: "Use this skill when understanding ContentBox architecture, core modules, service layer, multi-site behavior, security foundations, and the recommended extension points for building or maintaining ContentBox solutions."
applyTo: "**/*.{cfc,cfm,cfml}"
---

# ContentBox CMS Overview (CFML)

ContentBox is a professional open-source hybrid modular CMS (Content Management System) built on ColdBox. It supports both traditional CMS and headless architectures, running on CFML engines (Lucee, Adobe ColdFusion) and BoxLang.

## Architecture

ContentBox follows a layered modular architecture:

```
/ (Root App)                    ← Host ColdBox application, local dev shell
├── modules/contentbox/         ← Core ContentBox engine
│   ├── ModuleConfig.cfc        ← Core module configuration
│   ├── models/                 ← Domain entities and services
│   ├── widgets/                ← Core widgets (19 built-in)
│   ├── themes/default/         ← Default theme
│   ├── migrations/             ← Database migrations
│   └── modules/                ← Sub-modules
│       ├── contentbox-admin/   ← Admin interface (entry: cbadmin)
│       ├── contentbox-api/     ← REST API (v1)
│       ├── contentbox-ui/      ← Public-facing UI
│       └── contentbox-deps/    ← Dependencies (cborm, etc.)
├── modules_app/contentbox-custom/  ← Customization layer
│   ├── _themes/                ← Custom user themes
│   ├── _widgets/               ← Custom user widgets
│   ├── _modules/               ← Custom user modules
│   └── _content/               ← Custom content
└── modules/                    ← Standalone ColdBox modules
```

## Key Concepts

### Hybrid CMS

ContentBox is both a **traditional CMS** and a **headless CMS**:

- **Traditional**: Full admin UI, theme system, widget rendering, page/entry management
- **Headless**: REST API (v1) with JWT authentication for decoupled frontends

### Modular Design

Everything in ContentBox is a module:

- Core functionality is in `modules/contentbox/`
- Admin, API, and UI are separate sub-modules
- Custom code goes in `modules_app/contentbox-custom/`
- Third-party modules go in `modules/`

### ORM & Database

- Uses **CFML ORM** (Hibernate) with `this.ormEnabled = true` in `Application.cfc`
- Database migrations via **cfmigrations**
- Entities extend `BaseEntity` with UUID primary keys
- Soft deletes with `isDeleted` flag

### Multi-Site Support

- Multiple sites from a single installation
- Domain-based site resolution
- Site-specific content, themes, settings, menus

### Security

- **cbSecurity** for RBAC (Role-Based Access Control)
- Database-driven security rules
- BCrypt password hashing
- Two-factor authentication (pluggable providers)
- Rate limiting for brute-force protection
- CSRF protection

## Core Services

| Service | DSL | Purpose |
|---------|-----|---------|
| `entryService` | `entryService@contentbox` | Blog entry CRUD |
| `pageService` | `pageService@contentbox` | Page CRUD |
| `contentService` | `contentService@contentbox` | Unified content service |
| `authorService` | `authorService@contentbox` | Author management |
| `categoryService` | `categoryService@contentbox` | Category CRUD |
| `menuService` | `menuService@contentbox` | Menu management |
| `settingService` | `settingService@contentbox` | Settings management |
| `securityService` | `securityService@contentbox` | Authentication/authorization |
| `widgetService` | `widgetService@contentbox` | Widget registry |
| `themeService` | `themeService@contentbox` | Theme management |
| `siteService` | `siteService@contentbox` | Multi-site management |
| `mediaService` | `mediaService@contentbox` | Media management |
| `contentStoreService` | `contentStoreService@contentbox` | Key-value content |
| `twoFactorService` | `twoFactorService@contentbox` | 2FA management |
| `cb` | `CBHelper@contentbox` | CBHelper for theme/UI |

## CBHelper

The `CBHelper@contentbox` is the primary API for theme and UI development:

```cfml
property name="cb" inject="CBHelper@contentbox";

// Site
cb.site()           // Current site entity
cb.siteURL()        // Site base URL
cb.siteName()       // Site name

// Content
cb.getContent()     // Current content (entry/page)
cb.entryURL( entry )
cb.pageURL( page )
cb.categoryURL( category )

// Theme
cb.getThemeSetting( "name" )

// Widgets
cb.widget( "WidgetName", { args } )

// Rendering
cb.renderCollection( template, collection )
cb.renderView( view )

// Menus
cb.menu( "main" )

// Media
cb.mediaURL( media )
cb.mediaURL( media, "thumbnail" )

// RSS
cb.rssURL()
cb.rssCommentsURL()

// Search
cb.searchURL()
cb.searchURL( "query" )

// Subscriptions
cb.subscribeURL()
cb.unsubscribeURL()
```

## Interception Points

### Core

| Point | When |
|-------|------|
| `cb_onContentRendering` | Before content is rendered |
| `cb_onContentStoreRendering` | Before ContentStore content is rendered |

### Admin (cbadmin_*)

Layout injection, content editor, content lifecycle, author management, dashboard, settings, and more. See the admin extension skill for the full list.

## Content Types

| Type | Description |
|------|-------------|
| **Entries** | Blog posts with dates, categories, comments |
| **Pages** | Static pages with hierarchical structure |
| **ContentStore** | Key-value content blocks |

## Widgets

Widgets extend `BaseWidget` and implement `renderIt()`. They are discovered from:
1. Active theme widgets (highest priority)
2. Custom widgets (`_widgets/`)
3. Core widgets
4. Module widgets

## Themes

Themes define the visual presentation. Required files:
- `Theme.cfc` — metadata, settings, lifecycle callbacks
- `layouts/blog.cfm`, `layouts/pages.cfm` — mandatory layouts
- `views/index.cfm`, `entry.cfm`, `page.cfm`, `archives.cfm`, `error.cfm` — mandatory views

## API

REST API v1 at `/api/v1/` with JWT authentication:
- Entries, Pages, Categories, Authors, Menus, Sites, Settings, ContentStore
- CRUD operations with pagination, filtering, sorting
- Memento-based JSON responses

## Commands

```bash
# Start local server
box server start

# Migrations
box run-script contentbox:migrate:create name=YourMigration
box run-script contentbox:migrate
box run-script contentbox:migrate:up
box run-script contentbox:migrate:down
box run-script contentbox:migrate:fresh

# Format code
box run-script format
box run-script format:check

# Frontend builds
npm run dev
npm run watch
npm run prod
npm run build-dev
npm run build-prod
```

## Testing

- Web tests: `http://127.0.0.1:8589/tests/runner.cfm`
- API tests: `http://127.0.0.1:8589/tests/runner-api.cfm`
- Base classes: `tests/resources/BaseTest.cfc`, `tests/resources/BaseApiTest.cfc`

## Engine Compatibility

ContentBox supports three engines:
- **Lucee 5+**
- **Adobe ColdFusion 2018+**
- **BoxLang**

Server configs: `server-lucee@*.json`, `server-adobe@*.json`, `server-boxlang@*.json`

## Key References

- Documentation: https://contentbox.ortusbooks.com/
- Source: https://github.com/Ortus-Solutions/ContentBox
- Community: https://community.ortussolutions.com/c/communities/contentbox/15
- Bug Tracker: https://ortussolutions.atlassian.net/browse/CONTENTBOX
