---
name: rulebox
description: >
  Use this skill when implementing a business rules engine in ColdBox/BoxLang using rulebox. Covers
  RuleEngine injection, defining rules with when/then/otherwise closures, named rule sets,
  chaining rules, running a rule set against a context, and production patterns for policy
  evaluation and workflow branching.
applyTo: "**/*.{bx,cfc,cfm,bxm}"
---

# Rulebox Skill

## When to Use This Skill

Load this skill when:
- Encapsulating business rules outside of handler/service logic
- Evaluating complex conditional logic (eligibility, pricing, access policies) against a context
- Composing reusable if/then rule chains that can be tested independently
- Replacing large `if/elseif` chains with named, testable rule objects

## Installation

```bash
box install rulebox
```

## Core API

### Injection

```js
property name="ruleEngine" inject="RuleEngine@rulebox";
```

### Define a Rule

```js
var rule = ruleEngine
    .newRule()
    .when( function( context ) {
        return context.age >= 18 && context.hasValidId
    } )
    .then( function( context ) {
        context.approved = true
        context.tier     = "standard"
    } )
    .otherwise( function( context ) {
        context.approved = false
        context.reason   = "Age restriction or missing ID"
    } )
```

### Run a Rule

```js
var context = {
    age        : rc.age,
    hasValidId : rc.hasValidId,
    approved   : false
}

rule.run( context )

if ( context.approved ) {
    // proceed
}
```

### Rule Sets (Named Collections)

```js
// Define a named rule set
ruleEngine.define( "eligibility", [

    ruleEngine.newRule()
        .when( function( c ) { return c.accountAge < 30 } )
        .then( function( c ) { c.flags.append( "new_account" ) } ),

    ruleEngine.newRule()
        .when( function( c ) { return c.balance < 0 } )
        .then( function( c ) {
            c.flags.append( "negative_balance" )
            c.eligible = false
        } ),

    ruleEngine.newRule()
        .when( function( c ) { return c.kycPassed && !c.flags.find( "negative_balance" ) } )
        .then( function( c ) { c.eligible = true } )

] )

// Run the rule set
var context = {
    accountAge : 15,
    balance    : 500,
    kycPassed  : true,
    eligible   : false,
    flags      : []
}

ruleEngine.run( "eligibility", context )
// context.eligible is now true or false
```

## Production Patterns

### Loan Application Eligibility

```js
class LoanEligibilityService {

    property name="ruleEngine" inject="RuleEngine@rulebox";

    LoanEligibilityService function init() {
        ruleEngine.define( "loan_eligibility", [

            // Must be of legal age
            ruleEngine.newRule()
                .when( function( c ) { return c.age < 18 } )
                .then( function( c ) {
                    c.approved = false
                    c.reasons.append( "Applicant must be 18 or older" )
                } ),

            // Credit score check
            ruleEngine.newRule()
                .when( function( c ) { return c.creditScore < 620 } )
                .then( function( c ) {
                    c.approved = false
                    c.reasons.append( "Credit score below minimum threshold" )
                } ),

            // Income check
            ruleEngine.newRule()
                .when( function( c ) { return c.annualIncome < c.requestedAmount * 3 } )
                .then( function( c ) {
                    c.approved = false
                    c.reasons.append( "Income insufficient for requested amount" )
                } ),

            // Approve if no disqualifiers
            ruleEngine.newRule()
                .when( function( c ) { return c.reasons.len() == 0 } )
                .then( function( c ) { c.approved = true } )

        ] )
        return this
    }

    struct function evaluate( application ) {
        var context = {
            age             : application.age,
            creditScore     : application.creditScore,
            annualIncome    : application.annualIncome,
            requestedAmount : application.amount,
            approved        : false,
            reasons         : []
        }

        ruleEngine.run( "loan_eligibility", context )

        return {
            approved : context.approved,
            reasons  : context.reasons
        }
    }
}
```

### Pricing Rule Engine

```js
ruleEngine.define( "pricing", [

    ruleEngine.newRule()
        .when( function( c ) { return c.isVip } )
        .then( function( c ) { c.discount += 0.20 } ),

    ruleEngine.newRule()
        .when( function( c ) { return c.orderTotal >= 500 } )
        .then( function( c ) { c.discount += 0.10 } ),

    ruleEngine.newRule()
        .when( function( c ) { return c.couponCode == "SAVE15" } )
        .then( function( c ) { c.discount += 0.15 } ),

    ruleEngine.newRule()
        .when( function( c ) { return c.discount > 0.35 } )
        .then( function( c ) { c.discount = 0.35 } )   // max 35% discount

] )

var context = {
    isVip      : userService.isVip( userId ),
    orderTotal : orderTotal,
    couponCode : rc.couponCode ?: "",
    discount   : 0.0
}

ruleEngine.run( "pricing", context )
var finalPrice = orderTotal * ( 1 - context.discount )
```

## Best Practices

- **Use named rule sets** for collections of related rules — enables `run("name", context)` calls
- **Keep individual rules focused** — one condition, one consequence
- **Treat context as a mutable struct** — rules communicate through shared context state
- **Test rules independently** — call `.run(context)` directly in TestBox specs
- **Avoid side effects with external I/O** in rule closures — rules should only modify context
- **Order rules carefully** — later rules can read decisions made by earlier rules in the set
- **Use `otherwise()`** for explicit fallback behavior rather than relying on default context state

## Documentation

- rulebox: https://github.com/coldbox-modules/rulebox
