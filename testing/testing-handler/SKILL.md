---
name: coldbox-testing-handler
description: "Use this skill when testing ColdBox handlers/controllers with BaseTestCase, executing events with execute(), asserting against rc/prc request collections, testing view rendering, verifying HTTP status codes, injecting mock services into handlers, or using the ColdBox integration test base class."
---

# Handler Testing in ColdBox

## Overview

ColdBox provides `coldbox.system.testing.BaseTestCase` for testing handlers in a full ColdBox context. Tests call `execute()` to simulate requests and inspect the resulting event object, private collection (prc), request collection (rc), rendered output, and HTTP status codes.

## Language Mode Reference

Examples use **BoxLang (`.bx`)** syntax by default. Adapt for your target language:

| Concept | BoxLang (`.bx`) | CFML (`.cfc`) |
|---------|-----------------|---------------|
| Class declaration | `class [extends="..."] {` | `component [extends="..."] {` |
| DI annotation | `@inject` above `property name="svc";` | `property name="svc" inject="svc";` |
| View templates | `.bxm` suffix | `.cfm` / `.cfml` suffix |
| Tag prefix | `<bx:if>`, `<bx:output>`, `<bx:set>` | `<cfif>`, `<cfoutput>`, `<cfset>` |

> **CFML Compat Mode**: With BoxLang + CFML Compat module, `.bx` and `.cfc` files coexist freely. BoxLang-native classes use `class {}` (`.bx` files); CFML-compat classes use `component {}` (`.cfc` files).

## Basic Handler Test Setup

```boxlang
component extends="coldbox.system.testing.BaseTestCase" appMapping="/root" {

    function beforeAll() {
        super.setup()
    }

    function afterAll() {
        structDelete( application, "cbController" )
    }

    function run() {
        describe( "UsersHandler", () => {

            it( "should list all users", () => {
                // Execute the event
                event = execute( event = "users.index", renderResults = true )

                // Assert view rendered
                expect( event.getCurrentView() ).toBe( "users/index" )

                // Assert data placed in prc
                expect( event.getPrivateValue( "users" ) ).toBeArray()
            } )

            it( "should show a single user", () => {
                event = execute(
                    event = "users.show",
                    renderResults = true,
                    eventArguments = { id: 1 }
                )

                expect( event.getPrivateValue( "user" ) ).toBeStruct()
                expect( event.getPrivateValue( "user" ).id ).toBe( 1 )
            } )
        } )
    }
}
```

## Testing REST API Handlers

```boxlang
component extends="coldbox.system.testing.BaseTestCase" appMapping="/root" {

    function beforeAll() {
        super.setup()
    }

    function run() {
        describe( "API Users Handler", () => {

            it( "should return JSON user list", () => {
                event = execute(
                    event = "api.v1.users.index",
                    renderResults = true
                )

                expect( event.getStatusCode() ).toBe( 200 )

                // Parse rendered JSON
                response = deserializeJSON( event.getRenderedContent() )
                expect( response ).toHaveKey( "data" )
                expect( response.data ).toBeArray()
            } )

            it( "should return 404 for missing user", () => {
                event = execute(
                    event = "api.v1.users.show",
                    renderResults = true,
                    eventArguments = { id: 99999 }
                )

                expect( event.getStatusCode() ).toBe( 404 )
            } )

            it( "should create a user via POST", () => {
                // Simulate POST request with form data
                event = execute(
                    event = "api.v1.users.create",
                    renderResults = true,
                    eventArguments = {
                        name: "Test User",
                        email: "test@example.com",
                        password: "SecurePass123!"
                    }
                )

                expect( event.getStatusCode() ).toBe( 201 )
                response = deserializeJSON( event.getRenderedContent() )
                expect( response.data ).toHaveKey( "id" )
            } )
        } )
    }
}
```

## Injecting Mock Services

```boxlang
component extends="coldbox.system.testing.BaseTestCase" appMapping="/root" {

    function beforeAll() {
        super.setup()
    }

    function run() {
        describe( "Users Handler with Mocks", () => {

            beforeEach( () => {
                // Get mock service and register in WireBox
                variables.mockUserService = createMock( "models.UserService" )
                getWireBox().registerNewInstance( "UserService" ).$results( mockUserService )
                // OR use injection:
                variables.handler = getController().getHandler( "users" )
                handler.userService = mockUserService
            } )

            it( "should call userService.findAll on index", () => {
                mockUserService.$( "findAll" ).$results( [
                    { id: 1, name: "Alice" },
                    { id: 2, name: "Bob" }
                ] )

                event = execute( event = "users.index", renderResults = true )

                // Verify mock was called
                expect( mockUserService.$once( "findAll" ) ).toBeTrue()

                // Verify data in prc
                users = event.getPrivateValue( "users" )
                expect( users ).toHaveLength( 2 )
            } )
        } )
    }
}
```

## Testing with HTTP Methods

```boxlang
component extends="coldbox.system.testing.BaseTestCase" appMapping="/root" {

    function beforeAll() {
        super.setup()
    }

    function run() {
        describe( "HTTP method testing", () => {

            it( "should handle GET request", () => {
                // Setup request as GET
                getRequestContext().setHTTPMethod( "GET" )

                event = execute( event = "users.index", renderResults = true )
                expect( event.getStatusCode() ).toBe( 200 )
            } )

            it( "should reject DELETE without auth", () => {
                getRequestContext().setHTTPMethod( "DELETE" )

                event = execute(
                    event = "users.delete",
                    renderResults = true,
                    eventArguments = { id: 1 }
                )

                expect( event.getStatusCode() ).toBeInRange( 401, 403 )
            } )
        } )
    }
}
```

## Testing Redirects

```boxlang
component extends="coldbox.system.testing.BaseTestCase" appMapping="/root" {

    function beforeAll() {
        super.setup()
    }

    function run() {
        describe( "Redirect testing", () => {

            it( "should redirect after form submit", () => {
                event = execute(
                    event = "users.store",
                    renderResults = true,
                    eventArguments = {
                        name: "New User",
                        email: "new@example.com"
                    }
                )

                // Check that relocate was called
                expect( event.isRedirect() ).toBeTrue()
                expect( event.getRelocationURL() ).toInclude( "users" )
            } )
        } )
    }
}
```

## Testing with Announcements/Interceptors

```boxlang
component extends="coldbox.system.testing.BaseTestCase" appMapping="/root" {

    function beforeAll() {
        super.setup()
        // Reset the interceptor state
        getController().getInterceptorService().clearInterceptionPoints()
    }

    function run() {
        describe( "Event announcement testing", () => {

            it( "should announce preUserCreate event", () => {
                intercepted = false

                // Register a test interceptor inline
                getController().getInterceptorService()
                    .registerInterceptor(
                        interceptorObject = new cfcs.TestInterceptor(
                            onPreUserCreate = () => { intercepted = true }
                        )
                    )

                execute( event = "users.store", eventArguments = { name: "Test" } )

                expect( intercepted ).toBeTrue()
            } )
        } )
    }
}
```

## Key Methods Reference

| Method | Purpose |
|--------|---------|
| `execute( event, renderResults, eventArguments )` | Execute a ColdBox event |
| `event.getCurrentView()` | Get rendered view name |
| `event.getRenderedContent()` | Get rendered output string |
| `event.getPrivateValue( key )` | Get value from prc |
| `event.getValue( key )` | Get value from rc |
| `event.getStatusCode()` | Get HTTP status code |
| `event.isRedirect()` | Check if relocate() was called |
| `event.getRelocationURL()` | Get redirect destination |
| `getWireBox()` | Access WireBox for mock injection |
| `getController()` | Access ColdBox controller |
| `getRequestContext()` | Access the current request context |
| `createMock( path )` | Create a MockBox mock |
