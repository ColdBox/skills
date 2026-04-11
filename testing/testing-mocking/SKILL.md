---
name: coldbox-testing-mocking
description: "Use this skill when using MockBox to create mocks and stubs in ColdBox tests, stubbing method return values with .$(), verifying method call counts with $once()/$never()/$times(), creating spy objects, using $args() for argument-specific stubs, or resetting mocks between tests."
---

# Mocking with MockBox in ColdBox

## Overview

MockBox is TestBox's built-in mocking framework for creating test doubles (mocks, stubs, spies). Use it to replace external dependencies — databases, email services, APIs — with controlled test doubles that return predictable values.

## Language Mode Reference

Examples use **BoxLang (`.bx`)** syntax by default. Adapt for your target language:

| Concept | BoxLang (`.bx`) | CFML (`.cfc`) |
|---------|-----------------|---------------|
| Class declaration | `class [extends="..."] {` | `component [extends="..."] {` |
| DI annotation | `@inject` above `property name="svc";` | `property name="svc" inject="svc";` |
| View templates | `.bxm` suffix | `.cfm` / `.cfml` suffix |
| Tag prefix | `<bx:if>`, `<bx:output>`, `<bx:set>` | `<cfif>`, `<cfoutput>`, `<cfset>` |

> **CFML Compat Mode**: With BoxLang + CFML Compat module, `.bx` and `.cfc` files coexist freely. BoxLang-native classes use `class {}` (`.bx` files); CFML-compat classes use `component {}` (`.cfc` files).

## Creating Mocks

```boxlang
component extends="testbox.system.BaseSpec" {

    function run() {
        describe( "MockBox basics", () => {

            beforeAll( () => {
                // Create a full mock — all methods stubbed, return null by default
                variables.mockUserService = createMock( "models.UserService" )

                // Create a spy — real methods execute unless explicitly stubbed
                variables.spyEmailService = createSpy( new models.EmailService() )

                // Create empty mock (for interfaces, abstract classes)
                variables.mockInterface = createEmptyMock( "interfaces.IRepository" )
            } )
        } )
    }
}
```

## Stubbing Return Values

```boxlang
// Stub a method to return a value
mockService.$( "findById" ).$results( { id: 1, name: "Alice" } )

// Stub with multiple sequential results (cycles through)
mockService.$( "getNextItem" ).$results( "first", "second", "third" )

// Stub with a closure for dynamic responses
mockService.$( "calculate" ).$results( ( args ) => args[1] * 2 )

// Stub to throw an exception
mockService.$( "dangerousMethod" ).$throws( type: "CustomException", message: "Failed" )

// Stub based on specific arguments
mockService
    .$( "find" )
    .$args( id: 1 ).$results( { id: 1, name: "Alice" } )
    .$args( id: 2 ).$results( { id: 2, name: "Bob" } )

// Stub method to return void (for methods with no return)
mockService.$( "sendEmail" )

// Stub with a callback
mockService.$( "process", function( args ) {
    // Custom logic
    return { processed: true, count: args[1].len() }
} )
```

## Verifying Method Calls

```boxlang
component extends="testbox.system.BaseSpec" {

    function run() {
        describe( "Verification", () => {

            it( "should verify a method was called once", () => {
                mockService.$( "sendEmail" )

                orderService.placeOrder( orderData )

                // Called exactly once
                expect( mockService.$once( "sendEmail" ) ).toBeTrue()
            } )

            it( "should verify exact call count", () => {
                mockService.$( "log" )

                service.processItems( [ 1, 2, 3 ] )

                // Called exactly 3 times
                expect( mockService.$times( 3, "log" ) ).toBeTrue()
            } )

            it( "should verify a method was never called", () => {
                mockService.$( "sendAlert" )

                service.processNormalItem( item )

                expect( mockService.$never( "sendAlert" ) ).toBeTrue()
            } )

            it( "should verify at least N times", () => {
                mockService.$( "save" )

                service.bulkSave( items )

                expect( mockService.$atLeast( 1, "save" ) ).toBeTrue()
            } )
        } )
    }
}
```

## Inspecting Call Arguments

```boxlang
it( "should pass correct args to payment gateway", () => {
    mockPayment.$( "charge" ).$results( { success: true, txnId: "T123" } )

    orderService.placeOrder( { total: 99.99, customerId: 5 } )

    // Inspect what args were passed
    callLog = mockPayment.$callLog().charge

    expect( callLog ).toHaveLength( 1 )
    expect( callLog[1][1] ).toBe( 99.99 )     // first arg of first call
    expect( callLog[1].customerId ).toBe( 5 )  // named arg
} )
```

## Resetting Mocks

```boxlang
describe( "with mock resets", () => {

    beforeEach( () => {
        // Reset call counters but keep stubs
        mockService.$reset()

        // Re-stub for this test
        mockService.$( "findById" ).$results( testUser )
    } )

    it( "test 1", () => {
        service.doSomething()
        expect( mockService.$once( "findById" ) ).toBeTrue()
    } )

    it( "test 2 - mock is clean", () => {
        service.doSomething()
        // call count restarted from 0
        expect( mockService.$once( "findById" ) ).toBeTrue()
    } )
} )
```

## Full Integration Example

```boxlang
component extends="testbox.system.BaseSpec" {

    function run() {
        describe( "OrderService with mocks", () => {

            beforeAll( () => {
                // Create mocks
                variables.mockPaymentGateway = createMock( "models.PaymentGateway" )
                variables.mockEmailService   = createMock( "models.EmailService" )
                variables.mockInventory      = createMock( "models.InventoryService" )

                // Inject mocks into service (constructor injection)
                variables.orderService = new models.OrderService(
                    paymentGateway: mockPaymentGateway,
                    emailService:   mockEmailService,
                    inventory:      mockInventory
                )
            } )

            beforeEach( () => {
                // Reset all mocks before each test
                mockPaymentGateway.$reset()
                mockEmailService.$reset()
                mockInventory.$reset()

                // Default stubs
                mockInventory.$( "checkStock" ).$results( true )
                mockPaymentGateway.$( "charge" ).$results( { success: true, transactionId: "TXN001" } )
                mockEmailService.$( "sendOrderConfirmation" )
            } )

            it( "should process valid order", () => {
                order = orderService.placeOrder( {
                    items: [ { productId: 1, qty: 1 } ],
                    total: 29.99,
                    customerId: 42
                } )

                expect( order.status ).toBe( "confirmed" )
                expect( mockPaymentGateway.$once( "charge" ) ).toBeTrue()
                expect( mockEmailService.$once( "sendOrderConfirmation" ) ).toBeTrue()
            } )

            it( "should fail when out of stock", () => {
                // Override default stock stub
                mockInventory.$( "checkStock" ).$results( false )

                expect( () => {
                    orderService.placeOrder( { items: [ { productId: 1, qty: 1 } ] } )
                } ).toThrow( type: "OutOfStockException" )

                // Payment should NOT be charged
                expect( mockPaymentGateway.$never( "charge" ) ).toBeTrue()
            } )
        } )
    }
}
```

## MockBox Quick Reference

| Method | Purpose |
|--------|---------|
| `createMock( "path.to.Class" )` | Create mock of a class (all methods stubbed) |
| `createSpy( object )` | Create spy on real object |
| `createEmptyMock( "path" )` | Create empty mock (no real implementation) |
| `mock.$( "method" )` | Start stubbing a method |
| `.$results( val )` | Set method return value |
| `.$args( ... )` | Match specific arguments |
| `.$throws( type, message )` | Make method throw exception |
| `.$reset()` | Reset call counters |
| `mock.$once( "method" )` | Returns true if called exactly once |
| `mock.$never( "method" )` | Returns true if never called |
| `mock.$times( n, "method" )` | Returns true if called n times |
| `mock.$atLeast( n, "method" )` | Returns true if called at least n times |
| `mock.$callLog()` | Get struct of all method call logs |

## When to Mock

- **Mock**: Databases, file system, external HTTP APIs, email/SMS services, slow operations, third-party services
- **Don't Mock**: Value objects, simple utilities, the system under test itself
- **Use Spies**: When you want real behavior but also want to verify calls
