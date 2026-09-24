# Mobile Automation Conventions

This document defines reusable engineering conventions for Android mobile automation projects built with Appium, Java, Maven, Cucumber, and TestNG.

## Readiness Checklist (Before Starting Automation)
**Scope and requirements**

- Test cases selected for automation are stable, repeatable and high value (smoke and main regression flows first).
- Each selected test case has clear acceptance criteria and expected results.
- Out-of-scope items are listed (for example: maps, camera).
- Supported Android versions and device models are agreed.

**App build**

- A build (APK) is available, with a known way to get the latest one (pipeline artifact or store link).
-`appPackage` and the launch `appActivity` are known.
- The QA build is automation-friendly
- The build version is visible so it can be recorded in reports.

**Environment and data**

- Dedicated QA accounts exist for each role, with one account per parallel thread.
- A test data catalogue exists (which account and data to use for which scenario).
- A cleanup or data reset method exists (API, SQL script, or a documented manual process).

**Devices**

- Devices or emulator.
- Each device: Developer options and USB debugging on.
- Each device: language and region fixed, Play Store auto-update off.
- Cloud devices (for example BrowserStack): account, access keys.

**Local tools**

- JDK, Maven, Node.js, Appium 2, UiAutomator2 driver and Android SDK platform-tools installed.
- `adb devices` shows each device as `device`.
- Appium Inspector connects and shows the app's elements.

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

## CI Setup

- **Store the pipeline file in the test repository.** Review pipeline changes in a pull request, the same way as test code.
- **Use one pipeline for all test runs.** Choose what to run with parameters: suite, environment, and app build.
- **Download the app (APK) from where the developers publish their builds.** Do not save APK files in the test repository. They are large, and they quickly become outdated.
- **Let only one pipeline use a device at a time.** If two pipelines use the same phone, they break each other's tests.
- **Name each CI secret the same as its config key**, in uppercase with `_` instead of `.`: `qa.standard.password` becomes `QA_STANDARD_PASSWORD`. Make sure CI hides secret values in logs.

Stages: fast checks (compile, dry run) → environment check (Appium `/status`, `adb devices`) → run suite → publish results (always) → clean up.

| Trigger | Suite | Blocks merge / release? |
| --- | --- | --- |
| Pull request | Smoke | Yes |
| Nightly or manual trigger | Regression | No; failures notify the team |
| Release candidate | Smoke on the release build | Yes |
| Manual | Any suite or tag | No |

Rules:

- Show the suite, tags, app version and device in the run name or summary.
- Set a job timeout, so a hung session does not block the runner.
- Keep artifacts for 14 to 30 days.
