# ColdBox Skills

> AI agent skills for [ColdBox](https://www.coldbox.org/) — the leading enterprise MVC framework for BoxLang and CFML.

This repository provides reusable **AI skills** for ColdBox development, compatible with any agent that supports the [skills.sh](https://skills.sh) open standard — including Claude Code, Cursor, Copilot, and more.

Skills inject ColdBox domain knowledge directly into your AI agent so it can write accurate, idiomatic ColdBox code for you.

---

## What Are Skills?

Skills are Markdown files (`SKILL.md`) that give your AI agent expert-level context on a specific topic. They are discovered and loaded automatically by the agent when relevant — a query about REST APIs loads the REST skill, a question about testing loads the testing skill, and so on.

Each skill in this repo contains:

- Framework patterns and idioms with working code examples
- API references and configuration samples
- ColdBox conventions and best practices
- Integration patterns with the ColdFusion/BoxLang ecosystem

---

## Quick Install

Requires [Node.js](https://nodejs.org) — no installation needed, just `npx`:

```bash
# Install ALL ColdBox skills
npx skills add ortus-boxlang/coldbox-skills
```

That's it. Your AI agent now has ColdBox expertise.

---

## Categories

### `coldbox` — Core Framework Skills

For developers building ColdBox applications: handlers, routing, views, interceptors, REST APIs, configuration, and the event model.

```bash
npx skills add ortus-boxlang/coldbox-skills/coldbox
```

| Skill | What It Covers |
|-------|----------------|
| [`handler-development`](./coldbox/handler-development/SKILL.md) | Controllers (handlers), CRUD actions, dependency injection, preHandler/postHandler, secured handlers, REST handlers |
| [`routing-development`](./coldbox/routing-development/SKILL.md) | Router.cfc, RESTful resource routes, route groups, constraints, HTTP method mapping, named routes, subdomain routing |
| [`event-model`](./coldbox/event-model/SKILL.md) | Request context object (rc/prc), collection management, view rendering, redirects, HTTP headers, event caching |
| [`rest-api-development`](./coldbox/rest-api-development/SKILL.md) | RestHandler, CRUD endpoints, validation, error handling, authentication, versioning, rate limiting |
| [`interceptor-development`](./coldbox/interceptor-development/SKILL.md) | Cross-cutting concerns, framework interception points, security/logging/caching interceptors, custom events |
| [`coldbox-configuration`](./coldbox/coldbox-configuration/SKILL.md) | ColdBox.cfc, application settings, environment detection, module settings, conventions |
| [`view-rendering`](./coldbox/view-rendering/SKILL.md) | Views, layouts, partials, view helpers, rendering data, template caching |
| [`layout-development`](./coldbox/layout-development/SKILL.md) | Layouts, nested layouts, layout modules, conditional layouts |
| [`module-development`](./coldbox/module-development/SKILL.md) | Module structure, ModuleConfig.cfc lifecycle, module routes, settings, inter-module communication |
| [`coldbox-request-context`](./coldbox/coldbox-request-context/SKILL.md) | Request/response lifecycle, flash scope, collection manipulation, request metadata |
| [`coldbox-flash-messaging`](./coldbox/coldbox-flash-messaging/SKILL.md) | Flash RAM, message types (success/error/info/warning), persistence strategies |
| [`cache-integration`](./coldbox/cache-integration/SKILL.md) | ColdBox event caching, view caching, handler-level cache control, cache keys |

### `testing` — TestBox & Spec Writing Skills

For writing unit, integration, and BDD tests with TestBox.

```bash
npx skills add ortus-boxlang/coldbox-skills/testing
```

| Skill | What It Covers |
|-------|----------------|
| [`testing-bdd`](./testing/testing-bdd/SKILL.md) | BDD specs, `describe`/`it`/`expect`, matchers, `beforeEach`/`afterEach`, `xdescribe`/`xit` |
| [`testing-unit`](./testing/testing-unit/SKILL.md) | Unit tests, `TestCase` pattern, assertions, test lifecycle, isolated component testing |
| [`testing-integration`](./testing/testing-integration/SKILL.md) | ColdBox integration specs, `execute()` framework calls, endpoint testing |
| [`testing-handler`](./testing/testing-handler/SKILL.md) | Handler action testing, mock injection, rc/prc assertions, view/data assertions |
| [`testing-mocking`](./testing/testing-mocking/SKILL.md) | MockBox, `createMock`, `createStub`, method mocking, argument matchers |
| [`testing-fixtures`](./testing/testing-fixtures/SKILL.md) | Test fixture patterns, data seeding, factory patterns, database test helpers |
| [`testing-coverage`](./testing/testing-coverage/SKILL.md) | Code coverage configuration, coverage reports, CI integration |
| [`testing-ci`](./testing/testing-ci/SKILL.md) | Running TestBox in GitHub Actions, test reporters, CI/CD pipeline integration |

### `security` — Authentication & Authorization Skills

For implementing authentication, authorization, JWT, and security patterns.

```bash
npx skills add ortus-boxlang/coldbox-skills/security
```

| Skill | What It Covers |
|-------|----------------|
| [`authentication`](./security/authentication/SKILL.md) | CBAuth login/logout, user service, session management, remember-me tokens |
| [`authorization`](./security/authorization/SKILL.md) | CBSecurity permissions, roles, firewall rules, annotation-based security |
| [`jwt-development`](./security/jwt-development/SKILL.md) | JWT tokens, refresh tokens, token storage, API authentication with CBSecurity |
| [`security-implementation`](./security/security-implementation/SKILL.md) | Full CBSecurity setup, configuration, user validators |
| [`csrf-protection`](./security/csrf-protection/SKILL.md) | CSRF tokens, form integration, AJAX protection with cbcsrf |
| [`rbac-patterns`](./security/rbac-patterns/SKILL.md) | Role-based access control, permission hierarchies, dynamic authorization |
| [`api-authentication`](./security/api-authentication/SKILL.md) | API key auth, Bearer token auth, OAuth2 integration patterns |
| [`sso-integration`](./security/sso-integration/SKILL.md) | CBSSO, SAML, OAuth2/OIDC, social login integration |
| [`passkeys-integration`](./security/passkeys-integration/SKILL.md) | WebAuthn passkeys, FIDO2, cbsecurity-passkeys module |

### `orm` — Database & ORM Skills

For database access using QB Query Builder, Quick ORM, CBORM, and database migrations.

```bash
npx skills add ortus-boxlang/coldbox-skills/orm
```

| Skill | What It Covers |
|-------|----------------|
| [`qb`](./orm/qb/SKILL.md) | QB query builder, fluent queries, joins, aggregates, pagination, raw expressions |
| [`cborm`](./orm/cborm/SKILL.md) | CBORM active record, criteria builder, entity services, Hibernate mappings |
| [`quick-orm`](./orm/quick-orm/SKILL.md) | Quick active record, relationships (hasMany/belongsTo/etc.), scopes, eager loading |
| [`database-migrations`](./orm/database-migrations/SKILL.md) | cfmigrations schema builder, up/down migrations, seeders, CI database setup |

### `wirebox` — Dependency Injection Skills

For WireBox DI configuration, injection patterns, AOP, and object lifecycle.

```bash
npx skills add ortus-boxlang/coldbox-skills/wirebox
```

| Skill | What It Covers |
|-------|----------------|
| [`dependency-injection`](./wirebox/dependency-injection/SKILL.md) | Property/constructor/setter injection, DSL annotations, binder configuration, scopes, providers |
| [`aop-programming`](./wirebox/aop-programming/SKILL.md) | Aspect-oriented programming, method interceptors, before/after/around advice |

### `cachebox` — CacheBox Caching Skills

For CacheBox cache providers, patterns, and cache management.

```bash
npx skills add ortus-boxlang/coldbox-skills/cachebox
```

| Skill | What It Covers |
|-------|----------------|
| [`caching-patterns`](./cachebox/caching-patterns/SKILL.md) | Cache configuration, providers (RAM/Redis/Couchbase), getOrSet pattern, cache invalidation, event caching |

### `logbox` — LogBox Logging Skills

For LogBox appenders, log levels, categories, and structured logging.

```bash
npx skills add ortus-boxlang/coldbox-skills/logbox
```

| Skill | What It Covers |
|-------|----------------|
| [`logging-patterns`](./logbox/logging-patterns/SKILL.md) | LogBox configuration, appenders (file/console/email/DB), log levels, category loggers, structured logging |

---

## Install Individual Skills

```bash
# Specific skill
npx skills add ortus-boxlang/coldbox-skills/coldbox/handler-development
npx skills add ortus-boxlang/coldbox-skills/testing/testing-bdd
npx skills add ortus-boxlang/coldbox-skills/security/authentication
npx skills add ortus-boxlang/coldbox-skills/orm/qb
```

---

## MCP Servers

This repository also includes an `.mcp.json` file with all major Ortus/ColdBox GitBook documentation servers. When installed via Claude Code or other supported agents, you get immediate access to live documentation for:

- ColdBox, CommandBox, TestBox, WireBox, CacheBox, LogBox
- QB, Quick ORM, CBORM, CFMigrations
- CBSecurity, CBAuth, CBValidation
- CBWire, CBQ, CBMailServices, CBFS, CBDebugger
- ContentBox, Relax, and more

---

## Resources

- [ColdBox Documentation](https://coldbox.ortusbooks.com/)
- [TestBox Documentation](https://testbox.ortusbooks.com/)
- [WireBox Documentation](https://wirebox.ortusbooks.com/)
- [CacheBox Documentation](https://cachebox.ortusbooks.com/)
- [LogBox Documentation](https://logbox.ortusbooks.com/)
- [BoxLang Documentation](https://boxlang.ortusbooks.com/)
- [Ortus Solutions](https://www.ortussolutions.com/)
