---
name: coldbox-testing-ci
description: "Use this skill when setting up continuous integration for ColdBox/BoxLang applications, configuring GitHub Actions workflows for automated testing, running TestBox in CI/CD pipelines, setting up matrix testing across multiple CF engines, uploading coverage reports, or configuring test caching and artifact management."
---

# Continuous Integration Testing for ColdBox

## Overview

Continuous Integration (CI) automatically runs tests on every commit, ensuring the application stays in a working state. ColdBox/BoxLang applications use CommandBox to start servers and run TestBox tests in CI environments like GitHub Actions or GitLab CI.

## Language Mode Reference

Examples use **BoxLang (`.bx`)** syntax by default. Adapt for your target language:

| Concept | BoxLang (`.bx`) | CFML (`.cfc`) |
|---------|-----------------|---------------|
| Class declaration | `class [extends="..."] {` | `component [extends="..."] {` |
| DI annotation | `@inject` above `property name="svc";` | `property name="svc" inject="svc";` |
| View templates | `.bxm` suffix | `.cfm` / `.cfml` suffix |
| Tag prefix | `<bx:if>`, `<bx:output>`, `<bx:set>` | `<cfif>`, `<cfoutput>`, `<cfset>` |

> **CFML Compat Mode**: With BoxLang + CFML Compat module, `.bx` and `.cfc` files coexist freely. BoxLang-native classes use `class {}` (`.bx` files); CFML-compat classes use `component {}` (`.cfc` files).

## GitHub Actions — Basic Workflow

```yaml
# .github/workflows/tests.yml
name: Tests

on:
  push:
    branches: [ main, development ]
  pull_request:
    branches: [ main, development ]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup CommandBox
        uses: ortus-solutions/setup-commandbox@v2

      - name: Install Dependencies
        run: box install

      - name: Start Server
        run: box server start port=8080 --noSaveSettings

      - name: Run Tests
        run: box testbox run

      - name: Stop Server
        if: always()
        run: box server stop
```

## Multi-Engine Matrix Testing

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [ main, development ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    name: Test on ${{ matrix.cfengine }}
    runs-on: ubuntu-latest

    strategy:
      fail-fast: false
      matrix:
        cfengine:
          - lucee@5.4
          - adobe@2021
          - boxlang@1.0.0

    steps:
      - uses: actions/checkout@v4

      - name: Setup CommandBox
        uses: ortus-solutions/setup-commandbox@v2

      - name: Cache CommandBox dependencies
        uses: actions/cache@v3
        with:
          path: |
            ~/.CommandBox
            ~/.box
          key: ${{ runner.os }}-commandbox-${{ hashFiles('**/box.json') }}

      - name: Install Dependencies
        run: box install

      - name: Start Server
        run: box server start cfengine=${{ matrix.cfengine }} port=8080 --noSaveSettings

      - name: Run Tests
        run: box testbox run

      - name: Generate Coverage (BoxLang only)
        if: matrix.cfengine == 'boxlang@1.0.0'
        run: box testbox run --coverage --reporter=Cobertura --outputFile=coverage.xml

      - name: Upload Coverage
        if: matrix.cfengine == 'boxlang@1.0.0'
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage.xml
          flags: unittests

      - name: Archive Test Results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: test-results-${{ matrix.cfengine }}
          path: tests/results/

      - name: Stop Server
        if: always()
        run: box server stop
```

## Full CI/CD Pipeline with Deploy

```yaml
# .github/workflows/pipeline.yml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: testdb
        ports:
          - 3306:3306
        options: --health-cmd="mysqladmin ping" --health-interval=10s

    steps:
      - uses: actions/checkout@v4

      - name: Setup CommandBox
        uses: ortus-solutions/setup-commandbox@v2

      - name: Install Dependencies
        run: box install

      - name: Configure Test Environment
        run: |
          cp .env.test .env
          box dotenv show

      - name: Start Server
        run: box server start port=8080 --noSaveSettings

      - name: Run Database Migrations
        run: box migrate up --datasource=testdb

      - name: Run Tests
        run: box testbox run --reporter=JUnit --outputFile=test-results.xml

      - name: Publish Test Results
        if: always()
        uses: dorny/test-reporter@v1
        with:
          name: TestBox Results
          path: test-results.xml
          reporter: java-junit

      - name: Stop Server
        if: always()
        run: box server stop

  deploy:
    name: Deploy to Staging
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to staging
        run: |
          # Your deployment steps here
          echo "Deploying to staging..."
```

## GitLab CI Configuration

```yaml
# .gitlab-ci.yml
stages:
  - test
  - deploy

variables:
  COMMANDBOX_HOME: $CI_PROJECT_DIR/.CommandBox

test:
  stage: test
  image: ubuntu:22.04

  cache:
    key: $CI_COMMIT_REF_SLUG
    paths:
      - .CommandBox/

  before_script:
    - apt-get update && apt-get install -y curl
    - curl -fsSL https://downloads.ortussolutions.com/debs/gpg | apt-key add -
    - echo "deb https://downloads.ortussolutions.com/debs/noarch /" >> /etc/apt/sources.list.d/commandbox.list
    - apt-get update && apt-get install commandbox
    - box install

  script:
    - box server start port=8080 --noSaveSettings
    - box testbox run
    - box server stop

  artifacts:
    when: always
    reports:
      junit: tests/results/test-results.xml
    paths:
      - tests/results/
    expire_in: 1 week
```

## box.json Test Scripts

```json
{
    "name": "my-coldbox-app",
    "scripts": {
        "start": "box server start",
        "stop": "box server stop",
        "test": "box testbox run",
        "test:coverage": "box testbox run reporter=HtmlCoverage",
        "test:unit": "box testbox run labels=unit",
        "test:integration": "box testbox run labels=integration",
        "test:ci": "box testbox run reporter=JUnit outputFile=results.xml"
    },
    "testbox": {
        "runner": "http://localhost:8080/tests/runner.cfm",
        "reporter": "text",
        "recursive": true,
        "labels": "",
        "directory": "tests/specs"
    }
}
```

## Environment Configuration for CI

```boxlang
// config/environments/testing.cfc
component extends="config.ColdBox" {

    function configure() {
        super.configure()

        // Override settings for CI environment
        coldbox.handlersIndexAutoReload = false
        coldbox.reinitPassword          = ""

        // Use test datasource
        datasources["default"] = {
            driver:   "MySQL",
            host:     "localhost",
            port:     3306,
            database: "testdb",
            username: getSystemSetting( "DB_USER", "test" ),
            password: getSystemSetting( "DB_PASSWORD", "test" )
        }

        // Disable email sending in tests
        coldbox.customInterceptionPoints = "onEmailSend"
    }
}
```

## CI Best Practices

1. **Cache CommandBox** dependencies to speed up builds (`~/.CommandBox`, `~/.box`)
2. **Use `--noSaveSettings`** when starting servers to avoid committing server settings
3. **Run unit and integration tests separately** — fail fast on unit tests before running slower integration tests
4. **Archive test results** as artifacts for debugging failed builds
5. **Use matrix builds** to verify compatibility across multiple CF engines
6. **Set environment variables** for secrets via CI secrets manager, never hardcode
7. **Health check services** (MySQL, Redis) before running tests that depend on them
