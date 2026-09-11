---
name: testbox-expectations
description: "Use this skill when writing fluent expectations in TestBox using expect(), the collection modes (expectAll, expectAny, expectSome, expectNone), expectation context with withContext(), and the built-in matchers: toBe, toBeTrue, toBeFalse, toBeTruthy, toBeFalsy, toBeNull, toBeArray, toBeStruct, toBeEmpty, toHaveLength, toHaveSize, toBeTypeOf, toBeInstanceOf, toBeSameInstanceAs, toHaveKey, toHaveDeepKey, toInclude, toIncludeWithCase, toIncludeAll, toIncludeAny, toIncludeNone, toMatch, toBeGT, toBeGTE, toBeLT, toBeLTE, toBeBetween, toBeCloseTo, toSatisfy, toThrow, toThrowMatching; the BoxLang-only Set matchers (toBeASet, toEqualSet, toBeSubsetOf, toBeSupersetOf, toBeDisjointFrom, toHaveUnion, toHaveIntersection, toHaveDifference, toHaveSymmetricDifference); the BoxLang-only Range matchers (toBeRange, toContainValue, toContainRange, toBeInRange, toBeBeforeRange, toBeAfterRange, toBeBounded, toBeUnbounded, toBeHalfBounded, toBeIterable, toBeAscending, toBeDescending, toHaveStep, toClampTo); the BoxLang-only Data Navigator matchers (toHavePath, toHavePathValue, toHavePathType, toHavePathSatisfying, path, queryPath); the not operator (notToBe, notToBeEmpty, etc.); chaining matchers on one expect(); or creating custom matchers with addMatchers()."
applyTo: "**/tests/**/*.{bx,bxm,cfc,cfm,cfml}"
---

# TestBox Expectations — Fluent Assertion DSL

## When to Use This Skill

- Writing fluent `expect( actual ).toBeXxx()` assertions in BDD or xUnit bundles
- Chaining multiple matchers on a single `expect()` call
- Asserting over a collection with `expectAll()`, `expectAny()`, `expectSome()` or `expectNone()`
- Labelling an expectation with `withContext()` so failures say *which* one broke
- Using the `not` operator to negate any matcher (`notToBe`, `notToBeEmpty`, etc.)
- Asserting against BoxLang `Set`, `Range` and nested data structures (BoxLang-only matchers)
- Building and registering custom matchers with `addMatchers()`

---

## Core Pattern

```boxlang
// expect( actual ).matcher( expected )
expect( result ).toBe( "hello" )

// Chained matchers on same actual value
expect( myArray )
    .toBeArray()
    .notToBeEmpty()
    .toHaveLength( 3 )

// Negative operators — prefix any matcher with "not"
expect( result ).notToBe( "wrong" )
expect( list ).notToBeEmpty()
expect( value ).notToBeNull()
```

---

## Expectation Context — `withContext()`

*TestBox 7.1+. Works on every engine.*

When one spec makes the same assertion several times, a bare failure message cannot tell you
*which* one broke. `withContext()` labels the expectation, and the label is prefixed to whatever
failure message the matcher produces.

```boxlang
expect( response.status ).withContext( "POST /users" ).toBe( 201 )
expect( response.status ).withContext( "GET /users/1" ).toBe( 200 )
// Failure reads: GET /users/1 — Expected [404] to be [200]
```

The context flows through the whole chain — negated matchers and custom matchers included:

```boxlang
// Loop over cases without losing track of which one failed
for ( var tc in testCases ) {
    expect( parser.parse( tc.input ) )
        .withContext( "case: #tc.name#" )
        .notToBeNull()
        .toBe( tc.expected )
}

// Custom matchers inherit it too
expect( user ).withContext( "admin fixture" ).toBeValidEmail()
```

> `withContext()` returns the expectation, so chain it immediately after `expect()` and before
> the first matcher.

---

## All Built-in Matchers

### Equality

```boxlang
expect( result ).toBe( expected )              // case-insensitive equality (simple and complex)
expect( result ).toBeWithCase( expected )      // case-sensitive equality
expect( result ).notToBe( expected )
expect( result ).notToBeWithCase( expected )
```

### Boolean / Truthiness

```boxlang
expect( value ).toBeTrue()
expect( value ).toBeFalse()
expect( value ).toBeNull()
expect( value ).notToBeNull()

// TestBox 7.1+ — loose truthiness, not a strict boolean check.
// Falsy = false, 0, "" (empty string) and null. Everything else is truthy.
expect( "hello" ).toBeTruthy()
expect( [ 1 ] ).toBeTruthy()
expect( 0 ).toBeFalsy()
expect( "" ).toBeFalsy()
expect( false ).toBeFalsy()

// notToBeTruthy() is the same assertion as toBeFalsy()
expect( 0 ).notToBeTruthy()
```

> Use `toBeTrue()` when the value must literally be a boolean `true`; use `toBeTruthy()` when
> any non-empty, non-zero value should pass.

### Emptiness & Length

```boxlang
expect( collection ).toBeEmpty()        // array, struct, string, query
expect( collection ).notToBeEmpty()
expect( collection ).toHaveLength( n )
expect( collection ).notToHaveLength( n )

// TestBox 7.1+ — toHaveSize() is an alias of toHaveLength(), for collections
// that read better as "size" than as "length"
expect( userMap ).toHaveSize( 3 )
expect( results ).notToHaveSize( 0 )
```

### Type Checks

```boxlang
// Generic type (uses CF isValid() under the hood)
expect( value ).toBeTypeOf( "array" )   // array, struct, component, numeric, boolean, date, uuid…
expect( value ).notToBeTypeOf( "array" )

// Dynamic shorthand — toBeXxx() where Xxx is any isValid() type
expect( [1,2,3] ).toBeArray()
expect( {} ).toBeStruct()
expect( "03/01/1990" ).toBeUsDate()
expect( createUUID() ).toBeUuid()
expect( 42 ).toBeNumeric()
expect( "hello" ).toBeString()
expect( true ).toBeBoolean()

// Instance check — is it OF this class/interface?
expect( myObj ).toBeInstanceOf( "models.UserService" )
expect( myObj ).notToBeInstanceOf( "models.OtherService" )

// TestBox 7.1+ — identity check: is it the SAME object, not merely an equal one?
var singleton = getInstance( "UserService" )
expect( getInstance( "UserService" ) ).toBeSameInstanceAs( singleton )   // singleton proof
expect( getInstance( "UserRequest" ) ).notToBeSameInstanceAs( singleton ) // transient proof
```

> `toBe()` compares values, `toBeInstanceOf()` compares types, `toBeSameInstanceAs()` compares
> object identity. Reach for the last one when testing WireBox scopes, caching, or any code that
> must hand back the very same object.

### Struct Key Existence

```boxlang
expect( myStruct ).toHaveKey( "email" )
expect( myStruct ).notToHaveKey( "password" )

// Deep key search (nested structs)
expect( myStruct ).toHaveDeepKey( "address" )
expect( myStruct ).notToHaveDeepKey( "ssn" )
```

### String / Array Inclusion

```boxlang
// Case-insensitive
expect( "Hello World" ).toInclude( "hello" )
expect( [1,2,3] ).toInclude( 2 )
expect( "Hello World" ).notToInclude( "foo" )

// Case-sensitive
expect( "Hello World" ).toIncludeWithCase( "Hello" )
expect( "Hello World" ).notToIncludeWithCase( "hello" )   // fails — "hello" != "Hello"
```

**Multi-needle inclusion** (TestBox 7.1+). Each takes an **array** of needles and is
case-insensitive, so you stop writing three `toInclude()` calls in a row:

```boxlang
// Every needle must be present
expect( response.body ).toIncludeAll( [ "id", "name", "email" ] )
expect( permissions ).toIncludeAll( [ "read", "write" ] )

// At least one needle must be present
expect( logOutput ).toIncludeAny( [ "WARN", "ERROR", "FATAL" ] )

// No needle may be present — good for leak assertions
expect( serializeJSON( memento ) ).toIncludeNone( [ "password", "salt", "apiKey" ] )
```

### Regular Expressions

```boxlang
expect( "foo@bar.com" ).toMatch( "^[^@]+@[^@]+" )        // case-insensitive
expect( "Hello" ).toMatchWithCase( "^Hello" )             // case-sensitive
expect( "123" ).notToMatch( "[a-z]" )
expect( "ABC" ).notToMatchWithCase( "[a-z]" )
```

### Numeric Comparisons

```boxlang
expect( 5 ).toBeGT( 4 )
expect( 5 ).toBeGTE( 5 )
expect( 5 ).toBeLT( 6 )
expect( 5 ).toBeLTE( 5 )
expect( 5 ).toBeBetween( 1, 10 )
expect( 5 ).notToBeBetween( 20, 30 )

// Approximate equality (numbers or dates)
expect( 3.14159 ).toBeCloseTo( expected: 3.14, delta: 0.01 )
expect( now() ).toBeCloseTo( expected: dateAdd( "s", 1, now() ), delta: 2, datepart: "s" )
```

### Exceptions

```boxlang
// Any exception
expect( () => {
    service.riskyOperation()
} ).toThrow()

// Specific type
expect( () => {
    service.delete( 999 )
} ).toThrow( type: "NotFoundException" )

// Type + message regex
expect( () => {
    service.delete( 999 )
} ).toThrow( type: "NotFoundException", regex: "999" )

// Assert NO exception
expect( () => {
    service.safeOperation()
} ).notToThrow()
```

**Predicate-based exception assertions** (TestBox 7.1+). When `type` + `regex` is not expressive
enough, `toThrowMatching()` hands the caught exception to a closure and passes if it returns true:

```boxlang
expect( () => service.delete( 999 ) ).toThrowMatching( ( e ) => e.errorCode == "E404" )

// Inspect any part of the exception struct
expect( () => api.call() ).toThrowMatching( ( e ) => {
    return e.type == "HTTPException" && e.extendedInfo contains "timeout"
} )

// Negated: it threw, but not one matching the predicate
expect( () => service.delete( 1 ) ).notToThrowMatching( ( e ) => e.errorCode == "E404" )
```

> `toThrowMatching()` still fails if the closure throws nothing at all.

---

## Chaining Multiple Matchers

All matchers return the expectation object, so you can chain:

```boxlang
expect( response )
    .toBeStruct()
    .toHaveKey( "status" )
    .toHaveKey( "data" )

expect( users )
    .toBeArray()
    .notToBeEmpty()
    .toHaveLength( 5 )

expect( email )
    .toBeString()
    .notToBeEmpty()
    .toMatch( ".+@.+\..+" )
```

---

## Collection Expectations — `expectAll` / `expectAny` / `expectSome` / `expectNone`

Each of these takes an array or a struct, applies the chained matcher to **every element**, and
then judges the pass count differently. All four accept any matcher, negated matchers included.

| Starter | Passes when | Since |
|---|---|---|
| `expectAll( c )` | **every** element passes | 7.0 |
| `expectAny( c )` | **at least one** element passes | 7.1 |
| `expectSome( c, min, max )` | between `min` and `max` elements pass (`max: 0` = no upper bound) | 7.1 |
| `expectNone( c )` | **zero** elements pass | 7.1 |

```boxlang
// Every element — all values are even numbers
expectAll( [ 2, 4, 6 ] ).toSatisfy( ( x ) => x % 2 == 0 )
expectAll( { a: "foo", b: "bar" } ).notToBeEmpty()
expectAll( userService.listAll() ).toSatisfy( ( user ) => user.isActive == true )

// At least one element — the search returned something relevant
expectAny( results ).toSatisfy( ( r ) => r.score > 0.9 )
expectAny( [ "draft", "published", "draft" ] ).toBe( "published" )

// A bounded number of elements
expectSome( users, 1, 3 ).toSatisfy( ( u ) => u.role == "admin" )  // 1 to 3 admins
expectSome( orders, 2 ).toSatisfy( ( o ) => o.status == "shipped" ) // at least 2, no ceiling

// Zero elements — nothing leaked, nothing failed
expectNone( auditLog ).toInclude( "password" )
expectNone( jobs ).toSatisfy( ( j ) => j.status == "failed" )
```

Failure messages report the mode, the pass count against the total, and the index (array) or key
(struct) of each offending element, so you do not have to bisect the collection yourself.

> The collection must be an array or a struct; anything else fails the expectation outright with
> a type message. Struct elements are evaluated by **value**, and the key is what gets reported.

---

## BoxLang-Only: Set Matchers

> **BoxLang only.** These operate on BoxLang `Set` objects, built with `setOf( ... )` or
> `setNew( type: "linked", values: [ ... ] )`. Lucee and Adobe ColdFusion have no `Set` type, so
> put these specs in a `.bx` file — TestBox skips `.bx` bundles entirely on non-BoxLang engines,
> which keeps the suite green everywhere.

```boxlang
// tests/specs/PermissionsSpec.bx
class extends="testbox.system.BaseSpec" {

    function run() {
        describe( "Permission sets", () => {

            it( "models permissions as a set", () => {
                var granted  = setOf( "read", "write", "delete" )
                var required = setOf( "read", "write" )

                // Is it a Set at all?
                expect( granted ).toBeASet()
                expect( [ "read", "write" ] ).notToBeASet()   // an array is not a Set

                // Equality is order-independent
                expect( granted ).toEqualSet( setOf( "delete", "read", "write" ) )

                // Containment
                expect( required ).toBeSubsetOf( granted )
                expect( granted ).toBeSupersetOf( required )

                // No overlap at all
                expect( setOf( "read" ) ).toBeDisjointFrom( setOf( "write", "delete" ) )
            } )

            it( "asserts the result of set algebra", () => {
                var a = setOf( 1, 2, 3 )
                var b = setOf( 3, 4, 5 )

                // expect( actual ).matcher( other, expectedResult )
                expect( a ).toHaveUnion( b, setOf( 1, 2, 3, 4, 5 ) )
                expect( a ).toHaveIntersection( b, setOf( 3 ) )
                expect( a ).toHaveDifference( b, setOf( 1, 2 ) )              // a minus b
                expect( a ).toHaveSymmetricDifference( b, setOf( 1, 2, 4, 5 ) ) // in one, not both
            } )

        } )
    }

}
```

| Matcher | Asserts |
|---|---|
| `toBeASet()` | The actual value is a BoxLang `Set` |
| `toEqualSet( expected )` | Same elements, order-independent |
| `toBeSubsetOf( expected )` | Every element of actual is in `expected` |
| `toBeSupersetOf( expected )` | Every element of `expected` is in actual |
| `toBeDisjointFrom( expected )` | The two sets share no element |
| `toHaveUnion( other, expected )` | `actual ∪ other` equals `expected` |
| `toHaveIntersection( other, expected )` | `actual ∩ other` equals `expected` |
| `toHaveDifference( other, expected )` | `actual − other` equals `expected` |
| `toHaveSymmetricDifference( other, expected )` | Elements in exactly one of the two equals `expected` |

> A set is both a subset and a superset of itself, and the empty set is a subset of everything —
> the matchers follow the mathematics, not an intuition about "strict" containment.

---

## BoxLang-Only: Range Matchers

> **BoxLang only.** Ranges are built with BoxLang's `..` operator and `.step( n )` — there is no
> `rangeNew()` BIF. `..` is not valid CFML syntax, so these specs must live in a `.bx` file.

```boxlang
// tests/specs/RangeSpec.bx
class extends="testbox.system.BaseSpec" {

    function run() {
        describe( "Ranges", () => {

            it( "builds and identifies ranges", () => {
                var scores  = 1..10              // bounded, ascending, step 1
                var byFives = (0..100).step( 5 ) // explicit step
                var reverse = 10..1              // descending
                var letters = "a".."z"           // character range
                var anytime = ..                 // fully unbounded
                var upTo    = ..10               // half-bounded (no start)
                var from    = 1..                // half-bounded (no end)

                expect( scores ).toBeRange()
                expect( 42 ).notToBeRange()
            } )

            it( "asserts membership both ways", () => {
                var scores = 1..10

                // Range contains a value
                expect( scores ).toContainValue( 5 )
                expect( scores ).toContainValue( 10 )    // bounds are inclusive
                expect( scores ).notToContainValue( 11 )

                // Value is in a range — same assertion, read from the value's side
                expect( 5 ).toBeInRange( scores )
                expect( "m" ).toBeInRange( "a".."z" )

                // Range fully contains another range
                expect( scores ).toContainRange( 3..7 )
            } )

            it( "asserts relative position", () => {
                expect( 1..5 ).toBeBeforeRange( 10..20 )
                expect( 15..20 ).toBeAfterRange( 1..10 )
            } )

            it( "asserts shape and direction", () => {
                // Assign unbounded and exclusive ranges to a variable first —
                // they do not read well inline as a call argument
                var anytime = ..
                var upTo    = ..10
                var empty   = 1>..<1

                expect( 1..10 ).toBeBounded()      // both ends known
                expect( anytime ).toBeUnbounded()  // neither end known
                expect( upTo ).toBeHalfBounded()   // exactly one end known
                expect( 1..10 ).toBeIterable()     // can be walked element by element
                expect( 1..10 ).toBeAscending()
                expect( 10..1 ).toBeDescending()
                expect( empty ).toBeEmpty()        // exclusive bounds, no members
            } )

            it( "asserts step and clamping", () => {
                expect( (0..100).step( 5 ) ).toHaveStep( 5 )
                expect( 1..10 ).toHaveStep( 1 )               // default step

                // toClampTo( value, expectedResult )
                expect( 1..10 ).toClampTo( 15, 10 )   // above the range clamps to the high bound
                expect( 1..10 ).toClampTo( 0, 1 )     // below clamps to the low bound
                expect( 1..10 ).toClampTo( 5, 5 )     // inside is returned untouched
            } )

        } )
    }

}
```

| Matcher | Asserts |
|---|---|
| `toBeRange()` | The actual value is a BoxLang `Range` |
| `toContainValue( value )` | The range includes `value` (bounds inclusive) |
| `toContainRange( expected )` | The range fully contains `expected` |
| `toBeInRange( range )` | The actual **value** falls inside `range` |
| `toBeBeforeRange( expected )` | The actual range ends at or before `expected` starts |
| `toBeAfterRange( expected )` | The actual range starts at or after `expected` ends |
| `toBeBounded()` | Both endpoints are defined |
| `toBeUnbounded()` | Neither endpoint is defined (`..`) |
| `toBeHalfBounded()` | Exactly one endpoint is defined (`..10`, `1..`) |
| `toBeIterable()` | The range can be iterated |
| `toBeAscending()` / `toBeDescending()` | Direction of travel |
| `toHaveStep( step )` | The range's step value (default `1`) |
| `toClampTo( value, expected )` | Clamping `value` into the range yields `expected` |

> `toBeEmpty()` — the ordinary matcher — also understands ranges: an exclusive range such as
> `1>..<1` has no members.

---

## BoxLang-Only: Data Navigator Matchers

> **BoxLang only.** These use BoxLang's data navigator and throw
> `TestBox.BoxLangFeatureNotAvailable` on any other runtime.

Path expressions are JSONPath-style and support dot notation, **1-based** array indexing,
recursive descent (`..key`), wildcards (`[*]`), inclusive slices (`[1:3]`, `[2:]`) and filters
(`[?(@.age > 18)]`).

```boxlang
// tests/specs/ConfigSpec.bx
class extends="testbox.system.BaseSpec" {

    function run() {
        describe( "Config document", () => {

            var config = {
                "app"   : { "name": "TestApp", "settings": { "debug": true, "port": 8080 } },
                "users" : [ { "name": "Alice", "age": 30 }, { "name": "Bob", "age": 25 } ]
            }

            it( "asserts structure without drilling by hand", () => {
                // Does the path resolve to anything?
                expect( config ).toHavePath( "app.settings.debug" )
                expect( config ).toHavePath( "users[1].name" )   // 1-BASED: [1] is Alice
                expect( config ).notToHavePath( "app.missing.field" )

                // Value at the path
                expect( config ).toHavePathValue( "app.name", "TestApp" )
                expect( config ).toHavePathValue( "users[2].name", "Bob" )

                // Type at the path — accepts aliases such as "num" and "bool"
                expect( config ).toHavePathType( "app.settings.port", "numeric" )
                expect( config ).toHavePathType( "users", "array" )

                // Arbitrary predicate on the value at the path
                expect( config ).toHavePathSatisfying( "app.settings.port", ( p ) -> p > 1000 )
            } )

            it( "navigates and then keeps asserting", () => {
                // path() returns an expectation on the FIRST match — chain any matcher onto it
                expect( config ).path( "app.name" ).toBe( "TestApp" )
                expect( config ).path( "app.settings.port" ).toBeGT( 8000 )
                expect( config ).path( "users" ).toHaveLength( 2 )
                expect( config ).path( "nonexistent" ).toBeNull()   // no match resolves to null

                // queryPath() returns an expectation on an ARRAY of ALL matches
                expect( config ).queryPath( "users[*].name" ).toHaveLength( 2 )
                expect( config ).queryPath( "users[*].name" ).toInclude( "Alice" )
                expect( config ).queryPath( "nonexistent" ).toBeEmpty()
            } )

            it( "supports the full path grammar", () => {
                var data = {
                    "items"    : [ "a", "b", "c", "d", "e" ],
                    "products" : [
                        { "name": "Widget", "price": 10 },
                        { "name": "Gadget", "price": 25 }
                    ],
                    "api" : { "v1": { "users": { "id": "v1" } }, "v2": { "users": { "id": "v2" } } }
                }

                expect( data ).queryPath( "items[1:2]" ).toHaveLength( 2 )               // inclusive slice
                expect( data ).queryPath( "products[?(@.price > 20)].name" ).toInclude( "Gadget" )
                expect( data ).queryPath( "..users" ).toHaveLength( 2 )                  // recursive descent
                expect( data ).path( "items[5]" ).toBe( "e" )                            // 1-based: [5] is the 5th
            } )

        } )
    }

}
```

| Matcher | Asserts |
|---|---|
| `toHavePath( path )` | The path resolves to at least one value |
| `toHavePathValue( path, expected )` | The value at `path` equals `expected` |
| `toHavePathType( path, type )` | The value at `path` is of `type` (`string`, `numeric`/`num`, `boolean`/`bool`, `struct`, `array`, …) |
| `toHavePathSatisfying( path, predicate )` | The closure returns true for the value at `path` |
| `path( path )` | **Returns a new expectation** on the first match — chain any matcher |
| `queryPath( path )` | **Returns a new expectation** on an array of every match |

> `path()` and `queryPath()` are navigators, not assertions: they hand back a fresh expectation,
> so every matcher in this skill, `not` included, works on the result. They combine with
> `withContext()` too.

---

## Custom Matchers

Register custom matchers in `beforeAll()` or `beforeEach()` for global availability:

### Inline Struct

```boxlang
function beforeAll() {
    addMatchers( {

        toBeValidEmail: function( expectation, args = {} ) {
            expectation.message = isNull( args.message )
                ? "[#expectation.actual#] is not a valid email address"
                : args.message
            var passes = isValid( "Email", expectation.actual )
            return expectation.isNot ? !passes : passes
        },

        toBePositive: function( expectation, args = {} ) {
            expectation.message = "[#expectation.actual#] is not a positive number"
            return expectation.isNot
                ? expectation.actual <= 0
                : expectation.actual > 0
        },

        toHaveStatus: function( expectation, args = {} ) {
            var expected = args[ 1 ] ?: args.status ?: 200
            expectation.message = "Expected HTTP status [#expected#] but got [#expectation.actual.getStatusCode()#]"
            var passes = expectation.actual.getStatusCode() == expected
            return expectation.isNot ? !passes : passes
        }

    } )
}
```

Usage:

```boxlang
expect( "alice@example.com" ).toBeValidEmail()
expect( -5 ).notToBePositive()
expect( event.getResponse() ).toHaveStatus( 200 )
```

### Class-Based Matchers (Reusable Library)

```boxlang
// tests/helpers/AppMatchers.cfc
component {

    boolean function toBeActiveUser( required expectation, args = {} ) {
        expectation.message = "Expected user to be active"
        var passes = expectation.actual.keyExists( "isActive" ) && expectation.actual.isActive
        return expectation.isNot ? !passes : passes
    }

    boolean function toHavePermission( required expectation, args = {} ) {
        var permission = args[ 1 ] ?: ""
        expectation.message = "Expected user to have permission [#permission#]"
        var passes = expectation.actual.permissions.findNoCase( permission ) > 0
        return expectation.isNot ? !passes : passes
    }

}
```

```boxlang
function beforeAll() {
    addMatchers( "tests.helpers.AppMatchers" )
    // or: addMatchers( new tests.helpers.AppMatchers() )
}

// Usage
expect( adminUser ).toBeActiveUser()
expect( adminUser ).toHavePermission( "MANAGE_USERS" )
expect( guestUser ).notToHavePermission( "MANAGE_USERS" )
```

---

## Custom Matcher Signature Rules

Every custom matcher function must follow this contract:

| Rule | Detail |
|---|---|
| Signature | `boolean function myMatcher( required expectation, args = {} )` |
| Return | `true` = passes, `false` = fails |
| `expectation.actual` | The value passed into `expect()` |
| `expectation.isNot` | `true` when called as `notToMyMatcher()` |
| `expectation.message` | Set this to provide a custom failure message |
| `args` | All arguments passed to the matcher call |

```boxlang
// Template for a custom matcher
boolean function toMeetCriteria( required expectation, args = {} ) {
    expectation.message = args.message ?: "Custom failure message"
    var passes = /* your evaluation */ true
    return expectation.isNot ? !passes : passes
}
```

---

## Matcher Quick Reference

| Matcher | Description |
|---|---|
| `toBe( val )` | Equality (case-insensitive for strings) |
| `toBeWithCase( val )` | Equality (case-sensitive) |
| `toBeTrue()` / `toBeFalse()` | Strict boolean assertion |
| `toBeTruthy()` / `toBeFalsy()` | Loose truthiness (falsy = `false`, `0`, `""`, null) — 7.1+ |
| `toBeNull()` | Null check |
| `toBeEmpty()` | Empty check (array/struct/string/query/range) |
| `toHaveLength( n )` | Size assertion |
| `toHaveSize( n )` | Alias of `toHaveLength()` — 7.1+ |
| `toBeTypeOf( type )` | Type via `isValid()` |
| `toBe{Type}()` | e.g. `toBeArray()`, `toBeStruct()` |
| `toBeInstanceOf( class )` | Class/interface check |
| `toBeSameInstanceAs( obj )` | Object **identity** check — 7.1+ |
| `toHaveKey( key )` | Struct key existence |
| `toHaveDeepKey( key )` | Deep nested struct key |
| `toInclude( needle )` | String/array inclusion |
| `toIncludeWithCase( needle )` | Case-sensitive inclusion |
| `toIncludeAll( [ needles ] )` | Every needle present — 7.1+ |
| `toIncludeAny( [ needles ] )` | At least one needle present — 7.1+ |
| `toIncludeNone( [ needles ] )` | No needle present — 7.1+ |
| `toMatch( regex )` | Regex match (no case) |
| `toMatchWithCase( regex )` | Regex match (case-sensitive) |
| `toBeGT( n )` | Greater than |
| `toBeGTE( n )` | Greater than or equal |
| `toBeLT( n )` | Less than |
| `toBeLTE( n )` | Less than or equal |
| `toBeBetween( min, max )` | Numeric/date range |
| `toBeCloseTo( expected, delta )` | Approximate numeric/date equality |
| `toSatisfy( closure )` | Arbitrary predicate |
| `toThrow( [type], [regex] )` | Exception assertion on a closure |
| `toThrowMatching( predicate )` | Exception assertion via a closure over the exception — 7.1+ |
| All of the above prefixed with `not` | Negated version of that matcher |

### BoxLang-Only Matchers (7.1+)

| Family | Matchers |
|---|---|
| Set | `toBeASet`, `toEqualSet`, `toBeSubsetOf`, `toBeSupersetOf`, `toBeDisjointFrom`, `toHaveUnion`, `toHaveIntersection`, `toHaveDifference`, `toHaveSymmetricDifference` |
| Range | `toBeRange`, `toContainValue`, `toContainRange`, `toBeInRange`, `toBeBeforeRange`, `toBeAfterRange`, `toBeBounded`, `toBeUnbounded`, `toBeHalfBounded`, `toBeIterable`, `toBeAscending`, `toBeDescending`, `toHaveStep`, `toClampTo` |
| Data Navigator | `toHavePath`, `toHavePathValue`, `toHavePathType`, `toHavePathSatisfying`, `path`, `queryPath` |

### Expectation Starters

| Starter | Purpose |
|---|---|
| `expect( actual )` | Single-value expectation |
| `expect( actual ).withContext( label )` | Label failure messages — 7.1+ |
| `expectAll( collection )` | Every element must pass |
| `expectAny( collection )` | At least one element must pass — 7.1+ |
| `expectSome( collection, min, max )` | A bounded count must pass — 7.1+ |
| `expectNone( collection )` | Zero elements may pass — 7.1+ |
