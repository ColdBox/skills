---
name: rulebox
description: >
  Use this skill when writing business rules with RuleBox, the natural-language rules engine for
  BoxLang and ColdBox. Covers RuleBook classes and defineRules(), the given/when/except/then/using/
  withPriority/stop/active/withDescription DSL, facts and the Result object, declaring and enforcing
  the facts a RuleBook takes (defineFacts, fact(), withFacts, enforceFacts, strictFacts,
  validateFacts), describing rulebooks and rules, the Builder, declared rulebooks
  (ruleBook( "name" ), inject="rulebook:name"), loading rules from JSON, YAML or a database with the
  condition grammar and registered actions/predicates, the audit trail, dryRun(), rule metrics,
  error handling, thread safety, the Rule Visualizer, and testing rulebooks with TestBox.
applyTo: "**/*.{bx,bxm,bxs,cfc,cfm,json,yaml,yml}"
---

# RuleBox Skill

RuleBox turns tangled `if`/`else` blocks into named rules you can read, test and audit. Given some
facts, when a condition holds, then act. Rules are written in BoxLang with a Given-When-Then DSL, or
as JSON, YAML or database rows that a team can change without a deploy. It is a BoxLang port of the
Java [RuleBook](https://github.com/rulebook-rules/rulebook) project.

## When to Use This Skill

Load this skill when:

- Replacing long `if`/`elseif` chains (eligibility, pricing, rates, discounts, routing, fraud flags) with named, testable rules
- Writing a class that extends `rulebox.models.RuleBook`, or calling `newRule()`, `addRule()`, `given()`, `run()` or `getResult()`
- Declaring which facts a RuleBook takes (types, required, defaults), enforcing them, or describing rulebooks and rules
- Loading rules from JSON, YAML or a database table with `loadRules()`, or declaring rulebooks under `moduleSettings.rulebox`
- Asking which rules fired, previewing rules with `dryRun()`, or reading rule metrics
- Enabling or securing the Rule Visualizer at `/rulebox-visualizer`
- Testing rulebooks with TestBox

## Requirements and Installation

- BoxLang 1.18+ and ColdBox 8+

```bash
box install rulebox
```

Optional BoxLang modules, only if you use the feature (RuleBox does not install them for you):

| Feature | Install |
|---|---|
| YAML rule files (`YAMLRuleSource`, `.yaml`/`.yml` files in the convention folder) | `box install bx-yaml` |
| Visualizer metrics that survive a restart (`SQLiteMetricsStore`) | `box install bx-sqlite`, plus a datasource |

### What gets registered in WireBox

| WireBox ID | Scope | Description |
|---|---|---|
| `RuleBook@rulebox` | Transient | A rule book that groups and chains rules |
| `Rule@rulebox` | Transient | A single rule |
| `Result@rulebox` | Transient | The result produced by a rule chain |
| `Builder@rulebox` | Singleton | Builds rules and rule books on the fly |
| `RuleBookRegistry@rulebox` | Singleton | Hands out rulebooks declared in settings or config files |

The module also adds a `ruleBook( name )` helper to handlers, views and layouts, and a
`rulebook` injection DSL (`inject="rulebook:name"`).

## Your First RuleBook

A RuleBook is a class that extends `rulebox.models.RuleBook` and adds rules in `defineRules()`.

```js
// models/HelloWorld.bx
class extends="rulebox.models.RuleBook"{

	function defineRules(){
		addRule(
			newRule( "sayHello" )
				.then( ( facts, result ) => result.setValue( "Hello " & facts.name ) )
		)
	}

}
```

```js
var greeting = getInstance( "HelloWorld" )
	.run( { name : "World" } )
	.getResult()
	.getValue()
// "Hello World"
```

- `run( facts )` returns the RuleBook itself (for chaining), not the answer. Read the answer with `.getResult().getValue()`.
- `defineRules()` runs on the first `run()` or `dryRun()`, not at creation, so WireBox injections are available inside it. Later runs on the same instance reuse the rules.
- `newRule( name )` takes an optional name. Rule names must be unique within a RuleBook: a duplicate throws `RuleBox.DuplicateRuleNameException`. Name every rule, because the audit trail and metrics are keyed by name.
- `addRule()` also accepts a closure that configures the rule: `addRule( ( rule ) => rule.setName( "x" ).then( ... ) )`.
- A RuleBook has a name too (`setName( name )`), used by auditing and the Visualizer.
- The class docblock becomes the rulebook's description, and a RuleBook can declare the facts it takes in `defineFacts()`. See [Declaring Facts](#declaring-facts) and [Descriptions](#descriptions).

## The DSL

Given-When-Then, plus `except()`:

| Method | Purpose |
|---|---|
| `given( name, value )` / `givenAll( struct, overwrite=true )` | Supply facts (on a RuleBook these are usually passed to `run()` instead) |
| `when( ( facts ) => boolean )` | The condition. One per rule; must return a boolean. Omitted means always true |
| `except( ( facts ) => boolean )` | Cancels the rule when it returns `true`, even if `when()` passed |
| `then( ( facts, result ) => ... )` | An action. A rule can have several, run in order. Returning `true` breaks that rule's `then()` chain; `void`/`false` continues |
| `using( "fact1,fact2" )` | Restricts the facts passed to the next `then()`. Takes a list or an array; chained `using()` calls add up |
| `withPriority( n )` | Higher runs earlier. Default `0`; ties keep insertion order. Can be set before or after `addRule()` |
| `stop()` | After this rule fires, no further rules run |
| `active( from, until )` | Only evaluate the rule inside a date window; either bound can be omitted. Outside it the rule is `SKIPPED` |
| `withDescription( text )` | Says what the rule does, for the Visualizer and `dryRun()`. No effect on how it runs |

```js
class extends="rulebox.models.RuleBook"{

	function defineRules(){
		addRule(
			newRule( "checkBlocklist" )
				.withPriority( 10 )
				.when( ( facts ) => facts.applicant.isBlocklisted() )
				.then( ( facts, result ) => result.setValue( 0 ) )
				.stop()
		)
		addRule(
			newRule( "dispenseCash" )
				.when( ( facts ) => facts.balance > 100 )
				.except( ( facts ) => facts.accountDisabled )
				.then( ( facts, result ) => result.setValue( facts.balance - 100 ) )
		)
		addRule(
			newRule( "greetBoth" )
				.when( ( facts ) => facts.keyExists( "hello" ) && facts.keyExists( "world" ) )
				.using( "hello" )
				.then( ( facts ) => println( facts.hello ) )
				.using( "world" )
				.then( ( facts ) => println( facts.world ) )
		)
		addRule(
			newRule( "blackFridayPromo" )
				.active( from : "2026-11-27", until : "2026-12-01" )
				.then( ( facts, result ) => result.setValue( result.getValue() * 0.8 ) )
		)
	}

}
```

## Facts and Results

Facts live in a struct and are passed by reference to every rule. Key them off simple values or
domain objects, whichever reads better:

```js
.when( ( facts ) => facts.creditScore < 600 )
.when( ( facts ) => facts.applicant.getCreditScore() < 600 )
```

- `run( facts )` is a shortcut for `givenAll( facts )` then `run()`. Passed facts replace existing ones of the same name; `run( facts, false )` keeps the ones already set.
- `withDefaultResult( value )` seeds the result; otherwise it starts as `null`. Struct, array and query defaults are deep-copied, so rules never mutate the default.
- The **same** `Result` instance flows through every `then()`, so rules accumulate onto it, much like `reduce`.
- `run()` resets the result to its default at the start of every run.

| `Result` method | Description |
|---|---|
| `setValue( value )` / `getValue()` | Set or read the value |
| `isPresent()` | `true` if a value is set or defaulted (falsy values such as `0`, `""` and `false` count) |
| `ifPresent( ( value ) => ... )` | Call the closure only if the value is not `null` |
| `orElse( other )` | The value, or `other` when absent |
| `orElseGet( () => ... )` | The value, or the closure's return when absent |
| `reset()` | Back to the default value |

### A complete example

```js
// models/HomeLoanRateRuleBook.bx
class extends="rulebox.models.RuleBook"{

	function defineRules(){
		// A credit score under 600 pays 4x the rate, and nothing else applies
		addRule(
			newRule( "lowCredit" )
				.when( ( facts ) => facts.applicant.getCreditScore() < 600 )
				.then( ( facts, result ) => result.setValue( result.getValue() * 4 ) )
				.stop()
		)
		// 600 to 699 pays one extra point
		addRule(
			newRule( "fairCredit" )
				.when( ( facts ) => facts.applicant.getCreditScore() < 700 )
				.then( ( facts, result ) => result.setValue( result.getValue() + 1 ) )
		)
		// 700+ with $25,000 cash on hand gets a quarter point off
		addRule(
			newRule( "goodCreditWithCash" )
				.when( ( facts ) => facts.applicant.getCreditScore() >= 700 && facts.applicant.getCashOnHand() >= 25000 )
				.then( ( facts, result ) => result.setValue( result.getValue() - 0.25 ) )
		)
		// First-time buyers get 20% off the adjusted rate
		addRule(
			newRule( "firstTimeBuyer" )
				.when( ( facts ) => facts.applicant.getFirstTimeHomeBuyer() )
				.then( ( facts, result ) => result.setValue( result.getValue() * 0.80 ) )
		)
	}

}
```

```js
// handlers/Loans.bx
class{

	function rate( event, rc, prc ){
		return getInstance( "HomeLoanRateRuleBook" )
			.withDefaultResult( 4.5 )
			.run( { applicant : new models.Applicant( 650, 20000, true ) } )
			.getResult()
			.getValue()
		// 4.4: 4.5 + 1 = 5.5, then 20% off
	}

}
```

## Declaring Facts

> Unreleased: on RuleBox's `development` branch, coming in the release after 2.0.0. Check the
> installed version before using it.

A RuleBook can declare the facts it takes. The declarations document the rulebook and drive the
Visualizer's dry run form; they are only checked when the rulebook **enforces** them.

```js
class extends="rulebox.models.RuleBook"{

	function defineFacts(){
		enforceFacts()   // optional: check facts on every run(); strictFacts() also rejects undeclared ones
		fact( "creditScore" ).type( "numeric" ).required().description( "300 to 850" ).example( 680 )
		fact( "cashOnHand" ).type( "numeric" ).defaultValue( 0 )
		fact( "loanType" ).type( "string" ).values( [ "fixed", "variable" ] ).defaultValue( "fixed" )
	}

	function defineRules(){ /* ... */ }

}
```

- `fact( name )` builder: `type()`, `required( value = true )`, `defaultValue( value )` (not `default()`: reserved word), `description()`, `example()`, `values( array )` (simple values compare without regard to case). Calling `fact()` again with the same name returns the existing declaration, so an instance can refine a class's facts.
- Types: `any` (default), `string` (any simple value), `numeric`, `integer`, `boolean`, `date`, `struct`, `array`, `object`. An unknown type or blank name throws `RuleBox.InvalidFactDefinitionException`.
- `withFacts( { creditScore : { type : "numeric", required : true, default : 0, description : "", example : 680, values : [] } } )` declares from data, for Builder rulebooks, rule files and config. It validates every declaration first and declares nothing if one is invalid.
- `defineFacts()` runs once per instance, the first time facts are needed.
- `enforceFacts( strict = false )` / `strictFacts()`. Called in `defineFacts()` they apply to every instance; called on an instance they win over the class. When enforcing, `run()` and `dryRun()`:
  1. fill missing (or `null`) facts that have a default (`run()` keeps them in the rulebook's facts; `dryRun()` checks a copy);
  2. check required, type and `values`, plus undeclared facts when strict;
  3. throw `RuleBox.InvalidFactsException` before any rule runs.
- The exception's `message` lists every problem; `extendedInfo` holds them as a JSON array of `{ fact, problem, message }`, where `problem` is `missing`, `type`, `value` or `undeclared`. Messages never include fact values.
- `validateFacts( facts )` returns the same problems without running or throwing, whether or not the rulebook enforces (use it to validate a form or API request first).
- `getFactDefinitions()` returns `[ { name, type, required, description, default?, example?, values? } ]` in declaration order. `getFactsEnforced()` / `getFactsStrict()` report the mode.

```js
try{
	ruleBook.run( facts )
} catch( "RuleBox.InvalidFactsException" e ){
	var problems = jsonDeserialize( e.extendedInfo )   // [ { fact, problem, message } ]
}
```

## Descriptions

Rulebooks and rules can say what they do. Descriptions are shown in the Visualizer and have no
effect on how anything runs.

```js
/**
 * Decides a loan application from the applicant's credit score.
 */
class extends="rulebox.models.RuleBook"{

	function defineRules(){
		addRule( newRule( "autoApprove" ).withDescription( "Approves a score of 680 or more" ).when( ... ) )
	}

}
```

- `RuleBook.getDescription()` returns the first of: a description set with `withDescription( text )`, a rule file's or config's `description`; the class's `@description( "..." )` or `@hint( "..." )` annotation; the class docblock (line breaks folded into spaces). A plain `RuleBook` (Builder or registry) never picks up RuleBook's own docblock, so it is `""` until set.
- `Rule.withDescription( text )` / `getDescription()`; a rule definition takes a `description` key.
- `dryRun()` reports each rule's `description`.

## The Builder

`Builder@rulebox` (a thread-safe singleton) builds rules and rulebooks at runtime, with no class.
Use it when rules are assembled from user choices, settings or data; use a RuleBook class when the
rules are fixed, so they can be named, reused and tested.

```js
class singleton{

	@inject( "Builder@rulebox" )
	property name="builder";

	function greet( name ){
		return variables.builder.rulebook( "Greeter" )
			.addRule( variables.builder.rule( "hello" ).then( ( facts, result ) => result.setValue( "Hello " & facts.name ) ) )
			.addRule( variables.builder.rule( "shout" ).then( ( facts, result ) => result.setValue( result.getValue() & "!" ) ) )
			.run( { name : arguments.name } )
			.getResult()
			.getValue()
	}

}
```

`builder.rulebook( name="" )` returns a normal `RuleBook`, so every RuleBook method works on it,
including `loadRules()`. `builder.rule( name )` returns a standalone `Rule`.

## Thread Safety

`RuleBook` and `Rule` are **transients**: `given()` and `run( facts )` write facts, the result and
the audit trail onto the instance. A RuleBook instance is not safe to share between requests or
threads.

- Call `getInstance( "MyRuleBook" )` where you use it, every time.
- Never keep a RuleBook in a singleton property or a shared scope, and never `inject="MyRuleBook"` into a singleton (it would keep one instance for its whole life).
- For declared rulebooks, `ruleBook( "name" )` and `RuleBookRegistry.getRuleBook( "name" )` return a fresh instance on every call.
- `inject="rulebook:name"` injects a small provider, not a RuleBook. Call `.get()` at the point of use, so it is safe even in a singleton:

```js
class singleton{

	@inject( "rulebook:credit" )
	property name="creditRules";

	function decide( score ){
		return variables.creditRules.get()
			.run( { creditScore : arguments.score } )
			.getResult()
			.getValue()
	}

}
```

## Rules as Data: JSON, YAML and Databases

`RuleBook.loadRules( source )` turns external rule definitions into real `Rule`s, with the same
priority chain, audit trail and `dryRun()` support. A definition can never carry code, so a file or
table from an untrusted source is safe to load.

### The rule-definition schema

| Key | Meaning |
|---|---|
| `name` | The rule's name (for auditing) |
| `description` | What the rule does, like `withDescription()` |
| `priority` | Same as `withPriority()`; default `0` |
| `activeFrom` / `activeUntil` | Same as `active()`; either can be omitted |
| `when` / `except` | A condition node, or a `{ "predicate": "name", "params": {} }` reference |
| `then` | An array of `{ "action": "name", "params": {} }` references |
| `using` | An array of fact names applied to every `then` action |
| `stop` | `true` to stop the chain after this rule fires |

```json
[
	{
		"name": "highRisk",
		"priority": 10,
		"when": { "lt": [ "creditScore", 600 ] },
		"then": [ { "action": "flagHighRisk" } ],
		"stop": true
	},
	{
		"name": "approve",
		"when": { "gte": [ "creditScore", 600 ] },
		"then": [ { "action": "approveApplicant", "params": { "reason": "good credit" } } ]
	}
]
```

### The condition grammar

A safe, declarative tree with no `eval`. Each node is a struct with exactly one operator:

| Operator | Shape | Meaning |
|---|---|---|
| `eq` / `neq` | `[ "factPath", value ]` | Equals / not equals |
| `lt` / `lte` / `gt` / `gte` | `[ "factPath", value ]` | Numeric or date comparison |
| `in` | `[ "factPath", [ values ] ]` | The fact is one of the values |
| `and` / `or` | `[ node, node, ... ]` | All / any of the child nodes |
| `not` | `node` | Negates the child node |

`factPath` supports dot notation into nested facts (`"applicant.address.state"`), and a missing
path resolves to `null` instead of throwing.

```json
{
	"and": [
		{ "eq": [ "state", "CA" ] },
		{ "or": [
			{ "lt": [ "creditScore", 600 ] },
			{ "not": { "eq": [ "flagged", true ] } }
		] }
	]
}
```

- `loadRules()` validates every definition up front. A malformed node throws `RuleBox.InvalidRuleDefinitionException` naming the rule (or its 1-based position) and the path, such as `when.and[2].lt`. Nothing is added from a source that fails.
- A `{ "predicate": ... }` reference is allowed only at the top level of `when`/`except`, not nested inside `and`/`or`/`not`. Put compound logic in one registered predicate instead.

### Registered actions and predicates

Definitions reference code by name, so register it on the RuleBook **before** `loadRules()`:

```js
ruleBook
	.registerPredicate( "isEligible", ( facts, params ) => facts.creditScore >= params.threshold )
	.registerAction( "approveApplicant", ( facts, result, params ) => result.setValue( params.reason ) )
	.registerAction( "flagHighRisk", "RiskService@myModule" )
	.loadRules( new rulebox.models.JSONRuleSource( expandPath( "/config/rules/credit.json" ) ) )
```

Each accepts a closure, an object with `execute( facts, result, params )` (actions) or
`test( facts, params )` (predicates) (`RuleAction`/`RulePredicate` document the optional contract),
or a WireBox ID string resolved immediately. A name used by a definition but never registered throws
`RuleBox.UnregisteredActionException` / `RuleBox.UnregisteredPredicateException` from `loadRules()`.

### Rule sources

```js
// JSON file: an array of definitions
ruleBook.loadRules( new rulebox.models.JSONRuleSource( "/path/to/rules.json" ) )

// YAML file: same schema (needs bx-yaml)
ruleBook.loadRules( new rulebox.models.YAMLRuleSource( "/path/to/rules.yaml" ) )

// Database: a datasource and SQL, or a query you already have
ruleBook.loadRules( new rulebox.models.DBRuleSource( datasource = "myApp", sql = "SELECT * FROM rules WHERE ruleset = 'credit'" ) )
ruleBook.loadRules( new rulebox.models.DBRuleSource( query = myQuery ) )
```

`DBRuleSource` columns: `name`, `priority`, `active_from`, `active_until`, `when_json`,
`except_json`, `then_json`, `stop`, `using_facts` (a comma-delimited list). The `*_json` columns
hold the same JSON as a JSON rule file. `stop` accepts `true/false`, `1/0`, `yes/no`, `y/n`. A bad
row throws `RuleBox.InvalidRuleRowException` naming the rule and column.

Any object with a `load()` method that returns an array of definitions is a valid source (a REST
call, a cache, a config service). There is no interface to implement.

A source (JSON, YAML or inline) may return an **envelope** instead of a bare array, to
describe the rulebook and declare its facts. `rules` is required; the rest is optional. Any other
key throws `RuleBox.InvalidRuleDefinitionException`, and the envelope is applied only after every
rule builds.

```json
{
	"description": "Approves applicants with a credit score of 600 or more",
	"facts": {
		"creditScore": { "type": "numeric", "required": true, "description": "FICO score", "example": 680 }
	},
	"enforceFacts": true,
	"strictFacts": false,
	"rules": [ { "name": "approve", "when": { "gte": [ "creditScore", 600 ] } } ]
}
```

RuleBox never watches files or tables. Call `ruleBook.reloadRules( source )` when you decide to pick
up edits. It is `clearRules()` (rules and audit trail) followed by `loadRules( source )`; registered
actions, predicates and metrics are kept.

## Declared Rulebooks

For apps with several named rulebooks, declare them in config and let `RuleBookRegistry@rulebox`
build them:

```js
// config/ColdBox.bx
moduleSettings = {
	rulebox : {
		rulebooks : {
			// A string: a rule file path (JSON or YAML by extension), relative to the app root
			"credit" : "config/rules/creditscore.yaml",
			// An array: inline rule definitions
			"promo" : [
				{ "name" : "blackFriday", "then" : [ { "action" : "applyDiscount" } ] }
			],
			// A struct: a source plus the actions/predicates it references (WireBox IDs only)
			"shipping" : {
				"source"     : "config/rules/shipping.json",
				"actions"    : { "applyDiscount" : "PromoActions@myModule" },
				"predicates" : { "isEligible" : "PromoPredicates@myModule" }
			},
			// A database source needs the struct form
			"fraud" : {
				"source" : { "type" : "db", "datasource" : "myApp", "sql" : "SELECT * FROM rules WHERE ruleset = 'fraud'" }
			},
			// Describe the rulebook and declare (and enforce) its facts in config
			"loans" : {
				"source"       : "config/rules/loans.json",
				"description"  : "Decides home loan applications",
				"facts"        : { "creditScore" : { "type" : "numeric", "required" : true } },
				"enforceFacts" : true
			}
		}
	}
}
```

- Any `*.json`, `*.yaml` or `*.yml` file in the convention folder (default `config/rulebox`, change it with `conventionPath`) is discovered automatically; the rulebook's name is the file name. Two files with the same name throw `RuleBox.DuplicateRuleBookException`.
- An explicit config entry with the same name as a discovered file layers its `actions`/`predicates` on top.
- Config can only reference WireBox IDs. To use a closure, get the rulebook and call `registerAction()` yourself.
- `description`, `facts`, `enforceFacts` and `strictFacts` work as in an envelope. A config `description` wins over the file's; config `facts` layer over the file's key by key; the flags only turn checking on.

Getting a declared rulebook (each call builds a **fresh** RuleBook):

```js
ruleBook( "credit" )                                                 // handlers, views, layouts
getInstance( "RuleBookRegistry@rulebox" ).getRuleBook( "credit" )    // anywhere
property name="creditRules" inject="rulebook:credit";                // a provider: creditRules.get()
property name="registry" inject="rulebook";                          // the registry itself
```

`getInstance( "RuleBookRegistry@rulebox" ).reload()` re-reads config and the convention folder
without a restart.

## Auditing, Dry Runs and Metrics

### Dry run

`dryRun( facts )` previews which rules *would* fire, without running any `then()`, touching the
result, the facts, the audit trail or metrics. It stops where a real run would stop.

```js
var report = getInstance( "HomeLoanRateRuleBook" ).dryRun( { applicant : applicant } )
// [ { name : "lowCredit", description : "", wouldExecute : false, wouldStop : false }, { name : "fairCredit", description : "", wouldExecute : true, wouldStop : false }, ... ]
```

`description` is the rule's description, or `""`. A rulebook that enforces its facts checks them
first (throwing `RuleBox.InvalidFactsException`).

A single `Rule` can be dry-run without a RuleBook:
`builder.rule( "lowScore" ).when( ( facts ) => facts.creditScore < 600 ).stop().dryRun( { creditScore : 550 } )`.

### The audit trail

Every rule has a state from `ruleBook.RULE_STATES`. `run()` resets all of them to `REGISTERED` first.

| State | Meaning |
|---|---|
| `REGISTERED` | Added, not evaluated this run (also every rule after a `stop()` or a failure) |
| `EXECUTED` | `when()` passed, `except()` did not, and the actions completed |
| `SKIPPED` | `when()`/`except()` did not pass, or the rule is outside its `active()` window |
| `STOPPED` | Executed, then `stop()` ended the chain |
| `FAILED` | Evaluating the rule threw |

```js
var rulebook = getInstance( "HomeLoanRateRuleBook" ).withDefaultResult( 4.5 ).run( facts )
rulebook.getRuleStatus( "fairCredit" )   // "EXECUTED"; an unknown name returns "NOT_AVAILABLE"
rulebook.getRuleStatusMap()              // { lowCredit : "SKIPPED", fairCredit : "EXECUTED", ... }
```

### Rule metrics

Each RuleBook instance also keeps running metrics per rule across every `run()` on that instance
(they are not reset by `run()`):

```js
rulebook.getRuleMetrics( "fairCredit" )
// { name, totalEvaluations, countsByState, totalDurationMs, avgDurationMs, minDurationMs, maxDurationMs,
//   completed, failed, completionRate, errorRate, lastError, errors, firstRunAt, lastRunAt, ... }
rulebook.getRuleMetricsMap()   // every rule's metrics, ready to serialize
rulebook.resetMetrics()
```

`completed` counts `EXECUTED`, `SKIPPED` and `STOPPED`; `failed` counts `FAILED`. Once a rule fails,
`lastError` holds `{ type, message, at, fingerprint }` and `errors` lists its distinct errors (each
stored once with a `count`, its cause chain, BoxLang frames and raw Java trace). For metrics across
instances and restarts, use the Visualizer's metrics store.

## Error Handling

- A throw from `when()`, `except()`, a `then()` action, or an unreadable `active()` date marks the rule `FAILED` in the audit trail and metrics, **then re-throws** to your code. Later rules stay `REGISTERED`.
- Calling `run()` on a `Rule` that was never added to a RuleBook throws `RuleBox.RuleNotAttachedException`. Build rules through a RuleBook or the Builder.

```js
var rulebook = getInstance( "PaymentRules" )
try{
	rulebook.run( facts )
} catch( any e ){
	// The audit trail already says which rule threw
	var failedRules = rulebook.getRuleStatusMap().filter( ( name, state ) => state == "FAILED" ).keyList()
	writeLog( text = "RuleBox rule [#failedRules#] failed: #e.message#", type = "error" )
	rethrow
}
```

| Exception | Thrown when |
|---|---|
| `RuleBox.DuplicateRuleNameException` | Adding a rule whose name is already used in the RuleBook |
| `RuleBox.RuleNotAttachedException` | Running a `Rule` that is not in a RuleBook |
| `RuleBox.InvalidRuleDefinitionException` | `loadRules()` finds a malformed definition or condition |
| `RuleBox.UnregisteredActionException` / `RuleBox.UnregisteredPredicateException` | A definition references an action/predicate that was never registered |
| `RuleBox.InvalidRuleRowException` | A `DBRuleSource` row has a bad column value |
| `RuleBox.DuplicateRuleBookException` | Two convention files map to the same rulebook name |
| `RuleBox.InvalidFactsException` | A rulebook that enforces its facts gets a missing, mistyped, disallowed or (strict) undeclared fact |
| `RuleBox.InvalidFactDefinitionException` | A fact declaration has an unknown type, key or a blank name |

## The Rule Visualizer

An admin UI for declared rulebooks: a dashboard with problem rules and slowest rules, the real
execution chain of each rulebook, a dry-run playground, per-rule health with errors and stack traces,
and a live tracker that streams every rule evaluation over server-sent events.

```js
moduleSettings = {
	rulebox : {
		visualizer : {
			enabled        : true,                            // default false
			metricsStore   : "InMemoryMetricsStore@rulebox",  // or "SQLiteMetricsStore@rulebox", or your IMetricsStore
			datasourceName : "rulebox_visualizer",            // only read by SQLiteMetricsStore
			maxStreams     : 25                               // open Live Tracker connections allowed at once
		}
	}
}
```

- It lives at `/rulebox-visualizer` (also `/rulebox-visualizer/chain?name=`, `/dryrun`, `/metrics`, `/live`).
- The dashboard, chain view and dry run show rulebook and rule descriptions; the chain view lists a rulebook's declared facts (with a described/enforced/strict badge); the dry run screen builds a form from them (Form/JSON toggle) and shows each problem next to its field when the rulebook enforces its facts. `GET /rulebox-visualizer/apiFacts?name=` returns `{ name, description, enforced, strict, facts }`.
- While `enabled` is `false` (the default), every route returns 404 and RuleBox records no metrics and broadcasts nothing.
- **RuleBox does not secure it.** It shows rule names, error messages and stack traces (file paths included), so protect `/rulebox-visualizer` with a cbsecurity rule or your own auth before enabling it outside development.
- `InMemoryMetricsStore` (the default) needs no setup but resets on restart. `SQLiteMetricsStore` persists to a `rulebox_events` table and needs `bx-sqlite` plus a datasource named by `datasourceName`. Its optional `retentionDays` (30), `maxStoredEvents` (100000), `circuitBreakerThreshold` (5) and `circuitBreakerCooldownSeconds` (60) settings bound its growth and failures.
- A custom store implements `IMetricsStore@rulebox` (`recordEvent`, `queryEvents`, `queryRuleBookSummary`, `queryRuleMetrics`, `queryAllRuleMetrics`, `queryRuleErrors`, `queryRuleBookNames`, `reset`).
- Settings are validated when the app starts; a bad value throws naming the key.

```js
// Application.bx: the datasource for SQLiteMetricsStore
this.datasources = {
	rulebox_visualizer : {
		driver   : "sqlite",
		database : "./.database/rulebox_visualizer.db"
	}
}
```

## Testing Rulebooks

Rulebooks are plain transients, so test them directly with TestBox. Get a fresh instance per spec,
then assert on the result and the audit trail:

```js
class extends="coldbox.system.testing.BaseTestCase" appMapping="/root"{

	function beforeEach(){
		setup()
	}

	function run(){
		describe( "Home loan rates", () => {
			it( "gives a first-time buyer with a 650 score 4.4", () => {
				var rulebook = getInstance( "HomeLoanRateRuleBook" )
					.withDefaultResult( 4.5 )
					.run( { applicant : new models.Applicant( 650, 20000, true ) } )

				expect( rulebook.getResult().getValue() ).toBe( 4.4 )
				expect( rulebook.getRuleStatus( "fairCredit" ) ).toBe( "EXECUTED" )
				expect( rulebook.getRuleStatus( "goodCreditWithCash" ) ).toBe( "SKIPPED" )
			} )

			it( "stops after the low credit rule", () => {
				var report = getInstance( "HomeLoanRateRuleBook" )
					.dryRun( { applicant : new models.Applicant( 550, 0, true ) } )

				expect( report ).toHaveLength( 1 )
				expect( report[ 1 ].wouldStop ).toBeTrue()
			} )
		} )
	}

}
```

- Test JSON/YAML rules by loading them into a Builder rulebook with the same registered actions as production, or test a declared rulebook through `getInstance( "RuleBookRegistry@rulebox" ).getRuleBook( "name" )`.
- `DBRuleSource( query = queryNew( ... ) )` tests database-driven rules without a database.
- Use `dryRun()` to assert which rules would fire without running their side effects.
- Assert fact handling with `validateFacts( facts )` (an array of `{ fact, problem, message }`) and `expect( () => ruleBook.run( badFacts ) ).toThrow( "RuleBox.InvalidFactsException" )`.

## Best Practices

- **Name every rule.** The audit trail, metrics, errors and Visualizer are keyed by name.
- **Describe rulebooks and rules, and declare the facts a rulebook takes.** It documents them for the next developer and drives the Visualizer's dry run form. Enforce facts at system boundaries (API input, forms) and use `validateFacts()` to report problems without running.
- **One concern per rule.** One condition and one outcome; compose with priority and `stop()` instead of large closures.
- **Get a fresh RuleBook per use.** Never cache one in a singleton or shared scope; use `getInstance()`, `ruleBook( "name" )` or an `inject="rulebook:name"` provider.
- **Accumulate through the `Result`**, seeded with `withDefaultResult()`, instead of mutating facts.
- **Keep actions small.** Put I/O and heavy logic in services and call them from a registered action or `then()`.
- **Use rules as data for business-owned thresholds**, and keep code-heavy logic in registered predicates and actions.
- **Prefer `dryRun()`** to preview or explain decisions without side effects.
- **Secure the Visualizer** before enabling it anywhere but development.

## Documentation

- RuleBox docs: https://rulebox.coldbox.org
- Tutorial course: https://rulebox.coldbox.org/course/
- Source: https://github.com/coldbox-modules/rulebox
- ForgeBox: https://forgebox.io/view/rulebox
