# ColdBox Skills

> AI agent skills for [ColdBox](https://www.coldbox.org/) and the surrounding Ortus ecosystem, including TestBox, WireBox, CacheBox, LogBox, ORM tooling, security, and DocBox.

This repository provides reusable skills for ColdBox and related tooling, compatible with skills.sh-style installs and Claude plugin workflows.

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
- `orm`
- `wirebox`
- `cachebox`
- `logbox`

The repository also contains a `docbox/` category with additional skills that are present in the repo but not currently listed in the plugin manifest.

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

### `orm` — Database and ORM Skills

| Skill | What It Covers |
|---|---|
| [`qb`](./orm/qb/SKILL.md) | QB query builder, fluent queries, joins, pagination |
| [`cborm`](./orm/cborm/SKILL.md) | CBORM active record and Hibernate integration |
| [`quick-orm`](./orm/quick-orm/SKILL.md) | Quick entities, relationships, scopes, eager loading |
| [`database-migrations`](./orm/database-migrations/SKILL.md) | cfmigrations schema builder, migrations, seeders |

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
| [`coldbox/`](./coldbox/) | Core ColdBox framework skills |
| [`testbox/`](./testbox/) | Dedicated comprehensive TestBox skills |
| [`security/`](./security/) | Authentication, authorization, CSRF, SSO, passkeys |
| [`orm/`](./orm/) | QB, Quick ORM, CBORM, migrations |
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
