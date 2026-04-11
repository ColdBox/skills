---
name: coldbox-testing-unit
description: "Use this skill when writing unit tests for ColdBox models and services using TestBox, applying the Arrange-Act-Assert pattern, using beforeAll/afterAll/beforeEach/afterEach lifecycle methods, isolating components with mocks, or testing individual functions in isolation."
---

# Unit Testing in ColdBox

## Overview

Unit tests verify individual units of code — functions, methods, services — in isolation. They use TestBox's `BaseSpec` and follow the AAA pattern (Arrange, Act, Assert). Mocks replace external dependencies to keep tests fast and deterministic.

## Language Mode Reference

Examples use **BoxLang (`.bx`)** syntax by default. Adapt for your target language:

| Concept | BoxLang (`.bx`) | CFML (`.cfc`) |
|---------|-----------------|---------------|
| Class declaration | `class [extends="..."] {` | `component [extends="..."] {` |
| DI annotation | `@inject` above `property name="svc";` | `property name="svc" inject="svc";` |
| View templates | `.bxm` suffix | `.cfm` / `.cfml` suffix |
| Tag prefix | `<bx:if>`, `<bx:output>`, `<bx:set>` | `<cfif>`, `<cfoutput>`, `<cfset>` |

> **CFML Compat Mode**: With BoxLang + CFML Compat module, `.bx` and `.cfc` files coexist freely. BoxLang-native classes use `class {}` (`.bx` files); CFML-compat classes use `component {}` (`.cfc` files).

## Basic Unit Test

```boxlang
component extends="testbox.system.BaseSpec" {

    function run() {
        describe( "UserValidator", () => {

            beforeAll( () => {
                variables.validator = new models.UserValidator()
            } )

            it( "should validate valid user data", () => {
                // Arrange
                userData = {
                    name: "John Doe",
                    email: "john@example.com",
                    age: 25
                }

                // Act
                result = validator.validate( userData )

                // Assert
                expect( result.isValid ).toBeTrue()
                expect( result.errors ).toBeEmpty()
            } )

            it( "should fail on missing name", () => {
                // Arrange
                userData = { email: "test@example.com" }

                // Act
                result = validator.validate( userData )

                // Assert
                expect( result.isValid ).toBeFalse()
                expect( result.errors ).toHaveKey( "name" )
            } )
        } )
    }
}
```

## Unit Test with Lifecycle

```boxlang
component extends="testbox.system.BaseSpec" {

    function run() {
        describe( "CalculatorService", () => {

            // Runs once before all tests in this describe block
            beforeAll( () => {
                variables.calculator = new models.CalculatorService()
            } )

            // Runs once after all tests
            afterAll( () => {
                structDelete( variables, "calculator" )
            } )

            // Runs before each individual test
            beforeEach( () => {
                variables.calculator.reset()
            } )

            // Runs after each individual test
            afterEach( () => {
                // cleanup state
            } )

            it( "should add numbers", () => {
                result = calculator.add( 5, 3 )
                expect( result ).toBe( 8 )
            } )

            it( "should subtract numbers", () => {
                result = calculator.subtract( 10, 4 )
                expect( result ).toBe( 6 )
            } )

            it( "should throw on division by zero", () => {
                expect( () => {
                    calculator.divide( 10, 0 )
                } ).toThrow( type: "MathException" )
            } )
        } )
    }
}
```

## Unit Test with Mocked Dependencies

```boxlang
component extends="testbox.system.BaseSpec" {

    function run() {
        describe( "OrderService", () => {

            beforeAll( () => {
                // Mock external dependencies
                variables.mockPaymentGateway = createMock( "models.PaymentGateway" )
                variables.mockEmailService   = createMock( "models.EmailService" )

                // Create service with mocked deps
                variables.orderService = new models.OrderService(
                    paymentGateway: mockPaymentGateway,
                    emailService: mockEmailService
                )
            } )

            it( "should process order and send confirmation", () => {
                // Arrange - stub payment to return success
                mockPaymentGateway.$( "charge" ).$results( { success: true, transactionId: "TXN123" } )
                mockEmailService.$( "sendOrderConfirmation" )

                // Act
                order = orderService.placeOrder( {
                    items: [ { productId: 1, qty: 2 } ],
                    total: 49.99,
                    customerId: 100
                } )

                // Assert service results
                expect( order.status ).toBe( "confirmed" )
                expect( order.transactionId ).toBe( "TXN123" )

                // Assert collaborator was called
                expect( mockEmailService.$once( "sendOrderConfirmation" ) ).toBeTrue()
            } )

            it( "should not send email when payment fails", () => {
                mockPaymentGateway.$( "charge" ).$results( { success: false, error: "Card declined" } )
                mockEmailService.$( "sendOrderConfirmation" )

                expect( () => {
                    orderService.placeOrder( { total: 100, customerId: 1 } )
                } ).toThrow( type: "PaymentException" )

                expect( mockEmailService.$never( "sendOrderConfirmation" ) ).toBeTrue()
            } )
        } )
    }
}
```

## Testing Private/Protected Methods

```boxlang
component extends="testbox.system.BaseSpec" {

    function run() {
        describe( "MyService private methods", () => {

            beforeAll( () => {
                // Use makePublic() to expose private methods for testing
                variables.service = createMock( "models.MyService" ).makePublic( "sanitizeInput", "formatResponse" )
            } )

            it( "should sanitize input", () => {
                result = service.sanitizeInput( "<script>alert('xss')</script>" )
                expect( result ).notToInclude( "<script>" )
            } )
        } )
    }
}
```

## Parameterized Tests

```boxlang
component extends="testbox.system.BaseSpec" {

    function run() {
        describe( "EmailValidator", () => {

            beforeAll( () => {
                variables.validator = new models.EmailValidator()
            } )

            // Use loop to test multiple cases
            validEmails = [
                "user@example.com",
                "user.name+tag@example.com",
                "user@subdomain.example.com"
            ]

            for ( email in validEmails ) {
                // Capture loop variable for closure scope
                capturedEmail = email
                it( "should accept valid email: #capturedEmail#", () => {
                    expect( validator.isValid( capturedEmail ) ).toBeTrue()
                } )
            }

            invalidEmails = [ "not-an-email", "@missing-local.com", "missing-at-sign.com" ]

            for ( email in invalidEmails ) {
                capturedEmail = email
                it( "should reject invalid email: #capturedEmail#", () => {
                    expect( validator.isValid( capturedEmail ) ).toBeFalse()
                } )
            }
        } )
    }
}
```

## Quick Assertions Reference

```boxlang
// Equality
expect( val ).toBe( expected )
expect( val ).notToBe( expected )

// Type
expect( val ).toBeTypeOf( "string" )
expect( val ).toBeInstanceOf( "models.User" )

// Collections
expect( struct ).toHaveKey( "id" )
expect( array ).toHaveLength( 3 )
expect( array ).toInclude( "item" )

// Numeric ranges
expect( num ).toBeGT( 0 )
expect( num ).toBeLTE( 100 )

// Null/empty
expect( val ).toBeNull()
expect( val ).toBeEmpty()
expect( val ).notToBeEmpty()
```
