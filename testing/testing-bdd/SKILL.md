---
name: coldbox-testing-bdd
description: "Use this skill when writing BDD-style tests in ColdBox/TestBox using describe/it blocks, setting up beforeEach/afterEach/beforeAll/afterAll lifecycle hooks, creating nested test suites, using expect() assertions, or organizing tests around behavior descriptions."
---

# BDD Testing in ColdBox

## Overview

BDD (Behavior-Driven Development) testing in ColdBox uses TestBox's `describe()/it()` syntax to organize tests around behaviors. Tests extend `testbox.system.BaseSpec` and structure assertions with `expect().toBe()` fluent assertions.

## Basic BDD Test Structure

```boxlang
component extends="testbox.system.BaseSpec" {

    function run() {
        describe( "UserService", () => {

            // Lifecycle hooks
            beforeAll( () => {
                variables.userService = getInstance( "UserService" )
            } )

            afterAll( () => {
                // cleanup
            } )

            beforeEach( () => {
                // runs before each it()
            } )

            afterEach( () => {
                // runs after each it()
            } )

            it( "should create a user", () => {
                user = userService.create( { name: "John", email: "john@example.com" } )
                expect( user.id ).toBeNumeric()
                expect( user.name ).toBe( "John" )
            } )

            it( "should throw on duplicate email", () => {
                expect( () => {
                    userService.create( { email: "duplicate@example.com" } )
                    userService.create( { email: "duplicate@example.com" } )
                } ).toThrow()
            } )
        } )
    }
}
```

## Nested Test Suites

```boxlang
component extends="testbox.system.BaseSpec" {

    function run() {
        describe( "UserService", () => {

            beforeAll( () => {
                variables.userService = getInstance( "UserService" )
            } )

            describe( "CRUD Operations", () => {

                it( "should create a user", () => {
                    user = userService.create( {
                        name: "John Doe",
                        email: "john@example.com"
                    } )
                    expect( user.id ).toBeNumeric()
                } )

                it( "should read a user", () => {
                    created = userService.create( { name: "Jane", email: "jane@example.com" } )
                    found = userService.findById( created.id )
                    expect( found.email ).toBe( "jane@example.com" )
                } )

                it( "should update a user", () => {
                    user = userService.create( { name: "Old Name", email: "user@example.com" } )
                    updated = userService.update( user.id, { name: "New Name" } )
                    expect( updated.name ).toBe( "New Name" )
                } )

                it( "should delete a user", () => {
                    user = userService.create( { name: "Delete Me", email: "del@example.com" } )
                    userService.delete( user.id )
                    expect( userService.exists( user.id ) ).toBeFalse()
                } )
            } )

            describe( "Validation", () => {

                it( "should reject missing required fields", () => {
                    expect( () => {
                        userService.create( {} )
                    } ).toThrow( type: "ValidationException" )
                } )

                it( "should reject invalid email", () => {
                    expect( () => {
                        userService.create( { name: "Test", email: "not-an-email" } )
                    } ).toThrow()
                } )
            } )
        } )
    }
}
```

## Common Assertions

```boxlang
// Equality
expect( result ).toBe( "expected" )
expect( result ).notToBe( "other" )

// Type checks
expect( result ).toBeString()
expect( result ).toBeNumeric()
expect( result ).toBeBoolean()
expect( result ).toBeArray()
expect( result ).toBeStruct()

// Truthiness
expect( result ).toBeTrue()
expect( result ).toBeFalse()
expect( result ).toBeNull()
expect( result ).notToBeNull()
expect( result ).toBeEmpty()
expect( result ).notToBeEmpty()

// Struct/Array
expect( myStruct ).toHaveKey( "name" )
expect( myArray ).toHaveLength( 3 )
expect( myArray ).toInclude( "item" )

// Numeric
expect( num ).toBeGT( 0 )
expect( num ).toBeLT( 100 )
expect( num ).toBeGTE( 1 )
expect( num ).toBeLTE( 99 )

// Exceptions
expect( () => {
    methodThatThrows()
} ).toThrow()

expect( () => {
    methodThatThrows()
} ).toThrow( type: "CustomException", message: "Expected message" )

// Closures / custom matchers
expect( "hello world" ).toMatch( "hello" )  // regex match
```

## Focused and Skipped Tests

```boxlang
// Focus tests (only focused tests run)
fdescribe( "focused suite", () => {
    fit( "focused test runs", () => { /* ... */ } )
    it( "skipped - not focused", () => { /* ... */ } )
} )

// Skip tests
xdescribe( "skipped suite", () => {
    it( "wont run", () => { /* ... */ } )
} )

xit( "skipped test", () => { /* ... */ } )
```

## WireBox Integration in Tests

```boxlang
component extends="testbox.system.BaseSpec" {

    property name="wirebox" inject="wirebox"

    function run() {
        describe( "MyService with WireBox", () => {
            beforeAll( () => {
                variables.myService = wirebox.getInstance( "MyService" )
                // Or use getInstance() shortcut on BaseSpec
                variables.myService = getInstance( "MyService" )
            } )

            it( "should wire up correctly", () => {
                expect( myService ).toBeInstanceOf( "models.MyService" )
            } )
        } )
    }
}
```

## Test Labels

```boxlang
describe( "Integration tests", { labels: "integration,slow" }, () => {
    it( "test with label", { labels: "database" }, () => {
        // ...
    } )
} )
```

Run with label filter: `box testbox run labels=unit excludes=slow`
