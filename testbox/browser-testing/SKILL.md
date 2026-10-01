---
name: testbox-browser-testing
description: "Use this skill when writing or running browser tests with TestBox on BoxLang: extending testbox.system.BrowserSpec, installing bx-playwright and Chromium (install-bx-module bx-playwright, bxPlaywright install), the browserProfile and baseURL class annotations, browse() with one isolated page per callback argument, this.playwright() and browserAvailable(), the browser matchers (toHaveTitle, toHaveURL, toHavePath, toSee, toHaveText, toBeVisible, toBeHidden, toHaveCount, toHaveValue and their not forms), automatic screenshot/trace/video attachments, attach(), spec retries (it retries argument, retries bundle or method annotation, --retries), rerunning failures with --failed, starting the app with --web-server/--web-server-url/--web-server-timeout, CI setup, and debugging failed browser specs with bxPlaywright show-trace."
applyTo: "**/tests/**/*.{bx,bxm}"
---

# TestBox Browser Testing

## When to Use This Skill

- Writing specs that drive a real browser (Chromium, Firefox or WebKit) against a running web app
- Choosing between `BrowserSpec`, ColdBox's `BrowserTestCase` and plain `BaseSpec`
- Asserting on pages and locators with `expect( page ).toSee( "Welcome" )` style matchers
- Getting screenshots, traces and videos of failed specs into the test report
- Retrying flaky specs, rerunning only failures, or letting the runner start the web server
- Setting up browser tests in CI and debugging a failed run

## Requirements and Engine Support

`testbox.system.BrowserSpec` is built on the [bx-playwright](https://bxplaywright.boxlang.io) module, so it is **BoxLang only** (BoxLang 1.17+, Java 21+, TestBox 7.2.0+ with `testbox.system.browser`).

- When the engine is not BoxLang, or bx-playwright is not installed, `browse()` and `this.playwright()` **skip** the running spec with the reason (`bx-playwright is not installed: install-bx-module bx-playwright`), so the rest of the suite still runs.
- Keep browser specs as `.bx` files in their own folder (for example `tests/specs/browser`) and exclude that folder on Lucee or Adobe, which cannot compile BoxLang classes.
- Attachments, retries, `--failed` and `Playwright.AssertionFailed` counted as a failure work for every spec on every engine.

| You test... | Extend |
|---|---|
| Any web app from TestBox | `testbox.system.BrowserSpec` |
| A ColdBox app (named routes, `loginAs()`) | `coldbox.system.testing.BrowserTestCase`, see the `coldbox-testing-browser` skill |
| Any other BoxLang spec | `testbox.system.BaseSpec` plus `testbox.system.browser.BrowserSupport` (see below) |

For CFML engines, the separate `cbplaywright` module (see `modules/cbplaywright`) is the alternative.

## Install bx-playwright and a Browser

```bash
# The module and its bxPlaywright CLI in the BoxLang OS runtime
install-bx-module bx-playwright

# Playwright driver, Node.js and Chromium, then check the setup
bxPlaywright install chromium
bxPlaywright doctor

# Linux CI machines: also install the system libraries the browser needs
bxPlaywright install chromium --with-deps
```

Everything downloads to `~/.boxlang/playwright`. The module must be loaded by the runtime that **runs the tests**: the BoxLang OS runtime for `./testbox/run`, or the CommandBox BoxLang server for the HTML runner (for example `"onServerInitialInstall": "install bx-playwright --noSave"` in `server.json`).

## Your First Browser Spec

```js
// tests/specs/browser/LoginSpec.bx
class extends="testbox.system.BrowserSpec" baseURL="http://localhost:8080" browserProfile="ci" {

	function run() {
		describe( "Login", () => {

			it( "signs in with valid credentials", () => {
				browse( ( page ) => {
					page.visit( "/login" )
						.fill( "Email", "luis@ortus.com" )
						.fill( "Password", "secret" )
						.click( "Sign in" )

					expect( page ).toHavePath( "/dashboard" )
					expect( page ).toSee( "Welcome back" )
				} )
			} )

			it( "rejects a wrong password", () => {
				browse( ( page ) => {
					page.visit( "/login" )
						.fill( "Email", "luis@ortus.com" )
						.fill( "Password", "nope" )
						.click( "Sign in" )

					expect( page ).toHavePath( "/login" )
					expect( page.locator( "@error" ) ).toHaveText( "Invalid credentials" )
				} )
			} )

		} )
	}

}
```

xUnit style works the same: `function testSignsIn() { browse( ( page ) => { ... } ) }`.

The page API (`visit()`, `fill()`, `click()`, locators, finders, `assertSee()`...) is bx-playwright's. Both assertion styles mix freely: TestBox browser matchers (`expect( page ).toSee()`) and bx-playwright inline assertions (`page.assertSee()`, `page.assertPathIs()`). A `Playwright.AssertionFailed` counts as a spec **failure**; other `Playwright.*` exceptions count as errors.

## How a BrowserSpec Works

- **One browser per bundle**, started on first use and closed after the bundle by `closeBrowser()`, which carries the `afterAll` annotation. Your own `beforeAll()` / `afterAll()` need no `super` calls.
- **Fresh pages per `browse()` call**, each in its own browser context (own cookies, storage, session), closed when the callback ends.
- **Browser matchers registered** for every spec of the bundle.
- **Failure artifacts attached**: when the callback throws, the kept screenshots, trace and videos are attached to the spec and the exception is rethrown unchanged.
- **Not thread safe**: never use `asyncAll` in suites that browse.

## Class Annotations

| Annotation | Description |
|---|---|
| `baseURL` | URL that relative visits such as `page.visit( "/login" )` resolve against |
| `browserProfile` | bx-playwright profiles for the bundle browser, a list such as `ci` or `ci,mobile` (merged left to right) |
| `retries` | Extra attempts for failing specs of the bundle (see [Retries](#retries)) |

`baseURL` and `browserProfile` are also found on parent classes, so a shared base spec can set them once.

`baseURL` resolution order:

1. The `baseURL` annotation of the spec class (or a class it extends)
2. The `--web-server-url` of the BoxLang runner when it started a web server (kept in `server.testbox.webServerURL`)
3. The bx-playwright `baseURL` setting or the `BX_PLAYWRIGHT_BASEURL` environment variable

Without `browserProfile`, bx-playwright uses its `defaultProfile` setting or `BX_PLAYWRIGHT_PROFILE`, so `BX_PLAYWRIGHT_PROFILE=ci` is a handy CI switch.

## `browse( callback, options = {} )`

Runs the callback with fresh pages: **one page per declared callback argument**, each in its own browser context. A callback with no declared arguments receives one page as its first positional argument. It returns the callback result.

```js
// One page
browse( ( page ) => page.visit( "/" ).assertSee( "Welcome" ) )

// Return a value out of the page
var title = browse( ( page ) => page.visit( "/" ).title() )
expect( title ).toInclude( "Home" )

// Two users signed in at once, no shared cookies
browse( ( alice, bob ) => {
	alice.visit( "/login" ).fill( "Email", "alice@example.com" ).fill( "Password", "secret" ).click( "Sign in" )
	bob.visit( "/login" ).fill( "Email", "bob@example.com" ).fill( "Password", "secret" ).click( "Sign in" )
	alice.visit( "/chat" ).fill( "Message", "Hi Bob" ).click( "Send" )
	expect( bob.visit( "/chat" ) ).toSee( "Hi Bob" )
} )

// Second argument: bx-playwright newContext() options for every page of the call
browse(
	( page ) => {
		page.visit( "/" )
		expect( page.locator( "@mobile-menu" ) ).toBeVisible()
	},
	{ viewport : { width : 390, height : 844 }, timeouts : { assertion : 10000 } }
)
```

## `this.playwright()` and `browserAvailable()`

```js
// The bundle manager, created with the bundle annotations and closed after the bundle
var response = this.playwright().request().get( "/api/products" )
expect( response.status() ).toBe( 200 )

var ctx = this.playwright().newContext()   // manual contexts when browse() is not enough
```

> Always call it as `this.playwright()`. An unqualified `playwright()` resolves to the bx-playwright BIF, even inside a `BrowserSpec`, and returns a **new** manager that ignores the bundle annotations and is not closed for you.

`browserAvailable()` is true on BoxLang with bx-playwright loaded. Use it to skip whole suites up front:

```js
describe( title = "Checkout", skip = !browserAvailable(), body = () => {
	it( "pays with a card", () => {
		browse( ( page ) => { ... } )
	} )
} )
```

## Browser Matchers

Registered automatically for `BrowserSpec` and ColdBox `BrowserTestCase` bundles. In any other BoxLang spec:

```js
addMatchers( new testbox.system.browser.BrowserMatchers() )
```

| Matcher | Target | Passes when |
|---|---|---|
| `toHaveTitle( title )` | page | Title is exactly `title`, or matches a `page.regex()` |
| `toHaveURL( url )` | page | Full URL is exactly `url`, or matches a `page.regex()` |
| `toHavePath( path )` | page | URL path equals `path` (case sensitive), ignoring scheme, host, query and hash |
| `toSee( text )` | page or locator | Text (or regex) is visible, like `assertSee()` |
| `toHaveText( text )` | locator | Text of the first element is exactly `text` (whitespace normalized), or matches a regex |
| `toBeVisible()` | locator | First element is visible |
| `toBeHidden()` | locator | First element is hidden or not in the page |
| `toHaveCount( count )` | locator | Locator matches exactly `count` elements |
| `toHaveValue( value )` | locator | Input, textarea or select value equals `value`, or matches a regex |

Every matcher has a `not` form: prefix with `not` (`notToSee`, `notToHaveTitle`, `notToBeVisible`, `notToHaveCount`...).

```js
expect( page ).toHaveTitle( "Dashboard" )
expect( page ).toHaveTitle( page.regex( "^Dash" ) )
expect( page ).toHaveURL( "http://localhost:8080/dashboard?tab=1" )
expect( page ).toHavePath( "/dashboard" )
expect( page ).toSee( "Welcome" )
expect( page ).notToSee( "Error" )
expect( page.locator( "nav" ) ).toSee( "Logout" )
expect( page.locator( "h1" ) ).toHaveText( "Todos" )
expect( page.locator( ".todo" ) ).toHaveCount( 3 )
expect( page.locator( "@spinner" ) ).toBeHidden()
expect( page.locator( "@error" ) ).notToBeVisible()
expect( page.locator( "#email" ) ).toHaveValue( "luis@ortus.com" )
```

- **Web-first**: they delegate to bx-playwright's retrying assertions and wait until they pass or the assertion timeout (`timeouts.assertion`, 5000 ms by default) expires. Negated forms wait too. Never `sleep()` before them.
- A failure is a normal TestBox failure carrying bx-playwright's message (expected, received, call log).
- They refuse values that are not bx-playwright pages or locators, except `toHavePath()`, which falls back to the core Data Navigator `toHavePath()` for structs and other data.

## Attachments

### Automatic browser artifacts

When a `browse()` callback throws, its contexts close with `failed = true`, so bx-playwright's artifact policies keep their files, and the kept files are attached to the spec with types `screenshot`, `trace` and `video`. Turn artifacts on with a profile (`browserProfile="ci"`, `record` or `debug`) or per call:

```js
browse( ( page ) => { ... }, {
	artifacts : { screenshot : "only-on-failure", trace : "retain-on-failure", video : "retain-on-failure" }
} )
```

Policies: `off`, `on`, `only-on-failure`, `retain-on-failure`.

### `attach( path, type = "file", name = "" )`

Any spec (browser or not, any engine) can attach files to the running spec. `name` defaults to the file name of `path`.

```js
it( "exports the report", () => {
	var file = service.export()
	attach( file, "file", "export.csv" )
	attach( logPath, "log" )
	expect( fileExists( file ) ).toBeTrue()
} )
```

- Call it from a spec body or a `beforeEach()` / `afterEach()` / `aroundEach()` closure. Outside a running spec it throws `TestBox.InvalidContext`.
- Attachments are kept for passed and failed specs in the `attachments` array of the spec stats.
- Reporters: the JSON report includes them, the Simple report links them, JUnit and ANTJunit add a `<system-out>` with one `[[ATTACHMENT|path]]` line per file, and text, console and stream outputs list them under failed specs.

## Retries

A failing or erroring spec reruns up to N **extra** times; only the final attempt is recorded. BDD reruns every `beforeEach()`, the `aroundEach()` closures, the body and `afterEach()`; xUnit reruns `setup()`, the test and `teardown()`. Skipped specs are never retried. Output shows "(passed after N attempts)" and the `attempts` spec stat counts the runs. Attachments of earlier attempts stay on the spec.

```js
// 1. Spec argument (wins), also on fit() and xit()
it( title = "talks to a flaky service", retries = 2, body = () => { ... } )

// 2. Bundle annotation
class extends="testbox.system.BrowserSpec" retries="1" { ... }

// xUnit: method annotation, wins over the bundle annotation
function testCheckout() retries="2" { ... }
```

```bash
# 3. Global default (lowest precedence)
./testbox/run --retries=2
```

Programmatically: `new testbox.system.TestBox( directory = "tests.specs", options = { retries : 2 } )`.

Retries hide flakiness, they do not fix it: prefer the web-first matchers, which already wait, and keep the count low.

## Running Browser Tests

```bash
# Run the browser folder
./testbox/run --directory=tests.specs.browser

# Rerun only what failed or errored in the last run
./testbox/run --failed

# Let the runner start the app, wait for it, run the tests and stop it
./testbox/run --directory=tests.specs.browser \
	--web-server="boxlang-miniserver --port 8080" \
	--web-server-url=http://localhost:8080 \
	--web-server-timeout=60
```

| Option | Default | Description |
|---|---|---|
| `--retries` | `0` | Extra runs for failing or erroring specs (spec and bundle values win) |
| `--failed` | `false` | Run only bundles and specs listed in `{reportpath}/.testbox-failed.json`, which every run writes. When missing or empty, prints a message and runs nothing |
| `--web-server` | | Shell command started before the tests (`sh -c`, `cmd /c` on Windows), stopped with its child processes after them |
| `--web-server-url` | `http://localhost:8080` | Polled until it answers with a status below 500; becomes the default `baseURL` of `BrowserSpec` bundles |
| `--web-server-timeout` | `60` | Seconds to wait; when the server does not answer the runner stops it and exits with code 1 |

If the server command has its own flags that share a runner option name (for example `--directory`), put it in a script.

## CI (GitHub Actions)

```yaml
jobs:
  browser-tests:
    runs-on: ubuntu-latest
    env:
      BX_PLAYWRIGHT_PROFILE: ci        # headless, keep artifacts of failures
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: "21" }
      - uses: ortus-boxlang/setup-boxlang@main
        with:
          version: latest
          modules: bx-playwright
      - uses: actions/cache@v4
        with:
          path: ~/.boxlang/playwright
          key: playwright-${{ runner.os }}
      - name: Install Chromium
        run: |
          export PATH="$HOME/.boxlang/bin:$PATH"
          bxPlaywright install chromium --with-deps
      - name: Run the browser tests
        run: |
          ./testbox/run --directory=tests.specs.browser \
            --web-server="boxlang-miniserver --port 8080" \
            --web-server-url=http://127.0.0.1:8080 \
            --retries=1 --reporter=junit
      - name: Upload failure artifacts
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: browser-test-artifacts
          path: |
            ~/.boxlang/playwright/artifacts
            tests/results
          if-no-files-found: ignore
```

Run browser specs in their own job so they do not slow down unit and integration tests, and cache `~/.boxlang/playwright` to skip the browser download.

## Debugging

- Open a trace from a failed spec (or a CI artifact): `bxPlaywright show-trace path/to/trace.zip`.
- Watch the browser: `browserProfile="debug"` (headed, slowMo, all artifacts) or `BX_PLAYWRIGHT_HEADLESS=false`.
- Inspect the page structure with `page.snapshot()`; record selectors with `bxPlaywright codegen <url>`.
- A spec reported as **skipped** with an install hint means bx-playwright is not loaded in the runtime running the tests: run `bxPlaywright doctor`.

## Using the Browser From Any BoxLang Spec

`BrowserSpec` delegates to `testbox.system.browser.BrowserSupport`; use it directly when you cannot change the base class, and close it yourself:

```js
class extends="testbox.system.BaseSpec" {

	function beforeAll() {
		variables.browser = new testbox.system.browser.BrowserSupport( this )
		addMatchers( new testbox.system.browser.BrowserMatchers() )
	}

	function afterAll() {
		variables.browser.close()
	}

	function run() {
		describe( "Home page", () => {
			it( "greets visitors", () => {
				variables.browser.browse( ( page ) => {
					page.visit( "http://localhost:8080/" )
					expect( page ).toSee( "Welcome" )
				} )
			} )
		} )
	}

}
```

## Related Skills

- `coldbox-testing-browser`: `BrowserTestCase` for ColdBox apps (named routes, `loginAs()`)
- `testbox-runners`, `testbox-reporters`, `testbox-expectations`, `testbox-bdd`, `testbox-unit`
- bx-playwright skills (`bx-playwright-browsing`, `bx-playwright-assertions`, `bx-playwright-testing`) for the full page API, page objects and visual regression
