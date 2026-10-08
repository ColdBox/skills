---
name: testbox-reporters
description: "Use this skill when selecting or configuring TestBox reporters: Agent, ANTJunit, Console, Doc, Dot, JSON, JUnit, Min, MinText, Simple, Text, XML, Streaming; the TestBox 7.2 HTML reporters (Simple, Min, Dot, Doc) with light/dark themes, keyboard shortcuts, status filters and Ask AI (aiAssist, aiProviders, aiContextLines, aiStackFrames, aiPrompt, urlParams); editor links; setting reporter options (hideSkipped); how reporters show spec attachments (attach(), browser screenshots/traces/videos) and retry attempts; or creating a custom reporter by implementing the IReporter interface."
applyTo: "**/tests/**/*.{bx,bxm,cfc,cfm,cfml}"
---

# TestBox Reporters — Comprehensive Reference

## When to Use This Skill

- Choosing the right reporter for a use case (CI, development, IDE, browser)
- Configuring reporter-specific options (hideSkipped, IDE links)
- Using the AgentReporter for token-efficient output for AI agents and automation
- Using the HTML reporters (`simple`, `min`, `dot`, `doc`): filters, shortcuts, themes, run links, **Ask AI** and its options (TB7.2+)
- Using the StreamingReporter for real-time SSE output
- Building a custom reporter by implementing `IReporter`

---

## Built-In Reporters

| Reporter Key | Class | Best For |
|---|---|---|
| `agent` | `testbox.system.reports.AgentReporter` | AI agents, minimal token usage (TB7.2+) |
| `antjunit` | `testbox.system.reports.ANTJunitReporter` | Ant/legacy CI pipelines |
| `console` | `testbox.system.reports.ConsoleReporter` | CI stdout logs |
| `doc` | `testbox.system.reports.DocReporter` | Living documentation, bundle navigation (HTML) |
| `dot` | `testbox.system.reports.DotReporter` | Whole run at a glance, one dot per spec (HTML) |
| `json` | `testbox.system.reports.JSONReporter` | API / tooling consumption |
| `junit` | `testbox.system.reports.JUnitReporter` | Modern CI (GitHub Actions, Jenkins) |
| `min` | `testbox.system.reports.MinReporter` | Compact HTML: only what needs attention |
| `mintext` | `testbox.system.reports.MinTextReporter` | Compact plain-text summary |
| `simple` | `testbox.system.reports.SimpleReporter` | Complete HTML view, failures first, Ask AI |
| `text` | `testbox.system.reports.TextReporter` | Plain-text verbose output |
| `xml` | `testbox.system.reports.XMLReporter` | XML consumers, legacy tools |
| `streaming` | `testbox.system.reports.StreamingReporter` | SSE real-time output (TB7+) |

---

## Reporter Details

### `min`: Minimal HTML

A compact HTML page that lists only what needs attention, one line per failure, under the verdict banner. It is one of the four rebuilt HTML reporters, see [HTML Reporters](#html-reporters-testbox-72-redesign).

```bash
./testbox/run --reporter=min
```

The BoxLang CLI runner writes it to `report.html`. The default CLI reporter is `console`.

---

### `mintext`: Minimal Text

A compact plain-text summary (one line of totals per bundle, no HTML). Suitable for log files.

```bash
./testbox/run --reporter=mintext
```

---

### `console` — Console (colored verbose)

Prints each spec name with colored pass/fail indicator. Useful in CI logs where you want readable spec names.

#### Options

```boxlang
new testbox.system.TestBox(
    directory: { mapping: "tests.specs", recurse: true },
    reporter:  {
        type:    "testbox.system.reports.ConsoleReporter",
        options: { hideSkipped: true }   // suppress skipped spec noise
    }
).run()
```

```bash
# Via CLI
testbox run reporter=console options.hideSkipped=true
```

---

### `simple` — Simple HTML

The complete HTML report: a *Needs attention* section with every failure and error first, then every bundle with its suites and specs. It is the default of the web runners. Shared behavior of all HTML reporters is under [HTML Reporters](#html-reporters-testbox-72-redesign).

```boxlang
new testbox.system.TestBox(
    bundles  : "tests.specs.MyTest",
    reporter : "simple"
).run()
```

#### Editor Links

Failures and stack frames link to the file and line in your editor. The editor is a **request parameter**, not a reporter option: pass `editor` in the runner URL (`runner.cfm?reporter=simple&editor=idea`), or, when you build the report from code where there is no `url` scope, in the `urlParams` option. The default is `vscode`.

```boxlang
new testbox.system.TestBox(
    bundles  : "tests.specs.MyTest",
    reporter : {
        type    : "testbox.system.reports.SimpleReporter",
        options : { urlParams : { editor : "idea" } }
    }
).run()
```

| `editor` | Link |
|---|---|
| `vscode` | `vscode://file/{path}:{line}` |
| `vscode-insiders` | `vscode-insiders://file/{path}:{line}` |
| `sublime` | `subl://open?url=file://{path}&line={line}` |
| `textmate` | `txmt://open?url=file://{path}&line={line}` |
| `emacs` | `emacs://open?url=file://{path}&line={line}` |
| `macvim` | `mvim://open/?url=file://{path}&line={line}` |
| `idea` | `idea://open?file={path}&line={line}` |
| `atom` | `atom://core/open/file?filename={path}&line={line}` |
| `espresso` | `x-espresso://open?filepath={path}&lines={line}` |

Any other value produces a plain `{path}:{line}`.

---

### `json` — JSON

Returns the full result set as a JSON string. Useful for programmatic consumption, test dashboards, or custom CI tooling.

```boxlang
var results = new testbox.system.TestBox(
    directory: { mapping: "tests.specs", recurse: true },
    reporter:  "json"
).runRaw()
```

Typical JSON shape:

```json
{
  "totalDuration": 512,
  "totalSpecs": 42,
  "totalPass": 40,
  "totalFail": 1,
  "totalError": 1,
  "totalSkipped": 0,
  "labels": [],
  "bundleStats": [...]
}
```

---

### `agent` — Agent (token-efficient JSON)

Compact, single-line JSON built for AI agents: totals plus only the failed/errored specs. A passing run is a few dozen tokens. Prefer it over `json` when the consumer is an LLM.

```bash
./testbox/run --reporter=agent
```

```boxlang
var report = new testbox.system.TestBox(
    bundles  : "tests.specs.MyTest",
    reporter : { type: "testbox.system.reports.AgentReporter", options: { maxFailures: 10, includeStack: true } }
).run()
```

Output shape:

```json
{"ok":false,"totals":{"pass":120,"fail":2,"error":1,"skipped":3,"specs":126,"ms":4210},
 "failures":[{"bundle":"tests.specs.FooTest","spec":"Foo > can add","status":"failed","message":"Expected [4] but received [3]","at":"tests/specs/FooTest.cfc:42"}],
 "truncated":0}
```

| Option | Default | Purpose |
|---|---|---|
| `detail` | `failures` | `summary` (totals only), `failures`, or `all` (adds a compact `specs` list) |
| `maxFailures` | `20` | Cap on listed failures, `0` = unlimited. Overflow is counted in `truncated` |
| `maxMessageLength` | `300` | Truncates each failure message, `0` = unlimited |
| `includeStack` | `false` | Adds a `stack` array of `file:line` frames to each failure |
| `stackDepth` | `3` | Frames kept when `includeStack` is true |
| `includeSkipped` | `false` | Adds a `skipped` array of spec paths |
| `includeDebug` | `false` | Adds a `debug` array of debug buffer output |

Notes: `at` paths are relative to the web/working root. Bundle-level exceptions (for example a failing `beforeAll()`) appear in `failures` with an empty `spec`. The reporter does not change the process exit code, read `ok`.

---

### `junit` — JUnit XML

Produces JUnit-compatible XML. Use this for GitHub Actions, Jenkins, CircleCI, or any CI that parses JUnit reports:

```bash
./testbox/run --reporter=junit > results/junit.xml
```

```yaml
# GitHub Actions
- name: Run Tests
  run: ./testbox/run --reporter=junit > test-results/junit.xml

- name: Publish Test Results
  uses: EnricoMi/publish-unit-test-result-action@v2
  with:
    files: test-results/junit.xml
```

---

### `antjunit` — ANTJunit XML

Legacy Ant-compatible JUnit XML format. Use only when your build tool requires Ant-style XML.

---

### `doc` — Documentation

Your suite rendered as living documentation: a bundle navigation on the side and every suite and spec as readable text, so spec names read like sentences describing application behavior. It has the same verdict banner, filters, themes and Ask AI as the other HTML reporters.

---

### `dot`: Dot

One dot per spec, grouped by bundle. Hover a dot for its name, click it to open the details, the code and Ask AI in a drawer. Status filters **dim** the dots that do not match, so the shape of the run stays in place. Not deprecated: it was rebuilt in 7.2.

---

### `text` — Text

Verbose plain-text output: prints every spec name, full error messages and stack traces. Best for deep-dive debugging.

```bash
./testbox/run --reporter=text --stacktrace
```

---

### `xml` — XML

Generic XML representation of the result tree. Different from JUnit — use when you need to feed custom XML consumers (XSLT transforms, legacy reporting tools).

---

### `streaming` — StreamingReporter (TestBox 7+)

Emits Server-Sent Events (SSE) so results appear in real time in the terminal or browser as specs complete.

```bash
./testbox/run --stream
testbox run --streaming
```

When accessed via HTTP, the Content-Type is `text/event-stream`. Each event payload is a JSON object describing a spec result:

```
data: {"specName":"it can create a user","status":"passed","duration":12}

data: {"specName":"it can delete a user","status":"failed","message":"Expected true but got false","duration":8}
```

---

## HTML Reporters (TestBox 7.2 redesign)

`simple`, `min`, `dot` and `doc` were rebuilt in 7.2 on Bootstrap 5.3, Bootstrap Icons, Alpine.js and Prism. **Everything is inlined**, so a report is one self-contained file that works airgapped. Use them with `reporter=simple|min|dot|doc` on the runner URL, or `reporter : "simple"` programmatically.

### What every HTML reporter has

- **Verdict first**: a green or red banner with totals, a proportion bar and status chips (Pass, Failed, Error, Skipped) that filter the whole report. A bundle that could not run (for example `beforeAll()` threw) gets its own alert above the verdict.
- **Light, dark and system themes**, remembered in the browser and applied before paint.
- **Keyboard**: `F` jumps to the next failure or error, `/` focuses the search box.
- **Filters**: global status chips, a per-bundle status filter, live text search, Expand all and Collapse all. Bundles with problems start open.
- **Highlighted BoxLang and CFML code** with the failing line marked.
- **Run links** on every bundle, suite and spec. A run link is a plain link to the runner carrying only the requested target (`testBundles`, `testSuites` or `testSpecs`), so a refresh runs it again. Every other runner option falls back to its default.
- **Responsive** layout.

| Reporter | Layout | Ask AI |
|---|---|---|
| `simple` | *Needs attention* section, then every bundle | Menu on every failure and error card, plus Copy all failures |
| `min` | One line per failure, bundle exceptions and debug output | Copy all failures for AI |
| `dot` | One dot per spec, details in a drawer | Menu inside the drawer, plus Copy all failures |
| `doc` | Bundle navigation and documentation-style text | Menu on every failure and error, plus Copy all failures |

### Ask AI

Every failure and error can be handed to an assistant. The prompt is built inside the page from the spec, status, message, the code around the failing line, the first stack frames and the command that re-runs only that spec. The menu offers **Copy prompt**, **Preview prompt**, **Open in ChatGPT / Claude** and **Copy for a coding agent** (the failure as JSON in the AgentReporter format). **Copy all failures for AI** builds one prompt for the whole run. Nothing leaves the page until a provider is clicked, and the first time the page shows a notice and waits for confirmation (remembered in the browser).

Ask AI is **on by default**. Options go in the reporter `options` struct:

| Option | Default | Purpose |
|---|---|---|
| `aiAssist` | `true` | Show Ask AI. `false` removes every Ask AI control and prompt. `?aiAssist=false` switches it off for one run |
| `aiProviders` | ChatGPT and Claude | Array of `{ id, name, url }`. `url` must contain `{prompt}`. Setting it replaces the defaults |
| `aiContextLines` | `5` | Lines of code before and after the failing line |
| `aiStackFrames` | `8` | Stack frames in the prompt. Long prompts are trimmed to fit a URL |
| `aiPrompt` | built in | Custom template with the tokens `{intro}` `{spec}` `{status}` `{message}` `{code}` `{stack}` `{rerun}` |
| `urlParams` | none | Struct of request params (`editor`, `aiAssist`...) for reports produced from code, where there is no `url` scope |

```boxlang
new testbox.system.TestBox(
    bundles  : "tests.specs",
    reporter : {
        type    : "testbox.system.reports.SimpleReporter",
        options : {
            aiContextLines : 8,
            aiProviders    : [ { id : "acme", name : "Acme AI", url : "https://ai.acme.test/?p={prompt}" } ],
            urlParams      : { editor : "idea" }
        }
    }
).run()
```

### Gotchas

- **Do not parse HTML reporter output.** When the consumer is a script or an LLM, use `agent` (or `json`). The HTML reporters are for people.
- The reporter struct uses the key `type` (`{ type : "testbox.system.reports.SimpleReporter", options : {} }`), not `class`.
- `editor` and `aiAssist` come from the request. In a headless run (BoxLang CLI) there is no `url` scope, pass them in `urlParams`.
- **Removed in 7.2:** the `url.fullPage` switch. An HTML reporter always returns a complete page.
- The inlined front-end libraries are vendored in `build/vendor` and rebuilt with `box run-script assets:update` (contributors only).

---

## Spec Attachments and Attempts (TestBox 7.2+)

Files attached to a spec, with `attach( path, type = "file", name = "" )` or automatically by browser specs (failure `screenshot`, `trace` and `video` files of `BrowserSpec` / `BrowserTestCase`), are stored in the `attachments` array of the spec stats (`{ path, type, name }`), for passed and failed specs. Each reporter surfaces them:

| Reporter | Attachments |
|---|---|
| `json` | Included in the spec stats (`attachments`), along with the `attempts` count |
| `simple` | Linked under the spec |
| `junit`, `antjunit` | A `<system-out>` per spec with one `[[ATTACHMENT|/abs/path]]` line per file (understood by Jenkins and JUnit report actions) |
| `text`, `console`, streaming output | Listed under failed specs |

Specs that passed after a retry show "(passed after N attempts)" in the text, console, Simple and stream outputs. See the `testbox-browser-testing` skill for `attach()` and retries.

---

## Programmatic Reporter Configuration

### By String Key

```boxlang
new testbox.system.TestBox(
    directory: { mapping: "tests.specs", recurse: true },
    reporter:  "min"
).run()
```

### By Struct with Options

```boxlang
new testbox.system.TestBox(
    directory: { mapping: "tests.specs", recurse: true },
    reporter: {
        type:    "testbox.system.reports.ConsoleReporter",
        options: { hideSkipped: true }
    }
).run()
```

### By Full Class Path

```boxlang
new testbox.system.TestBox(
    directory: { mapping: "tests.specs", recurse: true },
    reporter:  "testbox.system.reports.JUnitReporter"
).run()
```

---

## Custom Reporter

Implement the `IReporter` interface to create a fully custom reporter.

### Interface Contract

```boxlang
// testbox/system/reports/IReporter.cfc (interface)
interface {
    // Called once before any tests run — return initial output string
    string function init( required results, required testbox )

    // Called once after all tests run — return accumulated output string
    string function runReport( required results, required testbox, struct options={} )
}
```

### Minimal Implementation

```boxlang
// tests/reporters/MyReporter.cfc
component implements="testbox.system.reports.IReporter" {

    function init( required results, required testbox ) {
        return ""
    }

    function runReport( required results, required testbox, struct options={} ) {
        var sb = []

        sb.append( "=== Test Results ===" )
        sb.append( "Total: #results.totalSpecs# | Pass: #results.totalPass# | Fail: #results.totalFail# | Error: #results.totalError# | Skipped: #results.totalSkipped#" )
        sb.append( "Duration: #results.totalDuration#ms" )

        if ( results.totalFail > 0 || results.totalError > 0 ) {
            sb.append( "" )
            sb.append( "FAILURES:" )
            for ( var bundle in results.bundleStats ) {
                for ( var suite in bundle.suiteStats ) {
                    for ( var spec in suite.specStats ) {
                        if ( spec.status == "failed" || spec.status == "error" ) {
                            sb.append( "  [#spec.status.uCase()#] #spec.name#" )
                            if ( spec.keyExists( "failMessage" ) ) {
                                sb.append( "    #spec.failMessage#" )
                            }
                        }
                    }
                }
            }
        }

        return sb.toList( chr(10) )
    }

}
```

### Using Your Custom Reporter

```boxlang
new testbox.system.TestBox(
    directory: { mapping: "tests.specs", recurse: true },
    reporter:  "tests.reporters.MyReporter"
).run()

// or by instance
new testbox.system.TestBox(
    directory: { mapping: "tests.specs", recurse: true },
    reporter:  new tests.reporters.MyReporter()
).run()
```

---

## Reporter Selection Guide

| Scenario | Reporter |
|---|---|
| Fast dev feedback in terminal | `console` or `mintext` |
| Rich browser debugging | `simple` (add `&editor=idea` for editor links) |
| Whole run at a glance in a browser | `dot` |
| Compact browser view of what needs attention | `min` |
| CI (GitHub Actions / Jenkins) | `junit` or `antjunit` |
| Log file output | `text` or `mintext` |
| Test dashboard / API | `json` |
| AI agents / minimal tokens | `agent` |
| Real-time streaming | `streaming` (or `--stream` flag) |
| Living documentation | `doc` |
| Custom pipeline | Custom class via `IReporter` |
