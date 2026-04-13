---
name: cbplaywright
description: >
  Use this skill when writing end-to-end browser tests in ColdBox/BoxLang using the cbplaywright module
  (a Playwright wrapper). Covers test bundle setup, browser/page lifecycle, navigation, selectors,
  assertions, form interactions, screenshots, network interception, the Page Object pattern, and CI
  pipeline integration.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# CBPlaywright Skill

## When to Use This Skill

Load this skill when:
- Writing browser-level end-to-end tests for ColdBox applications
- Automating form submissions, navigation, or multi-step flows
- Taking screenshots or recording traces for visual regression
- Intercepting or mocking network requests in tests
- Structuring E2E tests with the Page Object pattern

## Installation

```bash
box install cbplaywright --saveDev
```

Install Playwright browsers (run once after installation):

```bash
box playwright install
```

## Core API

### Test Bundle Setup

```js
// tests/e2e/UserFlowSpec.bx
class extends="cbplaywright.models.PlaywrightTestCase" {

    // Browser type: chromium | firefox | webkit
    this.browser = "chromium"
    this.headless = true
    this.baseURL  = "http://localhost:8500"

    function run( spec ) {
        describe( "User registration flow", () => {

            it( "can register a new user", ( browser ) => {
                var page = browser.newPage()
                // test body
                page.close()
            } )

        } )
    }
}
```

### Navigation & Waiting

```js
page.navigate( baseURL & "/register" )
page.waitForLoadState( "networkidle" )
page.waitForURL( baseURL & "/dashboard" )
```

### Selectors

```js
page.locator( "input[name='email']" )
page.locator( "#submitBtn" )
page.locator( "text=Sign In" )
page.locator( "role=button[name='Submit']" )
page.getByPlaceholder( "Email address" )
page.getByLabel( "Password" )
page.getByRole( "link", { name: "Forgot password?" } )
page.getByText( "Welcome back" )
```

### Interactions

```js
page.locator( "input[name='email']" ).fill( "test@example.com" )
page.locator( "input[name='password']" ).fill( "secret123" )
page.locator( "#loginBtn" ).click()
page.locator( "select[name='role']" ).selectOption( "admin" )
page.locator( "input[type='checkbox']" ).check()
page.locator( "input[type='checkbox']" ).uncheck()
page.keyboard.press( "Enter" )
```

### Assertions

```js
expect( page.locator( "h1" ) ).toBeVisible()
expect( page.locator( ".alert-success" ) ).toContainText( "Registration successful" )
expect( page.url() ).toContain( "/dashboard" )
expect( page.locator( "table tbody tr" ) ).toHaveCount( 5 )
expect( page.locator( "input[name='email']" ) ).toHaveValue( "test@example.com" )
expect( page.locator( "#logoutBtn" ) ).toBeEnabled()
```

### Screenshots & Traces

```js
page.screenshot( { path: expandPath( "/tests/screenshots/home.png" ) } )

// Full-page screenshot
page.screenshot( { path: expandPath( "/tests/screenshots/full.png" ), fullPage: true } )
```

### Network Interception

```js
// Mock an API response
page.route( "**/api/users", ( route ) => {
    route.fulfill( {
        status      : 200,
        contentType : "application/json",
        body        : serializeJSON( [ { id: 1, name: "Mock User" } ] )
    } )
} )
```

## Production Patterns

### Page Object Pattern

```js
// tests/e2e/pages/LoginPage.bx
class {

    property name="page";
    property name="baseURL" default="http://localhost:8500";

    LoginPage function init( page, baseURL = "http://localhost:8500" ) {
        variables.page    = page
        variables.baseURL = baseURL
        return this
    }

    LoginPage function navigate() {
        page.navigate( baseURL & "/login" )
        return this
    }

    LoginPage function fillCredentials( email, password ) {
        page.locator( "input[name='email']" ).fill( email )
        page.locator( "input[name='password']" ).fill( password )
        return this
    }

    void function submit() {
        page.locator( "button[type='submit']" ).click()
        page.waitForLoadState( "networkidle" )
    }

    string function getErrorMessage() {
        return page.locator( ".alert-danger" ).innerText()
    }
}
```

```js
// In the test spec
it( "shows error for bad credentials", ( browser ) => {
    var page      = browser.newPage()
    var loginPage = new tests.e2e.pages.LoginPage( page )

    loginPage
        .navigate()
        .fillCredentials( "bad@example.com", "wrongpassword" )
        .submit()

    expect( loginPage.getErrorMessage() ).toContain( "Invalid credentials" )
    page.close()
} )
```

### Full CRUD Flow Test

```js
it( "can create and delete a user", ( browser ) => {
    var page = browser.newPage()

    // Login
    page.navigate( baseURL & "/login" )
    page.getByLabel( "Email" ).fill( "admin@example.com" )
    page.getByLabel( "Password" ).fill( "Admin1234!" )
    page.getByRole( "button", { name: "Sign In" } ).click()
    page.waitForURL( "**/dashboard" )

    // Create user
    page.navigate( baseURL & "/users/new" )
    page.getByLabel( "Name" ).fill( "E2E Test User" )
    page.getByLabel( "Email" ).fill( "e2e_#createUUID()#@example.com" )
    page.getByRole( "button", { name: "Save" } ).click()
    expect( page.locator( ".alert-success" ) ).toBeVisible()

    // Delete the user
    page.locator( "button.delete-user:last-of-type" ).click()
    page.locator( ".confirm-dialog .btn-danger" ).click()
    expect( page.locator( ".alert-success" ) ).toContainText( "User deleted" )

    page.close()
} )
```

## Best Practices

- **Use Page Objects** for any flow with more than 3 interactions — keeps tests readable
- **Use `getByRole` and `getByLabel`** selectors — more robust than CSS selectors when UI changes
- **Always close pages** in teardown or at the end of each test to avoid leaked browser contexts
- **Use `waitForLoadState( "networkidle" )`** after navigation or actions that trigger async requests
- **Store base URL in a single property** — avoids hardcoding across hundreds of tests
- **Mock external APIs** with `page.route()` to isolate tests from third-party flakiness
- **Run in headless mode in CI** and only enable headed mode for local debugging

## Documentation

- cbplaywright: https://github.com/coldbox-modules/cbplaywright
- Playwright API: https://playwright.dev/docs/api/class-page
