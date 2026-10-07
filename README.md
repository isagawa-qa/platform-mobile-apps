# Isagawa QA Platform (Mobile)

![Python](https://img.shields.io/badge/Python-3.12-blue)
![Appium](https://img.shields.io/badge/Appium-3.7.0-green)

[![iOS reference suite](https://github.com/isagawa-qa/platform-mobile-apps/actions/workflows/ios-reference.yml/badge.svg)](https://github.com/isagawa-qa/platform-mobile-apps/actions/workflows/ios-reference.yml)
[![prod-test L3](https://github.com/isagawa-qa/platform-mobile-apps/actions/workflows/prod-test-l3.yml/badge.svg)](https://github.com/isagawa-qa/platform-mobile-apps/actions/workflows/prod-test-l3.yml)

AI-driven Appium test automation for native and hybrid mobile apps on iOS and Android. Describe a test in plain English, and an AI agent opens a live device session, explores the app, writes the test into one consistent framework, runs it, and separates application defects from test problems. Every generated test is plain Python code your team owns and maintains.

---

## Status

The released platform runs its reference flow against a real iOS simulator in CI. On the GitHub `macos-15` runner, both `ios-reference.yml` (the reference flow) and `prod-test-l3.yml` (the live L3 batch) execute green against the simulator.

**iOS from Windows, free:** the `ios-tunnel` workflow holds a GitHub-hosted simulator open behind a Cloudflare quick tunnel. On 2026-10-06 the reference flow ran from a Windows host through it and passed (see [`SETUP.md`](SETUP.md#4b1-ios-on-windows)).

Android runs on a **local emulator**: an API 34 AVD driven through Appium with the UiAutomator2 driver, recorded end to end in [`SETUP.md`](SETUP.md#windows-host). There is no Android CI workflow.

Cloud real-device execution is **not yet supported**. The cloud cells of the support matrix (`L3-03`, `L3-04`, `L3-05`) are OPEN: they need BrowserStack credentials and an exercised run before they can be claimed.

Clean-clone acceptance, a fresh `git clone` taken through `/kernel/domain-setup` and `/qa-workflow` on a bare host, is still open on Windows (`INT-05`) and macOS (`INT-10`); neither has a recorded end-to-end pass yet.

Everything else in the support matrix below is an install path without a run record, or needs an input this repo does not ship (a device id, an Appium endpoint, or cloud credentials).

---

## The Problem

AI can generate mobile tests in seconds. Mobile makes the failure modes worse than web:

- Locators differ per platform, so a generated test silently breaks on the other OS.
- A device, host OS, and Appium matrix means "works on my machine" rarely transfers.
- Appium sessions are flaky; without discipline the agent papers over failures with retries and sleeps.
- The same architecture mistakes repeat every session.

Generate, breaks on Android, fix, breaks the iOS locators, start over.

---

## The Solution

iOS and Android tests share one structure, so platform differences are handled in one place instead of breaking tests on the other OS. The agent works under guardrails from the [Isagawa Kernel](https://github.com/isagawa-co/isagawa-kernel), so it follows your conventions instead of papering over failures with retries and sleeps, and it does not repeat mistakes it has already made.

## How It Works

There is no URL: a native app is launched from the session capabilities; a `-web` platform targets the device browser.

1. **Describe.** Give the agent a requirement in plain English, the app, and the platform (iOS, Android, or the device browser).
2. **Explore.** The agent opens a live Appium session on a simulator, emulator, or device and maps the screens the test needs.
3. **Build.** It writes the test and its supporting code into one consistent structure, reusing what already exists instead of duplicating it.
4. **Run.** The test runs with pytest against the chosen device.
5. **Triage.** On a failure, the agent separates an application defect from a test problem and proposes a fix for you to approve.
6. **Learn.** What went wrong is recorded, so later tests avoid the same mistake.

## Support Matrix

What runs where today. `runs today` means an exercised path (a CI run on record, or an end-to-end run recorded in `SETUP.md`). For the `ci` row the executing host is the GitHub `macos-15` runner; each host column records whether a developer on that host can trigger it and see the result.

| Platform | Device location | macOS host | Windows host |
|---|---|---|---|
| iOS | local | supported, unverified | not supported |
| iOS | remote | needs IOS_DEVICE_APPIUM_URL + IOS_UDID | runs today (free `ios-tunnel` workflow, [SETUP 4b.1](SETUP.md#4b1-ios-on-windows)) |
| iOS | cloud | needs BrowserStack credentials | needs BrowserStack credentials |
| iOS | ci | runs today | runs today |
| Android | local | supported, unverified | runs today |
| Android | remote | needs ANDROID_UDID + Appium endpoint | needs ANDROID_UDID + Appium endpoint |
| Android | cloud | needs BrowserStack credentials | needs BrowserStack credentials |
| Android | ci | not supported | not supported |

iOS has no local Apple toolchain on Windows, so local iOS is macOS only. Android has no `ci` device location by design. Per-cell evidence is kept with the engineering notes, not here.

---

## Quick Start

Full, host-specific steps live in [`SETUP.md`](SETUP.md). This section is the map.

### Prerequisites

- Python 3.10+ (3.12 in CI), Node.js 22+, JDK 17
- Appium 3.7.0 with the XCUITest (iOS) and UiAutomator2 (Android) drivers
- A simulator, emulator, or device for your platform
- [Claude Code](https://claude.ai/claude-code)

Host details: [`SETUP.md#prerequisites-all-hosts`](SETUP.md#prerequisites-all-hosts).

### Install

```bash
git clone https://github.com/isagawa-qa/platform-mobile-apps.git
cd platform-mobile-apps
python -m venv venv
source venv/bin/activate   # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env
```

See [`SETUP.md#step-2-python-environment`](SETUP.md#step-2-python-environment).

### Configure

App builds are downloaded at run time and never committed. Fetch the reference apps, then edit `framework/resources/config/environment_config.json` for your app and device. See [`SETUP.md#step-3-fetch-the-reference-apps`](SETUP.md#step-3-fetch-the-reference-apps).

Then complete your host section: [`SETUP.md#macos-host`](SETUP.md#macos-host) or [`SETUP.md#windows-host`](SETUP.md#windows-host), and verify with [`SETUP.md#step-5-verify-setup`](SETUP.md#step-5-verify-setup).

### Run

Boot your simulator or emulator and the Appium server (per your host section), then from Claude Code. The repo ships before domain setup (no protocol, no domain hooks yet), so the first two commands are both required, in this order:

```bash
claude                  # start in the project directory
> /kernel/session-start # initialize the session
> /kernel/domain-setup  # generate the protocol and domain hooks; restart Claude Code when it says so
> /qa-workflow          # generate your first test
> /pr                   # review generated code against the architecture
```

Details and the verify step: [`SETUP.md#step-6-run-kernelsession-start-then-kerneldomain-setup`](SETUP.md#step-6-run-kernelsession-start-then-kerneldomain-setup).

### Tests

```bash
PYTHONPATH=tests pytest -p conftest framework/_reference/tests -m ios --platform=ios          # iOS reference flow (what CI runs)
PYTHONPATH=tests pytest -p conftest framework/_reference/tests -m android --platform=android  # Android reference flow

pytest --collect-only                  # no device yet: confirm the project collects (no session)
```

The reference suite is documented at [`SETUP.md#step-7-run-the-reference-suite`](SETUP.md#step-7-run-the-reference-suite).

---

## Troubleshooting

Start with [`SETUP.md#troubleshooting`](SETUP.md#troubleshooting). If a problem is a framework issue rather than your setup, report it on [GitHub Issues](https://github.com/isagawa-qa/platform-mobile-apps/issues) for this repo.

---

## Other Platforms

| Platform | Domain | Repo |
|----------|--------|------|
| **platform-mobile-apps** | Mobile (Appium) test automation | (this repo) |
| **platform-selenium** | Selenium web test automation | [isagawa-qa/platform-selenium](https://github.com/isagawa-qa/platform-selenium) |
| **platform-playwright** | Playwright web test automation | [isagawa-qa/platform-playwright](https://github.com/isagawa-qa/platform-playwright) |

---

## Contributing

Adding a screen, platform, or device provider? See [`CONTRIBUTING.md`](CONTRIBUTING.md) for the architecture rules, the areas we're looking for, and the PR process.

---

## Author

Built by Alain Ignacio, QA lead and test automation architect.
Portfolio: [alain-ignacio.github.io](https://alain-ignacio.github.io) · LinkedIn: [linkedin.com/in/alain-ignacio](https://www.linkedin.com/in/alain-ignacio)

## License

Proprietary. Copyright (c) 2025 Isagawa. All rights reserved. Source is available for evaluation only. See [LICENSE](LICENSE).
