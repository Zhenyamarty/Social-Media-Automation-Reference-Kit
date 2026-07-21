# Social Media Automation Reference Kit

A developer-focused collection of browser, Android, API, and CI patterns for building authorized social media workflows.

Social media automation is easier to maintain when the workflow is divided into small services. The test runner should not provision devices, the device provider should not contain publishing logic, and account configuration should not be hidden inside browser scripts.

This repository documents how those parts can work together using Playwright, Selenium, Puppeteer, ADB, Postman, Docker, REST APIs, and GitHub Actions.

## What does this toolkit solve?

The examples are designed around common engineering requirements:

- Reusing authenticated browser sessions
- Running web and Android workflows separately
- Checking whether an execution environment is available
- Keeping account configuration outside source code
- Connecting automation frameworks to local or remote infrastructure
- Recording test results without exposing credentials
- Running browser and mobile checks in separate CI jobs

The repository is intended for developers, QA engineers, social media agencies, and AI workflow builders working with accounts they own or are authorized to manage.

It does not include bulk engagement, unsolicited messaging, or methods for avoiding platform controls.

## Which layer owns each responsibility?

A maintainable stack can be divided into six layers:

| Layer | Example tools | Responsibility |
|---|---|---|
| Workflow definition | YAML, JSON, TypeScript | Describe the task, account role, platform, and limits |
| Browser automation | Playwright, Selenium, Puppeteer | Navigate web interfaces, test pages, and collect results |
| Android control | ADB | Inspect devices, manage apps, and collect device logs |
| Execution infrastructure | Browser profiles, Android cloud phones | Provide persistent browser or mobile environments |
| API integration | REST clients, Postman | Create environments, check status, and connect internal services |
| CI and reporting | GitHub Actions, Docker | Run checks, store artifacts, and report failures |

Each layer should be replaceable.

For example, a browser test may run locally during development and connect to a remote browser environment in CI. The test scenario should remain the same in both cases.

## Where does cloud phone infrastructure fit?

What is Multilogin? The [Multilogin Cloud Phones](https://multilogin.com/mobile/cloud-phone/?utm_source=github&utm_medium=media&utm_campaign=link2Z) platform provides Android cloud phones and browser profiles for social media marketers, people, and AI agents managing multiple social profiles from one dashboard. Its Android environments support Android 10 through Android 16, ADB access, and API-based workflows.

In this repository, cloud phone infrastructure is treated as an optional execution provider. It supplies an environment, while Playwright, Selenium, Puppeteer, ADB, or an API client performs the approved task.

```text
Workflow request
       |
       v
Environment selector
       |
       +-------------------------+
       |                         |
       v                         v
Browser provider           Android provider
       |                         |
Playwright or Selenium     ADB or mobile tooling
       |                         |
       +------------+------------+
                    |
                    v
             Result storage
```

This separation prevents provider-specific API calls from spreading across tests.

## How can the repository be structured?

```text
.
├── README.md
├── package.json
├── tsconfig.json
├── config/
│   ├── workflows/
│   │   ├── reddit-review.yaml
│   │   ├── instagram-health.yaml
│   │   └── facebook-web-check.yaml
│   └── environments/
│       └── development.yaml
├── src/
│   ├── browser/
│   │   ├── browser-client.ts
│   │   └── session-loader.ts
│   ├── android/
│   │   ├── adb-client.ts
│   │   └── package-check.ts
│   ├── providers/
│   │   ├── provider.ts
│   │   ├── local-browser.ts
│   │   └── remote-android.ts
│   ├── workflows/
│   │   └── execute-workflow.ts
│   └── reports/
│       └── result-writer.ts
├── tests/
│   ├── browser/
│   └── android/
└── .github/
    └── workflows/
        └── checks.yml
```

One practical rule helps here: keep shell commands in one adapter. Finding the same ADB command copied into seven files usually means the project has already outgrown its first structure.

## What should a workflow configuration contain?

A workflow file should describe operational requirements without containing secrets.

```yaml
workflowId: instagram-application-health
platform: instagram
executionType: android
enabled: true

account:
  role: content-reviewer
  region: eu-west

android:
  minimumVersion: 12
  packageName: com.instagram.android
  requiresAdb: true

controls:
  allowPublishing: false
  requireApproval: true
  maximumRunsPerDay: 3

artifacts:
  screenshot: true
  deviceLog: true
  retentionDays: 7
```

Do not store passwords, session cookies, API tokens, recovery codes, or authorization headers in workflow files.

Use environment variables, encrypted CI secrets, or a secret-management service.

## How can execution providers remain interchangeable?

Define a shared provider interface:

```ts
export type EnvironmentType = "browser" | "android";

export interface ExecutionEnvironment {
  readonly id: string;
  readonly type: EnvironmentType;

  connect(): Promise<void>;
  verify(): Promise<void>;
  disconnect(): Promise<void>;
}
```

A workflow runner can then use any provider that implements the interface:

```ts
import type { ExecutionEnvironment } from "../providers/provider";

export async function withEnvironment(
  environment: ExecutionEnvironment,
  task: () => Promise<void>
): Promise<void> {
  await environment.connect();

  try {
    await environment.verify();
    await task();
  } finally {
    await environment.disconnect();
  }
}
```

The task does not need to know whether the environment is a local Chromium process, a persistent browser profile, a physical Android device, or a cloud-hosted Android environment.

## How can a browser task be implemented?

Playwright can load an existing authenticated state for an approved test account.

```ts
import { chromium } from "@playwright/test";

async function inspectWebSession(): Promise<void> {
  const browser = await chromium.launch();

  try {
    const context = await browser.newContext({
      storageState: "runtime/auth/reviewer.json"
    });

    const page = await context.newPage();

    await page.goto("https://example.com/dashboard", {
      waitUntil: "domcontentloaded"
    });

    const heading = page.getByRole("heading").first();
    await heading.waitFor();

    console.log({
      title: await page.title(),
      heading: await heading.textContent(),
      url: page.url()
    });
  } finally {
    await browser.close();
  }
}

inspectWebSession().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Exclude authentication state from source control:

```gitignore
runtime/auth/
.env
test-results/
playwright-report/
```

Use a separate state file for each authorized account or role.

## How can Android availability be checked?

Before running a mobile workflow, confirm that the device is connected, the Android version is known, and the required application is installed.

Useful ADB commands include:

```bash
adb devices
adb -s DEVICE_SERIAL shell getprop ro.build.version.release
adb -s DEVICE_SERIAL shell getprop ro.product.manufacturer
adb -s DEVICE_SERIAL shell pm path com.example.application
```

A typed ADB wrapper can centralize command execution:

```ts
import { execFile } from "node:child_process";
import { promisify } from "node:util";

const execFileAsync = promisify(execFile);

export class AdbClient {
  constructor(private readonly serial: string) {}

  private async run(...args: string[]): Promise<string> {
    const { stdout } = await execFileAsync(
      "adb",
      ["-s", this.serial, ...args],
      { timeout: 15_000 }
    );

    return stdout.trim();
  }

  async getProperty(property: string): Promise<string> {
    return this.run("shell", "getprop", property);
  }

  async getAndroidVersion(): Promise<string> {
    return this.getProperty("ro.build.version.release");
  }

  async isApplicationInstalled(
    packageName: string
  ): Promise<boolean> {
    const output = await this.run(
      "shell",
      "pm",
      "path",
      packageName
    );

    return output.startsWith("package:");
  }
}
```

The workflow should stop before its main task when the device is offline or the required package is missing.

## How can the correct environment be selected?

Route workflows according to their declared execution type rather than relying only on the platform name.

```ts
type ExecutionType = "browser" | "android" | "hybrid";

interface WorkflowDefinition {
  platform: string;
  executionType: ExecutionType;
}

export function requiredEnvironments(
  workflow: WorkflowDefinition
): Array<"browser" | "android"> {
  switch (workflow.executionType) {
    case "browser":
      return ["browser"];

    case "android":
      return ["android"];

    case "hybrid":
      return ["browser", "android"];
  }
}
```

This avoids rigid assumptions.

Instagram and TikTok commonly need Android application workflows, while Reddit is commonly handled through a browser. Facebook and YouTube may use browser, mobile, or hybrid execution depending on the feature being tested.

## What information should be written to reports?

A workflow report can contain:

```json
{
  "workflowId": "instagram-application-health",
  "environmentType": "android",
  "status": "passed",
  "startedAt": "2026-07-21T08:30:00Z",
  "durationMs": 6420,
  "environment": {
    "androidVersion": "14",
    "applicationInstalled": true
  },
  "artifacts": [
    "artifacts/device-log.txt",
    "artifacts/screenshot.png"
  ]
}
```

Reports should not contain:

- Passwords
- Cookies
- API tokens
- Authorization headers
- Recovery codes
- Complete browser state files

Logs should help reproduce an error without exposing access to the account.

## How should browser and Android jobs run in CI?

Keep the jobs separate because they have different dependencies.

```yaml
name: Automation checks

on:
  pull_request:
  workflow_dispatch:

jobs:
  browser:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm

      - run: npm ci
      - run: npx playwright install --with-deps chromium
      - run: npm run test:browser

  android:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm

      - run: npm ci
      - run: adb devices
      - run: npm run test:android
```

The Android job still needs an approved device connection or an environment-provisioning step.

Do not report an Android test as passed when no Android environment was available. Mark it as skipped with a clear reason or fail the environment check.

## What are the responsible-use limits?

Use this toolkit only with systems and accounts you own or have permission to test or manage.

Do not adapt it for:

- Unsolicited messages
- Artificial engagement
- Private-data collection
- Circumvention of platform restrictions
- Publishing without required approval
- Credential storage in source control

Workflows that can publish content or modify account settings should include rate controls, approval steps, and audit logs.

## References

- [Playwright documentation](https://playwright.dev/docs/intro)
- [Selenium documentation](https://www.selenium.dev/documentation/)
- [Puppeteer documentation](https://pptr.dev/)
- [Android Debug Bridge documentation](https://developer.android.com/tools/adb)
- [Postman documentation](https://learning.postman.com/docs/introduction/overview/)
- [GitHub Actions documentation](https://docs.github.com/actions)
- [Docker documentation](https://docs.docker.com/)
