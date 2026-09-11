---
name: testbox-bdd
description: "Use this skill when writing BDD-style tests with TestBox using describe/it blocks, feature/story/scenario/given/when/then Gherkin-style suites, lifecycle hooks (beforeAll/afterAll/beforeEach/afterEach/aroundEach), focused specs (fit/fdescribe), skipping specs (xit/xdescribe/skip()) or whole test classes with the class-level skip annotation, engine detection helpers (isBoxLang/isLucee/isAdobe), grouped assertions with assertAll(), collection expectations (expectAll/expectAny/expectSome/expectNone), spec data binding, asyncAll parallel specs, nested suite trees, labels, or organizing tests around behavior descriptions."
applyTo: "**/tests/**/*.{bx,bxm,cfc,cfm,cfml}"
---

# BDD Testing with TestBox

## When to Use This Skill

- Writing BDD-style test bundles for any ColdBox/BoxLang application
- Using `describe()`, `it()`, or Gherkin aliases (`feature/story/scenario/given/when/then`)
- Setting up lifecycle methods: `beforeAll`, `afterAll`, `beforeEach`, `afterEach`, `aroundEach`
- Focusing or skipping suites, specs, or an entire test class
- Passing data into specs via data binding
- Grouping related assertions with `assertAll()` so every failure is reported at once
- Running specs in parallel with `asyncAll`

---

## Language Reference

| Concept | BoxLang (`.bx`) preferred | CFML (`.cfc`) compatible |
|---|---|---|
| Class declaration | `class extends="testbox.system.BaseSpec" {}` | `component extends="testbox.system.BaseSpec" {}` |
| Arrow closures | `() => {}` | `function() {}` |
| Lambda with args | `( currentSpec, data ) => {}` | `function( currentSpec, data ) {}` |

---

## Canonical Bundle Structure

```boxlang
class extends="testbox.system.BaseSpec" {

    // Global lifecycle — runs ONCE for the entire bundle
    function beforeAll() {
        variables.sut = getInstance( "models.UserService" )
    }

    function afterAll() {
        // global teardown
    }

    function run( testResults, testBox ) {

        describe( "UserService", () => {

            // Suite lifecycle — runs around each it()
            beforeEach( ( currentSpec ) => {
                variables.mockRepo = createMock( "models.UserRepository" )
                variables.sut.$property( propertyName = "userRepository", mock = mockRepo )
            } )

            afterEach( ( currentSpec ) => {
                // per-spec teardown
            } )

            it( "creates a user and returns the saved entity", () => {
                mockRepo.$( "save" ).$results( { id: 1, name: "Alice" } )
                var result = sut.create( { name: "Alice", email: "alice@example.com" } )
                expect( result ).toBeStruct()
                expect( result.id ).toBeNumeric()
                expect( result.name ).toBe( "Alice" )
            } )

            it( "throws ValidationException for missing email", () => {
                expect( () => {
                    sut.create( { name: "Alice" } )
                } ).toThrow( type: "ValidationException" )
            } )

        } )

    }

}
```

---

## Lifecycle Methods

### Global (`beforeAll` / `afterAll`)

Run **once** for the entire bundle — use for expensive one-time setup (boot DI container, seed DB, create JWT keys, etc.).

```boxlang
function beforeAll() {
    ORMSessionClear()
    structClear( request )
    variables.jwt = getInstance( "JWTService@cbsecurity" )
    variables.jwt.getSettings().jwt.tokenStorage.driver = "cachebox"
    variables.securityService = getInstance( "SecurityService" )
}

function afterAll() {
    variables.securityService.logout()
    directoryDelete( "/tests/tmp", true )
}
```

### Suite (`beforeEach` / `afterEach`)

Run around **every** `it()` in the containing `describe()`. In nested suites, TestBox walks **down** the tree for `beforeEach` and **up** for `afterEach`.

```boxlang
describe( "Outer", () => {

    beforeEach( ( currentSpec, data ) => {
        // runs first for all nested specs
    } )

    describe( "Inner", () => {

        beforeEach( ( currentSpec, data ) => {
            // runs second (after outer beforeEach)
        } )

        afterEach( ( currentSpec, data ) => {
            // runs first on the way out (before outer afterEach)
        } )

    } )

    afterEach( ( currentSpec, data ) => {
        // runs last for all nested specs
    } )

} )
```

### `aroundEach`

Wraps each spec — useful for transaction rollbacks or resource acquisition/release patterns.

```boxlang
aroundEach( ( spec, suite ) => {
    transaction {
        spec.body()   // run the spec inside the transaction
        transactionRollback()
    }
} )
```

---

## Suite Aliases (Gherkin)

`describe()` is aliased as: `story()`, `feature()`, `scenario()`, `given()`, `when()`.
`it()` is aliased as `then()` (but `then` uses the argument name `then` instead of `title`).

```boxlang
feature( "Box Volume Calculator", () => {

    scenario( "Calculate box volume", () => {

        given( "a width of 20, height of 30, depth of 40", () => {

            when( "I run the calculation", () => {

                then( "the result is 24000", () => {
                    expect( boxCalc.volume( 20, 30, 40 ) ).toBe( 24000 )
                } )

            } )

        } )

    } )

} )
```

```boxlang
story( "As an author, I want to list all active users", () => {

    given( "no options", () => {
        then( "it returns all active system users", () => {
            var event = this.get( "/cbapi/v1/authors" )
            expect( event.getResponse() ).toHaveStatus( 200 )
            expect( event.getResponse().getData() ).toBeArray().notToBeEmpty()
        } )
    } )

    given( "isActive = false", () => {
        then( "it returns inactive users", () => {
            var event = this.get( "/cbapi/v1/authors?isActive=false" )
            expect( event.getResponse() ).toHaveStatus( 200 )
            expect( event.getResponse().getData() ).toBeArray().notToBeEmpty()
        } )
    } )

} )
```

---

## Focused Specs and Suites

Prefix with `f` to run **only** those suites/specs. All others are skipped.

```boxlang
fdescribe( "I only run", () => {
    fit( "I am the only spec that runs", () => {
        expect( true ).toBeTrue()
    } )
    it( "I am skipped because parent fdescribe runs all children", () => {
        // still runs — parent focus runs all children
    } )
} )

describe( "Normal suite", () => {
    fit( "focused individual spec", () => {
        expect( 1 ).toBe( 1 )
    } )
    it( "this one is skipped", () => { } )
} )
```

> Focused prefixes work on all suite aliases: `fdescribe`, `fstory`, `ffeature`, `fscenario`, `fgiven`, `fwhen`, `fit`, `fthen`.

---

## Skipping Specs and Suites

Prefix with `x` or use the `skip` argument or call `skip()` inline.

```boxlang
xdescribe( "disabled suite", () => {
    it( "never runs", () => { } )
} )

xit( "disabled spec", () => { } )

it( "engine-conditional skip", () => {
    if ( !server.keyExists( "lucee" ) ) {
        skip( "Only runs on Lucee" )
    }
    expect( luceeSpecificBehavior() ).toBeTrue()
} )

it( title: "conditional skip closure", body: () => {
    expect( 1 ).toBe( 1 )
}, skip: () => {
    return !structKeyExists( server, "railo" )
} )
```

### Skipping an Entire Test Class

*TestBox 7.1+.* A `skip` annotation on the **class declaration** skips the whole bundle: no spec
runs, `beforeAll`/`afterAll` do not fire, and the bundle is left out of the dry-run discovery
tree entirely.

```boxlang
// tests/specs/LegacyImportSpec.bx — BoxLang
class extends="testbox.system.BaseSpec" skip="true" {

    function run() {
        describe( "Legacy importer", () => {
            it( "never runs while the class is skipped", () => {} )
        } )
    }

}
```

```cfml
// tests/specs/LegacyImportSpec.cfc — CFML
component extends="testbox.system.BaseSpec" skip="true" {

    function run(){
        describe( "Legacy importer", function(){
            it( "never runs while the class is skipped", function(){} )
        } )
    }

}
```

The annotation also accepts a **method name on the class**, which TestBox invokes to decide —
this is how you skip a whole bundle per engine or per feature flag, with the reason stated in
the docblock:

```boxlang
/**
 * Skipped on any engine that is not BoxLang: this bundle exercises the
 * BoxLang-only Set and Range matchers.
 */
class extends="testbox.system.BaseSpec" skip="isBoxLangMissing" {

    function isBoxLangMissing() {
        return !isBoxLang()
    }

    function run() {
        describe( "Set matchers", () => {
            it( "only runs on BoxLang", () => {
                expect( setOf( 1, 2 ) ).toBeASet()
            } )
        } )
    }

}
```

| `skip` value | Effect |
|---|---|
| `skip="true"` (or a boolean expression) | Always skip the bundle |
| `skip=""` (present, empty) | Always skip the bundle |
| `skip="methodName"` | Call `methodName()` on the class; skip when it returns true |
| annotation absent | Run normally |

> Prefer the class-level `skip` over `xdescribe`-ing every suite in the file: it is one line, it
> reads at the top of the class, and the bundle disappears from dry-run discovery rather than
> reporting a tree of skipped specs.

### Engine-Conditional Skipping

`BaseSpec` gives you engine and OS predicates for skip decisions:

| Helper | True when |
|---|---|
| `isBoxLang()` | Running on BoxLang |
| `isLucee()` | Running on Lucee — and **not** BoxLang |
| `isAdobe()` | Running on Adobe ColdFusion |
| `isWindows()` / `isLinux()` / `isMac()` | Host OS |

```boxlang
it( "uses a Lucee-only function", () => {
    if ( !isLucee() ) {
        skip( "Lucee only" )
    }
    expect( luceeSpecificBehavior() ).toBeTrue()
} )
```

> **TestBox 7.1 fix:** `isLucee()` used to return `true` under BoxLang, because BoxLang's CFML
> compatibility layer also populates `server.lucee`. It now returns `false` on BoxLang. If you
> hand-rolled `structKeyExists( server, "lucee" )` as a Lucee guard, replace it with `isLucee()`
> — the raw check still misfires on BoxLang.

BoxLang-only language surface (the `..` range operator, `setOf()`, data navigators) cannot even
be *parsed* by Lucee or Adobe, so a runtime skip is too late. Put those specs in a **`.bx` file**
instead: TestBox's bundle discovery skips `.bx` bundles outright on non-BoxLang engines, so the
file is never compiled there.

---

## Spec Data Binding

Pass a `data` struct into `it()` to avoid closure-capture bugs in loops.

```boxlang
var filePaths = directoryList( "/tests/fixtures", false, "path", "*.json" )

for ( var filePath in filePaths ) {
    it(
        title: "#getFileFromPath( filePath )# is valid JSON",
        data: { filePath: filePath },
        body: ( data ) => {
            var json = fileRead( data.filePath )
            expect( json ).notToBeEmpty()
            expect( isJSON( json ) ).toBeTrue( "File is not valid JSON: #data.filePath#" )
        }
    )
}
```

---

## Nested and Labeled Suites

```boxlang
describe( "Payment processing", { labels: "integration,payments" }, () => {

    describe( "Credit cards", { labels: "credit" }, () => {
        it( "charges a card successfully", { labels: "happy-path" }, () => {
            var result = paymentService.charge( fixtures.validCard, 99.99 )
            expect( result.success ).toBeTrue()
            expect( result.transactionId ).notToBeEmpty()
        } )

        it( "declines an expired card", () => {
            expect( () => {
                paymentService.charge( fixtures.expiredCard, 99.99 )
            } ).toThrow( type: "PaymentDeclinedException" )
        } )
    } )

    describe( "Refunds", () => {
        it( "processes a full refund", () => {
            var result = paymentService.refund( fixtures.completedTransaction )
            expect( result.refunded ).toBeTrue()
        } )
    } )

} )
```

Run with labels: `box testbox run labels=credit excludes=slow`

---

## Parallel (asyncAll) Suites

```boxlang
describe( "Parallel cache reads", { asyncAll: true }, () => {

    it( "reads product 1", () => {
        var p = productService.getById( 1 )
        expect( p.getId() ).toBe( 1 )
    } )

    it( "reads product 2", () => {
        var p = productService.getById( 2 )
        expect( p.getId() ).toBe( 2 )
    } )

} )
```

**Rules for `asyncAll`:**
- All spec variables **must** be locally scoped (`var`) or in `variables` with no inter-spec mutation
- Add CFThread `name` attributes or unique keys to avoid lock collisions
- Reserve for IO-bound or concurrency-sensitive behavior; do not over-parallelize

---

## Grouped Assertions in a Spec — `assertAll()`

*TestBox 7.1+.* A spec normally stops at the first failed assertion. When several assertions
describe **one** outcome, `assertAll()` runs them all and reports every failure together:

```boxlang
it( "returns a fully-populated user response", () => {
    var res = api.get( "/users/1" )

    assertAll( [
        () => expect( res.status ).toBe( 200 ),
        () => expect( res.body ).toHaveKey( "id" ),
        () => expect( res.body ).toHaveKey( "name" ),
        () => expect( res.body ).notToHaveKey( "passwordHash" )
    ], "GET /users/1" )
} )
```

Keep separate `it()` blocks for genuinely separate behaviours — `assertAll()` is for one
behaviour with several facets, not a way to cram a suite into a single spec. See the
[`testbox-assertions`](../assertions/SKILL.md) skill for the full contract.

---

## Key Functions Quick Reference

| Function | Alias(es) | Description |
|---|---|---|
| `describe( title, body, [labels], [asyncAll], [skip] )` | `story`, `feature`, `scenario`, `given`, `when` | Define a test suite |
| `it( title, body, [labels], [skip], [data] )` | `then` | Define a spec / test case |
| `beforeAll( body )` | — | Run once before all specs in the bundle |
| `afterAll( body )` | — | Run once after all specs in the bundle |
| `beforeEach( body, [data] )` | — | Run before each spec in the suite |
| `afterEach( body, [data] )` | — | Run after each spec in the suite |
| `aroundEach( body )` | — | Wrap each spec (transaction rollback pattern) |
| `expect( actual )` | — | Start a fluent expectation chain |
| `expectAll( collection )` | — | Assert every element in an array/struct |
| `expectAny( collection )` | — | Assert at least one element passes (7.1+) |
| `expectSome( collection, min, max )` | — | Assert a bounded count passes (7.1+) |
| `expectNone( collection )` | — | Assert zero elements pass (7.1+) |
| `assertAll( executables, [heading] )` | `$assert.all()` | Run every assertion closure, report all failures at once (7.1+) |
| `skip( [message], [detail] )` | — | Skip the current spec or suite inline |
| `addMatchers( matchers )` | — | Register custom matchers |
| `getInstance( name )` | — | Shortcut for WireBox `getInstance()` |
| `debug( var, [label] )` | — | Output a variable to the debug panel |

---

## CommandBox Scaffolding

```bash
# Create a new BDD spec
coldbox create bdd name=UserServiceSpec open=true

# Create with handler integration
coldbox create integration-test handler=users
```
