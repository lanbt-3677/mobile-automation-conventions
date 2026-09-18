# Mobile Automation Conventions

This document defines reusable engineering conventions for Android or cross-platform mobile automation projects built with Appium, Java, Maven, Cucumber, and TestNG.
## Recommended Architecture

```text
Cucumber feature files (business behaviour)
        ↓ filtered by a Cucumber tag
TestNG suite XML → Cucumber/TestNG runner
        ↓
Cucumber hooks ─────────→ scenario setup, teardown
        ↓
Step definitions
        ↓
Screen/page objects and reusable UI components
        ↓
Driver manager, waits, capabilities, configuration, test data, logging
        ↓
Appium server → Android/iOS device or emulator

Test results, logs, and safe attachments → selected report and CI artifacts
```

Responsibilities:

- **Feature files** describe behaviour in business language. Do not put selectors, waits, or technical setup in them.
- **TestNG suite XML and runner** start Cucumber under TestNG. The suite selects the runner; Maven/CI configuration supplies the Cucumber tag expression and other run parameters. Keep runner code minimal; do not create a runner for every scenario.
- **Hooks** create and close the driver session, prepare a known state, and collect safe diagnostics after a failure. They should not contain business-flow steps.
- **Step definitions** translate Gherkin into high-level screen actions and assertions. Keep them short; move repeated UI logic elsewhere.
- **BaseScreen** owns the common UI operations used by screens and components, including driver access, explicit-wait helpers, element lookup, scrolling, keyboard handling, and safe diagnostics.
- **Screen/page objects and components** extend or use `BaseScreen` and own only locators and interactions for one screen, dialog, or reusable UI area.
- **Framework services** own driver lifecycle, waits, capabilities, configuration loading, test-data helpers, logging, and report attachments.
- **Reporting** turns execution output into the chosen test report and keeps its raw results/attachments available to CI.
- **CI** chooses the suite, target environment, device, app build, secrets, and artifact retention.

## Suggested Project Layout

```text
src/
├── main/java/.../
│   ├── config/          # typed configuration, validation, environment resolution
│   ├── driver/          # driver factory, capabilities, driver lifecycle
│   ├── screen/
│   │   ├── BaseScreen.java    # common UI actions, waits, scrolling, diagnostics
│   │   └── ...Screen.java     # screen-specific locators and business UI actions
│   └── data/            # non-secret test-data models
├── test/java/.../
│   ├── cucumber/hooks/  # scenario setup, teardown, safe diagnostics
│   ├── cucumber/steps/  # thin business-step implementations
│   └── runners/         # Cucumber/TestNG runner
├── test/resources/
│   ├── features/        # Gherkin files grouped by business capability
│   ├── testdata/        # non-secret fixtures/catalogues
│   ├── testng/          # smoke, regression, focused-suite XML files
│   └── reporting/       # optional configuration for the selected reporting tool
├── apps/                # APK only when distribution policy permits it
├── scripts/             # doctor, local-run, Appium helper scripts
├── docs/                # architecture, runbook, project-specific guidance
├── pom.xml
├── .github/workflows/    # GitHub Actions, when GitHub is the selected CI platform
├── azure-pipelines.yml   # Azure DevOps, when Azure DevOps is the selected CI platform
└── Jenkinsfile           # Jenkins, when Jenkins is the selected CI platform
```

## BaseScreen Convention

Every screen object and reusable component must inherit from, or compose, `BaseScreen`. Keep framework-wide UI mechanics in this class so they are implemented, configured, and diagnosed consistently. Typical responsibilities include:

- access to the current driver and shared explicit waits;
- finding an element, waiting for it to be visible/clickable/gone, and performing guarded tap, type, clear, and text-read actions;
- scrolling or swiping to a known element, hiding the keyboard, and handling platform-common UI actions;
- timeout diagnostics such as current activity, safe screenshot/page-source capture, and Appium-log correlation.

`BaseScreen` must not contain screen-specific locators, product assertions, or business-flow methods. Those belong in the relevant screen or component class. Step definitions call high-level screen methods; they must not repeat low-level Appium interactions that belong in `BaseScreen`.

## Configuration and Secrets

Use a clear precedence order:

```text
Maven -D property → environment variable → ignored local file → safe default
```

Keep environment-specific values configurable:

- platform, automation engine, device UDID/name, OS version
- Appium URL and timeouts
- application package/activity or APK path/version
- reset and permission behaviour
- target backend environment (for example, QA, staging, or a dedicated test environment), including its base URL or API endpoint
- feature flags that change the flow under test; make their expected value explicit when a scenario depends on one

Rules:

- Never commit usernames, passwords, PINs, access tokens, API keys, private certificates, or other sensitive connection credentials.
- Put local-only credentials in an ignored file such as `local.properties`.
- Store CI credentials in the CI platform's encrypted secrets store.
- Use dedicated QA accounts.
- Keep a versioned, non-secret reference that maps account roles (for example, standard member or administrator) and named environments to their purpose and required settings. Store the actual credentials separately in local/CI secrets.
- At the start of a run, check that every required setting is present and usable before creating a session. If one is missing, stop early with a specific message such as: `APP_PATH is required for local Android runs` or `QA_USER_PASSWORD must be set for @requires-qa-account scenarios`.
- Use isolated accounts and unique test data where possible; do not use personal accounts or mutable shared production records.
- State whether a scenario creates or changes data, and provide cleanup or a documented data-reset process.

## Locator Convention

Locator quality is the largest driver of mobile-test stability. Use this preference order:

1. A stable automation/accessibility identifier agreed with the app team. On Android this is commonly the view's `content-desc` attribute and is located by Appium accessibility ID; it is not an XPath expression.
2. A stable Android `resource-id` assigned by the app, such as `com.example:id/sign_in_button`.
3. Visible text only when that exact text is part of the product requirement and is not expected to change with copy updates or localization.
4. A narrowly scoped native selector based on a stable parent/child relationship when there is no identifier. Document the locator debt and ask for an identifier.
5. XPath only as a last, constrained fallback; never use a full absolute hierarchy path.

Locators cannot be predicted or guessed from a screen name, code convention, visible UI, or a previous app version. Verify every locator against the running application before adding it to automation:

- When working manually, inspect the target element with **Appium Inspector** and copy/validate the actual available attributes.
- When working through an agent, use the **Appium MCP** to inspect the live UI and obtain the locator attributes.

Do not use:

- indexes such as `//Button[3]` unless no alternative exists and the reason is documented.
- coordinates, sleeps, random generated IDs, or styling/layout hierarchy.
- broad text selectors that can match multiple controls.

Ask application developers to provide stable automation identifiers as part of the feature definition. They make tests less dependent on UI layout and text changes.

## Waits and Synchronization

- Use explicit waits for a meaningful state: visible, clickable, gone, text changed, activity changed, or network-driven screen ready.
- Keep a small set of shared wait helpers in the base screen class.
- Do not use `Thread.sleep` as synchronization. A short delay may be acceptable only for a documented platform animation limitation.
- Make timeout values configurable and use one default across the framework.
- Capture the observed state on timeout: current activity, page source when safe, screenshot when safe, and Appium log reference.

## Driver and Scenario Lifecycle

- Create one Appium session per scenario by default. This prevents leaked authentication and scenario ordering dependencies.
- Always quit the driver in an `@After` hook, even after a failure.
- Keep `noReset`, `fullReset`, and permission capabilities explicit and environment-configurable.
- Use a known initial state before each scenario. If resetting is expensive, document a safe cleanup flow.
- Do not share a static driver across parallel scenarios without a thread-safe driver manager.
- Include the scenario name and device identifier in logs.

## Cucumber, TestNG, and Suite Conventions

Write scenarios from the user's perspective:

```gherkin
Scenario: Existing member signs in successfully
  Given the mobile application is launched
  When the member signs in with a valid account
  Then the authenticated home screen is displayed
```

Guidelines:

- One scenario should test one business outcome.
- Use scenario outlines for true data variations, not unrelated workflows.
- Put technical retries, setup, and cleanup in hooks/framework code, not feature files.
- Use tags to express test intent and risk, for example `@smoke`, `@regression`, `@authentication`, `@payment`, `@requires-qa-account`.

## Suite Strategy

| Suite | Purpose | Expected size |
| --- | --- | --- |
| Smoke | Critical launch, login, and one primary journey | Minutes; stable and small |
| Regression | Wider functional coverage and edge cases | Longer-running |
| Focused | One feature or defect verification | Short and narrowly scoped |
| Device compatibility | Critical flows across several models/OS versions | Controlled parallelism |

Smoke must remain small. If every test becomes smoke, the suite loses its ability to provide quick, reliable feedback.

## Reporting and Diagnostics

Every CI run should publish:

- machine-readable Cucumber results (for example JSON) and a human-readable test summary;
- output from the selected reporting solution, when one is configured;
- Appium server log;
- screenshots and page source only when safe;
- device, app, framework, and environment metadata.

Security rules for diagnostics:

- Do not attach credentials, tokens, session cookies, or payment data.
- Avoid screenshots/page source on credential-entry screens unless explicitly approved.
- Keep reports private and set an appropriate artifact retention period.

Choose reporting tooling that meets the team's needs for result visibility, trends, duration, retries, attachments, and failure categorization. This may be Cucumber/TestNG-native output, a CI test-report view, or a dedicated tool such as Allure. Publish the supported report and its raw results, where applicable, as CI artifacts.

## Change and Review Convention

Before merging an automation change:

- review the affected code and configuration for duplicated logic, hard waits, unstable locators, leaked secrets, and incomplete cleanup;
- run Maven compilation and the relevant Cucumber/TestNG suite;
- verify that tags, suite selection, test data, diagnostics, and reports remain correct;
- document any known locator, environment, or device limitation with the change.
