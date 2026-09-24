# Mobile Automation Conventions

This document defines reusable engineering conventions for Android mobile automation projects built with Appium, Java, Maven, Cucumber, and TestNG.

## Recommended Architecture

```text
Cucumber feature files (business behaviour)
        ↓ filtered by a Cucumber tag
TestNG suite XML → Cucumber/TestNG runner
        ↓
Cucumber hooks ─────────→ driver start/quit, failure evidence, cleanup
        ↓
Step definitions
        ↓
Screen objects
        ↓
BaseScreen
        ↓
Driver/Config: Driver factory, capabilities, config
        ↓
Appium server → Android device or emulator

```

Responsibilities:

- **Feature files** describe behaviour in business language. 
- **TestNG suite XML and runner** start Cucumber and select a suite
- **Hooks** create and close the driver session, collect evidence, then clean up
- **Step definitions** translate Gherkin into high-level screen actions and assertions
- **BaseScreen** shared UI mechanics: waits, tap, type, read text, scroll, hide keyboard
- **Screen/page objects and components** extend or use `BaseScreen` and own only locators and interactions for one screen
- **Framework** driver factory, capabilities, config, test data, logging

## Suggested Project Layout

```text
src/main/java/.../
  config/        # config loading and validation
  driver/        # driver factory, capabilities
  screens/       # BaseScreen + one class per screen/component
src/test/java/.../
  hooks/
  stepdefinitions/
  runners/
src/test/resources/
  features/      # grouped by business area
  testdata/      # non-secret data only
  testng/        # smoke.xml, regression.xml
pom.xml

```

## Locators

Use this order:
1. Accessibility ID (content-desc) agreed with developers.
2. resource-id (for example`com.example:id/login_button`).
3. Exact visible text, only if the text is part of the requirement and not localized.
4. UiAutomator selector with a stable parent/child relationship.
5. Relative XPath

Rules:
- Verify every locator on the running app with Appium Inspector. Do not guess locators.
- Do not use coordinates or layout-based paths. If there is no other option, add a comment with the reason and ask developers for an ID.

## Waits
- Use explicit waits only (visible, clickable, gone, text present, activity changed).
- Do not use Thread.sleep for synchronization. If it is unavoidable, add a comment with the reason.
- One configurable default timeout for the whole framework.
- All wait helpers live in BaseScreen.

## Driver and Session
- One Appium session per scenario. Start in @Before, always quit in @After, even on failure.
- For parallel runs, give each device its own UDID and systemPort.
- Each scenario starts from a known state and must not depend on another scenario.

## Configuration and Secrets
- Never commit usernames, passwords, PINs, access tokens, API keys, certificates, or other sensitive connection credentials.
- Local secrets go in local.properties (in .gitignore). CI secrets go in the CI secret store.
- Use dedicated QA accounts. No personal or production accounts.

## Cucumber and Tags
- One scenario = one business outcome.
- Use Scenario Outline only for data variations.
- Write steps from the user's view, not the UI's (When the user signs in, not When the user taps button X).
- Standard tags: @smoke, @regression, plus feature tags such as @login, @payment.
- Smoke stays small: app launch, login, one main journey.

## Test Data
- Use unique or generated data where possible.
- Mark scenarios that create or change data, and clean up after them.
- Keep non-secret test data in src/test/resources/testdata.

## Reporting and Diagnostics
- On failure, attach: screenshot, page source, current activity
- Every CI run publishes: Cucumber/TestNG results, the test report (for example, Allure), Appium log, and device/app/environment info.
- Security: Never attach credentials, tokens, or payment data. Skip screenshots and page source on credential-entry screens. Keep reports private.

## Review Checklist

Before merge:

 - Code compiles and the affected suite passes locally.
 - No Thread.sleep, absolute XPath, or index locators without a reason.
 - No secrets in code, config, or reports.
 - No duplicated logic between steps and screens.
 - Tags and test data are correct; cleanup is in place.
 - Known locator or device limitations are documented.
