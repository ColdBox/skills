# ColdBox Skills

> AI agent skills for [ColdBox](https://www.coldbox.org/) and the surrounding Ortus ecosystem, including TestBox, WireBox, CacheBox, LogBox, ORM tooling, security, DocBox, and 40+ Ortus/ColdBox modules.

This repository is a [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces) that provides reusable skills for ColdBox and related tooling.

---

## Install via Claude Code

Add the marketplace and install the plugin:

```bash
/plugin marketplace add ortus-boxlang/coldbox-skills
/plugin install coldbox-agent-skills@coldbox-skills
```

Or using the CLI:

```bash
claude plugin marketplace add ortus-boxlang/coldbox-skills
claude plugin install coldbox-agent-skills@coldbox-skills
```

## ColdBox CLI

```bash
# Install the ColdBox CLI if you haven't already
box install coldbox-cli

# Install AI integration into your app
# This reads your app, box.json and installs skills based on your stack and preferences
coldbox ai install

# Skills Management

# List installed skills
coldbox ai skills list
# Add a skill
coldbox ai skills add Ortus-Solutions/skills/vuejs-expert
# Remove a skill
coldbox ai skills remove vuejs-expert
```

All skills are installed at `.agents/skills/` in your project.

---

## Manifest Coverage

The [`.claude-plugin/plugin.json`](./.claude-plugin/plugin.json) registers these skill categories:

- `coldbox` — Core ColdBox framework skills
- `testbox` — Comprehensive TestBox skills
- `security` — Authentication, authorization, CSRF, SSO, passkeys
- `wirebox` — WireBox dependency injection
- `cachebox` — CacheBox standalone caching
- `logbox` — LogBox logging
- `modules` — 40+ Ortus/ColdBox module skills
- `docbox` — DocBox documentation generation

---

## Categories

### `coldbox` — Core Framework Skills

| Skill | What It Covers |
|---|---|
| [`handler-development`](./coldbox/handler-development/SKILL.md) | Handlers, CRUD actions, REST handlers, dependency injection |
| [`routing-development`](./coldbox/routing-development/SKILL.md) | Router configuration, named routes, constraints, route groups |
| [`event-model`](./coldbox/event-model/SKILL.md) | Event lifecycle, `event`, `rc`, `prc`, rendering and redirects |
| [`rest-api-development`](./coldbox/rest-api-development/SKILL.md) | RestHandler patterns, API validation, versioning, error handling |
| [`interceptor-development`](./coldbox/interceptor-development/SKILL.md) | Interceptors, framework interception points, custom events |
| [`configuration`](./coldbox/configuration/SKILL.md) | `ColdBox.cfc`, settings, conventions, environment configuration |
| [`app-layouts`](./coldbox/app-layouts/SKILL.md) | Flat, BoxLang, and Modern app layout selection, directory structures, engine compatibility, and migration guidance |
| [`view-rendering`](./coldbox/view-rendering/SKILL.md) | Views, layouts, partials, helpers, data rendering |
| [`layout-development`](./coldbox/layout-development/SKILL.md) | Layout patterns, nested layouts, conditional layouts |
| [`module-development`](./coldbox/module-development/SKILL.md) | Module structure, `ModuleConfig.cfc`, routes, settings |
| [`request-context`](./coldbox/request-context/SKILL.md) | Request context, flash scope, collections, metadata |
| [`flash-messaging`](./coldbox/flash-messaging/SKILL.md) | Flash RAM, message persistence, success/error/info/warning patterns |
| [`cache-integration`](./coldbox/cache-integration/SKILL.md) | Event and view caching, cache keys, cache invalidation |
| [`database-migrations`](./coldbox/database-migrations/SKILL.md) | Schema migrations with cfmigrations and CommandBox CLI: create, run, rollback, and seed |
| [`async-programming`](./coldbox/async-programming/SKILL.md) | Async pipelines, ColdBox Futures, `allApply()/anyOf()`, thread-pool executors, AsyncManager |
| [`scheduled-tasks`](./coldbox/scheduled-tasks/SKILL.md) | Scheduler.cfc, task frequencies, lifecycle hooks, module schedulers, clustered fixation |
| [`decorators`](./coldbox/decorators/SKILL.md) | ControllerDecorator, RequestContextDecorator, extending framework internals |
| [`coldbox-proxy`](./coldbox/coldbox-proxy/SKILL.md) | ColdBox Proxy objects for web services, Flex/AIR, event gateways, CFC data binding |
| [`logging`](./coldbox/logging/SKILL.md) | LogBox in ColdBox, per-environment levels, WireBox logbox DSL, proxy logging |
| [`ai-integration`](./coldbox/ai-integration/SKILL.md) | bx-ai module, chat, streaming, pipelines, agents, RAG/vector memory, tool calling |
| [`coldbox-cli`](./coldbox/coldbox-cli/SKILL.md) | `coldbox create` workflows, app skeletons, language flags, scaffolding |
| [`coldbox-documenter`](./coldbox/coldbox-documenter/SKILL.md) | Documentation standards for handlers, models, modules, config files |
| [`coldbox-reviewer`](./coldbox/coldbox-reviewer/SKILL.md) | Code review heuristics for ColdBox applications and modules |

#### Testing Skills (under `coldbox/`)

| Skill | What It Covers |
|---|---|
| [`testing-base-classes`](./coldbox/testing-base-classes/SKILL.md) | Testing class hierarchy, annotations (appMapping, configMapping, unloadColdBox), test harness setup, when to use each base class |
| [`testing-handler`](./coldbox/testing-handler/SKILL.md) | Handler testing with `execute()`, rc/prc assertions, view selection, renderData, relocations, mock injection, BaseHandlerTest isolation |
| [`testing-integration`](./coldbox/testing-integration/SKILL.md) | Full virtual-app integration tests, lifecycle management, route testing, database rollback, custom matchers |
| [`testing-http-methods`](./coldbox/testing-http-methods/SKILL.md) | `get()`, `post()`, `put()`, `patch()`, `delete()`, `request()` simulation, headers, JSON bodies, `toHaveStatus()`, `toHaveInvalidData()` |
| [`testing-model`](./coldbox/testing-model/SKILL.md) | BaseModelTest, the `model` variable, mockLogger/mockLogBox/mockCacheBox/mockWireBox, mocking collaborators, init() patterns |
| [`testing-interceptor`](./coldbox/testing-interceptor/SKILL.md) | BaseInterceptorTest, the `interceptor` variable, mockController/mockRequestService/mockFlash, configProperties, announce point testing |

### `testbox` — Comprehensive TestBox Skills

| Skill | What It Covers |
|---|---|
| [`bdd`](./testbox/bdd/SKILL.md) | BDD suites with describe/it, Gherkin-style given/when/then, lifecycle hooks, focused/skipped specs, nested suites, labels |
| [`unit`](./testbox/unit/SKILL.md) | xUnit-style tests (testXxx), setup/teardown, `$assert` object, Arrange-Act-Assert pattern |
| [`assertions`](./testbox/assertions/SKILL.md) | `$assert` object, isTrue/isEqual/includes/isEmpty/throws/between/match, custom assertion functions, BoxLang dynamic assertion methods |
| [`expectations`](./testbox/expectations/SKILL.md) | Fluent `expect()`, all matchers (toBe/toBeTrue/toHaveKey/toThrow/toMatch), `not` operator, `expectAll()` over collections, custom matchers |
| [`mockbox`](./testbox/mockbox/SKILL.md) | MockBox mocks/stubs/spies, `$()` stubbing, `$args/$results/$throws`, call-count verification, `$callLog`, `querySim`, `$spy` |
| [`cbmockdata`](./testbox/cbmockdata/SKILL.md) | Generating fake data (age, email, name, uuid, lorem, etc.), arrays of objects, nested objects, custom supplier closures |
| [`runners`](./testbox/runners/SKILL.md) | CommandBox CLI runner, BoxLang CLI, HTML runner, programmatic TestBox, watcher mode, streaming runner, all CLI flags |
| [`reporters`](./testbox/reporters/SKILL.md) | ANTJunit/Console/Doc/JSON/JUnit/Min/Simple/Text/XML reporters, reporter options, custom IReporter implementations |
| [`listeners`](./testbox/listeners/SKILL.md) | Run listeners (onBundleStart/End, onSuiteStart/End, onSpecStart/End), progress indicators, custom loggers, live dashboards |
| [`testing-fixtures`](./testbox/testing-fixtures/SKILL.md) | Fixture factories, test data builders, shared fixture files, cbMockData integration, setup/teardown strategies |
| [`testing-coverage`](./testbox/testing-coverage/SKILL.md) | Code coverage setup, coverage reporting, CI integration, TestBox coverage options, improving coverage of untested paths |

### `security` — Security and Authentication Skills

| Skill | What It Covers |
|---|---|
| [`authentication`](./security/authentication/SKILL.md) | CBAuth login/logout, session management, remember-me patterns |
| [`authorization`](./security/authorization/SKILL.md) | CBSecurity roles, permissions, firewall rules |
| [`jwt-development`](./security/jwt-development/SKILL.md) | JWT flows, refresh tokens, API auth |
| [`security-implementation`](./security/security-implementation/SKILL.md) | End-to-end CBSecurity setup and configuration |
| [`csrf-protection`](./security/csrf-protection/SKILL.md) | CSRF tokens, forms, AJAX protection |
| [`rbac-patterns`](./security/rbac-patterns/SKILL.md) | Role and permission hierarchy patterns |
| [`api-authentication`](./security/api-authentication/SKILL.md) | API keys, bearer auth, OAuth2 integration |
| [`sso-integration`](./security/sso-integration/SKILL.md) | SAML, OIDC, OAuth2, social login patterns |
| [`passkeys-integration`](./security/passkeys-integration/SKILL.md) | Passkeys, WebAuthn, FIDO2, module integration |

### `wirebox` — Dependency Injection Skills

| Skill | What It Covers |
|---|---|
| [`wirebox-di`](./wirebox/SKILL.md) | Bootstrapping the injector, binder configuration, injection DSL, scopes, providers, lazy properties, property observers, child injectors, object populator |

### `cachebox` — Caching Skills

| Skill | What It Covers |
|---|---|
| [`cachebox-standalone`](./cachebox/SKILL.md) | Standalone CacheFactory, DSL configuration, object stores, eviction policies, cache providers, stampede protection, monitoring |

### `logbox` — Logging Skills

| Skill | What It Covers |
|---|---|
| [`logbox`](./logbox/SKILL.md) | LogBox standalone and ColdBox integration, appenders, root logger, category inheritance, structured logging, performance patterns |

### `modules` — Ortus/ColdBox Module Skills

| Skill | What It Covers |
|---|---|
| [`bcrypt`](./modules/bcrypt/SKILL.md) | BCrypt password hashing, work-factor config, hashing/verifying credentials, salts, registration, auth, password reset |
| [`cbantisamy`](./modules/cbantisamy/SKILL.md) | HTML/XML XSS sanitization via OWASP AntiSamy, policy selection, custom policy files, validation integration |
| [`cbauth`](./modules/cbauth/SKILL.md) | cbauth authentication, IUserService/IAuthUser, session login/logout, CBStorages integration, Flash messaging |
| [`cbcsrf`](./modules/cbcsrf/SKILL.md) | CSRF token generation, form helpers, AJAX/meta-tag patterns, route exemptions, SPA integration, token rotation |
| [`cbdebugger`](./modules/cbdebugger/SKILL.md) | CBDebugger panel, performance profiling, SQL query tracking, cache monitoring, request inspection |
| [`cbelasticsearch`](./modules/cbelasticsearch/SKILL.md) | Elasticsearch indexing, full-text search, query DSL, aggregations, highlighting, scroll/pagination, index aliases |
| [`cbfeeds`](./modules/cbfeeds/SKILL.md) | RSS/Atom feed reading (FeedReader) and generation (FeedGenerator), caching feeds |
| [`cbfs`](./modules/cbfs/SKILL.md) | File system abstraction, disk config (local/S3/RAM/FTP), CRUD operations, streaming, URL generation, uploads |
| [`cbi18n`](./modules/cbi18n/SKILL.md) | Internationalization, resource bundles (.properties/JSON), locale management, `$r`/getResource, named substitutions |
| [`cbjavaloader`](./modules/cbjavaloader/SKILL.md) | Dynamic JAR loading at runtime, classpath config, creating Java objects, reloading in development |
| [`cbmailservices`](./modules/cbmailservices/SKILL.md) | Sending email, SMTP/Postmark/SendGrid config, fluent mail builder, templates, attachments, async sending |
| [`cbmarkdown`](./modules/cbmarkdown/SKILL.md) | Markdown-to-HTML conversion, Processor injection, GFM/tables/code blocks, security for user-generated content |
| [`cbmessagebox`](./modules/cbmessagebox/SKILL.md) | Flash-scope messages (info/success/warning/error), setting in handlers, rendering in views, Bootstrap output |
| [`cbmockdata`](./modules/cbmockdata/SKILL.md) | Generating realistic fake/seed data, `mock()` API, entity population, bulk generation |
| [`cborm`](./modules/cborm/SKILL.md) | CBORM/Hibernate ORM, BaseORMService, VirtualEntityService, Criteria Builder, transactions, event interceptions |
| [`cbpaginator`](./modules/cbpaginator/SKILL.md) | Server-side pagination, Paginator service, metadata for APIs/views, QB integration, Bootstrap pagination controls |
| [`cbplaywright`](./modules/cbplaywright/SKILL.md) | End-to-end browser tests with Playwright, browser/page lifecycle, selectors, assertions, Page Object pattern, CI |
| [`cbproxies`](./modules/cbproxies/SKILL.md) | Dynamic Java-compatible proxies, ProxyFactory, Java interface implementation, JDBC/Hibernate/threading integration |
| [`cbq`](./modules/cbq/SKILL.md) | Async background job processing, job classes, queue config (database/Redis/SQS), dispatch, retry, worker pools |
| [`cbsecurity`](./modules/cbsecurity/SKILL.md) | Firewall rule config, annotation-based security, JWT auth, role/permission checks, custom validators |
| [`cbsecurity-passkeys`](./modules/cbsecurity-passkeys/SKILL.md) | WebAuthn/Passkeys, credential repository, registration/authentication ceremony, JavaScript integration |
| [`cbsso`](./modules/cbsso/SKILL.md) | SSO with SAML2/OAuth2/OIDC, SSOService API, redirect-to-provider, callback processing, user provisioning |
| [`cbstorages`](./modules/cbstorages/SKILL.md) | Session/Cookie/Cache/Request/Application storage adapters, get/set/exists/delete, encryption, TTLs |
| [`cbswagger`](./modules/cbswagger/SKILL.md) | OpenAPI 3.x (Swagger) doc generation, JSDoc annotations, request/response schemas, security definitions |
| [`cbvalidation`](./modules/cbvalidation/SKILL.md) | Input validation, constraint definitions, `validate()/validateOrFail()`, built-in/custom validators, REST error handling |
| [`cbwire`](./modules/cbwire/SKILL.md) | Reactive UI components without JavaScript, wire:model/wire:click, lifecycle hooks, actions, computed properties |
| [`cfmigrations`](./modules/cfmigrations/SKILL.md) | Database schema changes, Schema Builder API, migration files, seed files, CommandBox CLI |
| [`commandbox-boxlang`](./modules/commandbox-boxlang/SKILL.md) | CommandBox BoxLang server instances, REPL, task runners, module installation, server.json BoxLang config |
| [`commandbox-migrations`](./modules/commandbox-migrations/SKILL.md) | Migration CLI commands (up/down/reset/refresh), migration files, seeders, migration status, CI/CD usage |
| [`cors`](./modules/cors/SKILL.md) | CORS configuration, allowed origins/methods/headers, credentials, preflight OPTIONS, dynamic origin functions |
| [`hyper`](./modules/hyper/SKILL.md) | HTTP client, HyperBuilder, fluent GET/POST/PUT/DELETE, headers, auth, timeouts, response/error handling |
| [`mementifier`](./modules/mementifier/SKILL.md) | ORM entity/model serialization to structs/JSON, `this.memento`, includes/excludes, computed properties, named profiles |
| [`qb`](./modules/qb/SKILL.md) | QB query builder, fluent clauses, aggregates, inserts/updates/deletes, raw expressions, sub-queries, grammar config |
| [`quick`](./modules/quick/SKILL.md) | Quick Active Record entities, CRUD, relationships, query scopes, eager loading, accessors/mutators, global scopes |
| [`relax`](./modules/relax/SKILL.md) | REST API modeling with Relax DSL, route definitions, response schemas, JSON/XML spec generation |
| [`route-visualizer`](./modules/route-visualizer/SKILL.md) | Route inspection UI, route table output, development-only access restriction |
| [`rulebox`](./modules/rulebox/SKILL.md) | Business rules engine, when/then/otherwise closures, named rule sets, chaining rules, policy evaluation |
| [`s3sdk`](./modules/s3sdk/SKILL.md) | Amazon S3 / S3-compatible storage, bucket ops, upload/download/delete, presigned URLs, multipart uploads |
| [`socketbox`](./modules/socketbox/SKILL.md) | Real-time WebSocket apps, onConnect/onDisconnect/onMessage hooks, broadcasting to rooms, JavaScript integration |
| [`unleashsdk`](./modules/unleashsdk/SKILL.md) | Feature flags, `isEnabled()`, `getVariant()` for A/B testing, custom context, gradual rollout strategies |

### `docbox` — Documentation Skills

| Skill | What It Covers |
|---|---|
| [`docbox-generation`](./docbox/docbox-generation/SKILL.md) | DocBox generation workflows, strategies, output formats |
| [`docbox-annotations`](./docbox/docbox-annotations/SKILL.md) | DocBox annotations and documentation comment patterns |

---

## MCP Servers

The bundled [`.mcp.json`](./.mcp.json) includes Ortus documentation MCP endpoints for ColdBox, CommandBox, TestBox, WireBox, CacheBox, LogBox, QB, Quick ORM, CBORM, CBSecurity, CBAuth, CBValidation, ContentBox, Relax, and more.

---

## Repository Layout

| Path | Purpose |
|---|---|
| [`coldbox/`](./coldbox/) | Core ColdBox framework skills, testing patterns, and database migration workflows |
| [`testbox/`](./testbox/) | Comprehensive TestBox skills (BDD, xUnit, MockBox, runners, reporters) |
| [`security/`](./security/) | Authentication, authorization, CSRF, SSO, passkeys |
| [`wirebox/`](./wirebox/) | WireBox dependency injection |
| [`cachebox/`](./cachebox/) | CacheBox standalone caching |
| [`logbox/`](./logbox/) | LogBox logging |
| [`modules/`](./modules/) | 40+ Ortus/ColdBox module skills |
| [`docbox/`](./docbox/) | DocBox documentation generation skills |

---

## Resources

- [ColdBox Documentation](https://coldbox.ortusbooks.com/)
- [TestBox Documentation](https://testbox.ortusbooks.com/)
- [WireBox Documentation](https://wirebox.ortusbooks.com/)
- [CacheBox Documentation](https://cachebox.ortusbooks.com/)
- [LogBox Documentation](https://logbox.ortusbooks.com/)
- [BoxLang Documentation](https://boxlang.ortusbooks.com/)
- [Ortus Solutions](https://www.ortussolutions.com/)


---

## Quick Install

```bash
# Install all skills registered in the plugin manifest
npx skills add ortus-boxlang/coldbox-skills
```

```bash
# Install a specific category
npx skills add ortus-boxlang/coldbox-skills/coldbox
```

```bash
# Install a specific skill
npx skills add ortus-boxlang/coldbox-skills/coldbox/handler-development

# Install the dedicated TestBox category
npx skills add ortus-boxlang/coldbox-skills/testbox

# Install a specific TestBox skill
npx skills add ortus-boxlang/coldbox-skills/testbox/bdd-tdd-foundations
```

## Claude Plugin Install

Install this repository as a Claude plugin:

```bash
claude plugin install https://github.com/ortus-boxlang/coldbox-skills
```

If you use plugin marketplace commands:

```bash
/plugin marketplace add ortus-boxlang/coldbox-skills
/plugin install coldbox-agent-skills@ortus-boxlang
```

---

## Manifest Coverage

The current [`.claude-plugin/plugin.json`](./.claude-plugin/plugin.json) registers these categories for plugin install:

- `coldbox`
- `testbox`
- `security`
- `wirebox`
- `cachebox`
- `logbox`
- `modules` — 40+ Ortus/ColdBox module skills
- `docbox` — DocBox documentation generation

---

## Categories

### `coldbox` — Core Framework Skills

| Skill | What It Covers |
|---|---|
| [`handler-development`](./coldbox/handler-development/SKILL.md) | Handlers, CRUD actions, REST handlers, dependency injection |
| [`routing-development`](./coldbox/routing-development/SKILL.md) | Router configuration, named routes, constraints, route groups |
| [`event-model`](./coldbox/event-model/SKILL.md) | Event lifecycle, `event`, `rc`, `prc`, rendering and redirects |
| [`rest-api-development`](./coldbox/rest-api-development/SKILL.md) | RestHandler patterns, API validation, versioning, error handling |
| [`interceptor-development`](./coldbox/interceptor-development/SKILL.md) | Interceptors, framework interception points, custom events |
| [`configuration`](./coldbox/configuration/SKILL.md) | `ColdBox.cfc`, settings, conventions, environment configuration |
| [`app-layouts`](./coldbox/app-layouts/SKILL.md) | Flat, BoxLang, and Modern app layout selection, directory structures, engine compatibility, and migration guidance |
| [`view-rendering`](./coldbox/view-rendering/SKILL.md) | Views, layouts, partials, helpers, data rendering |
| [`layout-development`](./coldbox/layout-development/SKILL.md) | Layout patterns, nested layouts, conditional layouts |
| [`module-development`](./coldbox/module-development/SKILL.md) | Module structure, `ModuleConfig.cfc`, routes, settings |
| [`request-context`](./coldbox/request-context/SKILL.md) | Request context, flash scope, collections, metadata |
| [`flash-messaging`](./coldbox/flash-messaging/SKILL.md) | Flash RAM, message persistence, success/error/info/warning patterns |
| [`cache-integration`](./coldbox/cache-integration/SKILL.md) | Event and view caching, cache keys, cache invalidation |
| [`database-migrations`](./coldbox/database-migrations/SKILL.md) | Schema migrations with cfmigrations and CommandBox CLI: create, run, rollback, and seed |
| [`coldbox-cli`](./coldbox/coldbox-cli/SKILL.md) | `coldbox create` workflows, app skeletons, language flags, scaffolding |
| [`coldbox-documenter`](./coldbox/coldbox-documenter/SKILL.md) | Documentation standards for handlers, models, modules, config files |
| [`coldbox-reviewer`](./coldbox/coldbox-reviewer/SKILL.md) | Code review heuristics for ColdBox applications and modules |

#### Testing Skills

| Skill | What It Covers |
|---|---|
| [`testing-base-classes`](./coldbox/testing-base-classes/SKILL.md) | Testing class hierarchy, annotations (appMapping, configMapping, unloadColdBox), test harness setup, when to use each base class |
| [`testing-handler`](./coldbox/testing-handler/SKILL.md) | Handler testing with `execute()`, rc/prc assertions, view selection, renderData, relocations, mock injection, BaseHandlerTest isolation |
| [`testing-integration`](./coldbox/testing-integration/SKILL.md) | Full virtual-app integration tests, lifecycle management, route testing, database rollback, custom matchers |
| [`testing-http-methods`](./coldbox/testing-http-methods/SKILL.md) | `get()`, `post()`, `put()`, `patch()`, `delete()`, `request()` simulation, headers, JSON bodies, `toHaveStatus()`, `toHaveInvalidData()` |
| [`testing-model`](./coldbox/testing-model/SKILL.md) | BaseModelTest, the `model` variable, mockLogger/mockLogBox/mockCacheBox/mockWireBox, mocking collaborators, init() patterns |
| [`testing-interceptor`](./coldbox/testing-interceptor/SKILL.md) | BaseInterceptorTest, the `interceptor` variable, mockController/mockRequestService/mockFlash, configProperties, announce point testing |

### `testbox` — Comprehensive TestBox Skills

| Skill | What It Covers |
|---|---|
| [`bdd-tdd-foundations`](./testbox/bdd-tdd-foundations/SKILL.md) | Test style selection, behavior-first suite design, BDD/TDD strategy |
| [`specs-and-lifecycle`](./testbox/specs-and-lifecycle/SKILL.md) | Suite hooks, setup/teardown patterns, deterministic test isolation |
| [`assertions-and-matchers`](./testbox/assertions-and-matchers/SKILL.md) | Expectations, matcher families, exception assertions, readable failures |
| [`mockbox-testing-doubles`](./testbox/mockbox-testing-doubles/SKILL.md) | Mocks, stubs, spies, behavior setup, interaction verification |
| [`coldbox-integration-testing`](./testbox/coldbox-integration-testing/SKILL.md) | `BaseTestCase`, event execution, request/response assertions |
| [`data-fixtures-and-isolation`](./testbox/data-fixtures-and-isolation/SKILL.md) | Fixture builders, deterministic data, rollback and cleanup strategies |
| [`runners-reporters-and-ci`](./testbox/runners-reporters-and-ci/SKILL.md) | Runner commands, reporters, coverage, CI integration |
| [`cfml-boxlang-duality`](./testbox/cfml-boxlang-duality/SKILL.md) | BoxLang-first with CFML parity and mixed-codebase migration patterns |

### `security` — Security and Authentication Skills

| Skill | What It Covers |
|---|---|
| [`authentication`](./security/authentication/SKILL.md) | CBAuth login/logout, session management, remember-me patterns |
| [`authorization`](./security/authorization/SKILL.md) | CBSecurity roles, permissions, firewall rules |
| [`jwt-development`](./security/jwt-development/SKILL.md) | JWT flows, refresh tokens, API auth |
| [`security-implementation`](./security/security-implementation/SKILL.md) | End-to-end CBSecurity setup and configuration |
| [`csrf-protection`](./security/csrf-protection/SKILL.md) | CSRF tokens, forms, AJAX protection |
| [`rbac-patterns`](./security/rbac-patterns/SKILL.md) | Role and permission hierarchy patterns |
| [`api-authentication`](./security/api-authentication/SKILL.md) | API keys, bearer auth, OAuth2 integration |
| [`sso-integration`](./security/sso-integration/SKILL.md) | SAML, OIDC, OAuth2, social login patterns |
| [`passkeys-integration`](./security/passkeys-integration/SKILL.md) | Passkeys, WebAuthn, FIDO2, module integration |

### `wirebox` — Dependency Injection Skills

| Skill | What It Covers |
|---|---|
| [`wirebox-di`](./wirebox/wirebox-di/SKILL.md) | DI patterns, scopes, DSLs, binder configuration |
| [`wirebox-aop`](./wirebox/wirebox-aop/SKILL.md) | AOP, method interception, before/after/around advice |

### `cachebox` — Caching Skills

| Skill | What It Covers |
|---|---|
| [`caching-patterns`](./cachebox/caching-patterns/SKILL.md) | Cache providers, invalidation, get-or-set, event caching |

### `logbox` — Logging Skills

| Skill | What It Covers |
|---|---|
| [`logging-patterns`](./logbox/logging-patterns/SKILL.md) | LogBox config, appenders, log levels, structured logging |

### `docbox` — Repository Extras

| Skill | What It Covers |
|---|---|
| [`docbox-generation`](./docbox/docbox-generation/SKILL.md) | DocBox generation workflows, strategies, output formats |
| [`docbox-annotations`](./docbox/docbox-annotations/SKILL.md) | DocBox annotations and documentation comment patterns |

---

## MCP Servers

The bundled [`.mcp.json`](./.mcp.json) includes Ortus documentation MCP endpoints for ColdBox, CommandBox, TestBox, WireBox, CacheBox, LogBox, QB, Quick ORM, CBORM, CBSecurity, CBAuth, CBValidation, ContentBox, Relax, and more.

---

## Repository Layout

| Path | Purpose |
|---|---|
| [`coldbox/`](./coldbox/) | Core ColdBox framework skills and database migrations |
| [`testbox/`](./testbox/) | Dedicated comprehensive TestBox skills |
| [`security/`](./security/) | Authentication, authorization, CSRF, SSO, passkeys |
| [`wirebox/`](./wirebox/) | WireBox DI and AOP skills |
| [`cachebox/`](./cachebox/) | CacheBox skills |
| [`logbox/`](./logbox/) | LogBox skills |
| [`docbox/`](./docbox/) | Additional DocBox-specific skills in the repo |

---

## Resources

- [ColdBox Documentation](https://coldbox.ortusbooks.com/)
- [TestBox Documentation](https://testbox.ortusbooks.com/)
- [WireBox Documentation](https://wirebox.ortusbooks.com/)
- [CacheBox Documentation](https://cachebox.ortusbooks.com/)
- [LogBox Documentation](https://logbox.ortusbooks.com/)
- [BoxLang Documentation](https://boxlang.ortusbooks.com/)
- [Ortus Solutions](https://www.ortussolutions.com/)
