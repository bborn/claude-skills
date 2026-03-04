---
name: qa-seed-verify
description: Create QA seed scripts and verify them end-to-end after finishing a feature or bug fix. Generates idempotent seed data, runs it, verifies via browser automation, and produces copy-paste PR QA steps. Use this skill when finishing a feature, preparing a PR for review, writing QA steps, creating test data seeds, or when someone says "QA this", "write QA steps", "create a seed script", or "how do I test this PR". Also useful when a PR has no QA instructions or when a reviewer asks "how do I test this?"
---

# QA Seed & Verification

After finishing a feature or bug fix, create a working QA seed script, verify it end-to-end with browser automation, and produce concrete PR QA steps.

<preferences>
* A broken or placeholder seed script is worse than no seed script. If you write one, it must run without errors.
* QA steps must be specific enough that someone unfamiliar with the change can follow them in under 5 minutes.
* Login/auth is often the biggest QA friction point. Address it explicitly — don't assume the reviewer knows how to get past auth.
* I'd rather you ask me a clarifying question than guess wrong about the app setup.
</preferences>

<seed-philosophy>
The seed script should do as much setup work as possible so the QA tester only has to verify the specific feature or fix — nothing else.

**Principle: Seed to the edge of what you're testing.**

If you're testing a checkout flow, the seed should create a fully published product with pricing, a buyer account, and everything needed so the tester can go straight to the checkout page. Don't make the tester manually create a product, set pricing, and publish it just to get to the thing they're actually testing.

The seed eliminates friction. The only manual steps should be the ones that exercise the actual change being tested.

If it can be scripted into the seed, it should be — unless that exact step IS what's being tested.
</seed-philosophy>

<step name="Gather Context">
Before writing anything, determine:

1. **Ticket/branch ID** — from branch name, recent commits, or ask the user.
2. **What changed** — read the diff or recent commits to understand what needs QA.
3. **Check for project QA knowledge** — look for a `QA.md` file in the project root or `docs/` directory. If one exists, read it first — it contains project-specific QA patterns, auth shortcuts, common setup steps, and lessons learned from previous QA sessions. Follow its guidance.
4. **App stack** — check the project for framework, language, database. Read any existing QA docs, seed conventions, or test helpers.
5. **Existing QA patterns** — look for existing QA seed scripts, seed helpers, or seed conventions. Follow them.
6. **Auth/login mechanism** — figure out how users log in. Check for QA bypasses, dev-mode shortcuts, or test credentials. If login requires email/SMS codes, find or propose a bypass for local QA.

If any of this is unclear, **ask the user** before proceeding. Do not guess.
</step>

<step name="Determine App Port">
The QA flow requires a running app.

1. Check for a running dev server (look for listening ports).
2. If not running, ask the user if they want you to start it. Determine the correct start command from the project.
3. Confirm the base URL with the user before proceeding.
</step>

<step name="Write the QA Seed Script">
Create a seed script following the project's conventions. Before writing ANY seed code:

1. **Read the relevant models/schemas** — verify field names, required attributes, valid values, and relationships from the actual source code.
2. **Read existing seed patterns** — follow the project's established conventions.
3. **Use existing helpers** — don't reinvent what the project already provides.
4. **Identify the testing boundary** — what is the feature/fix being tested? Everything BEFORE that boundary should be handled by the seed. Only the feature/fix itself should require manual interaction.

<seed-requirements>
- **Idempotent** — safe to run multiple times without duplicating data
- **Standalone-runnable** — executable without loading the full seed suite
- **Prints clear output** — what was created/found, credentials, and URLs to visit
- **Comment block** at the top describing what it sets up and why
- **Maximally complete** — set up all prerequisite state so the tester goes straight to the feature being tested
</seed-requirements>
</step>

<step name="Run the Seed Script">
Execute the seed script and **verify it succeeds**.

- If it fails, read the error, fix the script, and re-run.
- Do not move on until it runs cleanly.
- Common issues: wrong field names, missing required associations, invalid enum values, uniqueness violations.

**Do not skip this step.** A seed script that hasn't been run is not a seed script.
</step>

<step name="Browser Verification">
Write a standalone Playwright test script that verifies the QA flow end-to-end. This is a single executable file — not interactive MCP calls.

<browser-approach>
**Why a script instead of interactive browser use:** Interactive Playwright MCP tools require a permission prompt for every action (navigate, click, fill, screenshot). A Playwright test file runs as a single Bash command — one permission prompt for the whole flow.

**Write as a proper Playwright test file** (`verify.spec.mjs`) using `test()` and `expect()` — NOT a raw Node script. This lets you use Playwright's built-in HTML reporter instead of generating a custom report page.

**The test file should:**
1. Use `test.describe` for the ticket, with individual `test()` blocks for each verification step
2. Enable **video recording** and **screenshot on failure** via `test.use()`
3. Log in using the QA auth bypass (from Gather Context)
4. Navigate to the feature/page being tested
5. Perform the verification steps with `expect()` assertions
6. Attach screenshots at key points using `test.info().attach()`

**Example structure:**

```javascript
import { test, expect } from '@playwright/test';

const BASE_URL = 'http://localhost:3000';
const CHECKOUT_PATH = '/stacks/abc123/steps/def456';

test.use({
  viewport: { width: 1440, height: 900 },
  video: 'on',
  screenshot: 'on',
  headless: false,
});

test.describe('OL-XXXX: Feature description', () => {
  test('page loads correctly', async ({ page }) => {
    await page.goto(`${BASE_URL}${CHECKOUT_PATH}`, { waitUntil: 'domcontentloaded' });
    await expect(page.locator('.some-element')).toBeVisible();
  });

  test('validation shows contextual errors', async ({ page }) => {
    await page.goto(`${BASE_URL}${CHECKOUT_PATH}`, { waitUntil: 'domcontentloaded' });
    // ... test steps with expect() assertions
  });
});
```

**Running and viewing the report:**

```bash
# Run tests with HTML reporter — outputs to tmp/qa-reports/<ticket-id>/
npx playwright test tmp/qa-reports/<ticket-id>/verify.spec.mjs \
  --reporter=html \
  --output=tmp/qa-reports/<ticket-id>/test-results

# Open the interactive HTML report (shows tests, screenshots, video, traces)
npx playwright show-report
```

Playwright's HTML reporter provides an interactive UI with:
- Test status (pass/fail) per test
- Step-by-step execution with timing
- Embedded screenshots and video playback
- Expandable error details and traces
- Filtering and search

**No need to generate a custom `index.html`** — the built-in reporter is better.

**If the tests fail**, debug them — read the error output, fix the test, and re-run. The same standard applies as the seed: don't ship a broken verification script.
</browser-approach>

<qa-report-directory>
Save the test file and let Playwright manage the report output:

```
tmp/qa-reports/<ticket-id>/
├── verify.spec.mjs          # The Playwright test file
└── test-results/             # Playwright's output (screenshots, videos, traces)
```

Make sure `tmp/` is in `.gitignore` (add it if not).

After running the tests, open the report and the test-results folder so the user can drag-drop screenshots/video into the PR comment:

```bash
npx playwright show-report
open tmp/qa-reports/<ticket-id>/test-results/
```

GitHub supports `.webm` uploads via drag-and-drop on PR comments — no conversion needed. The video gives reviewers a quick walkthrough of the full QA flow, while screenshots are useful for inline PR references.
</qa-report-directory>
</step>

<step name="Write PR QA Steps">
Write specific, concrete QA steps for the PR description. Every step must be actionable by someone who has never seen the code.

<bad-example>
Run the seed script and test the feature.
</bad-example>

<good-example>
1. Start the app: `<exact start command>`
2. Run the seed: `<exact seed command>`
3. Go to `<exact URL>`
4. Log in as `<exact email>` using `<exact auth method>`
5. Navigate to `<exact page>`
6. Click `<exact element>`
7. Verify: `<exact expected outcome>`
</good-example>

Every QA step MUST include:
- **Exact commands** to run (start app, run seed)
- **Exact URL** to visit
- **Exact credentials** and login method
- **Exact actions** to perform (click what, fill in what)
- **Exact expected outcome** (what should be visible, what should change)
</step>

<step name="Present to User">
Show the user:
1. The seed script (summarized — they can read the file)
2. Proof the seed ran successfully (output)
3. The HTML report (should already be open in browser from the previous step)
4. A **ready-to-paste PR comment** — markdown-formatted QA steps that the user can copy into a GitHub PR comment. The comment should be complete text with no image/video placeholders — the user will drag-and-drop screenshots and the video from the QA report directory into the comment themselves.

<pr-comment-format>
The generated PR comment should look like:

```markdown
## QA Verification

**Seed:** `<exact command to run the seed>`
**Tested on:** `<base URL>`
**Logged in as:** `<user email>`

### Steps verified:
1. <step description> — <PASS/FAIL>
2. <step description> — <PASS/FAIL>
3. <step description> — <PASS/FAIL>

### To reproduce manually:
1. `<exact start command>`
2. `<exact seed command>`
3. Go to `<exact URL>`
4. Log in as `<exact email>` using `<exact auth method>`
5. <exact action>
6. Verify: <exact expected outcome>

Screenshots and video attached below.
```

The user will paste this into the PR comment and then drag-drop the screenshots and video at the bottom.
</pr-comment-format>

5. Ask if they want to adjust anything before finalizing
</step>

<step name="Update Project QA Knowledge">
After completing the QA flow, review what you learned and ask the user if any of it should be saved to the project's `QA.md` file for future sessions.

**Things worth saving:**
- Auth/login shortcuts that work for this project (bypass codes, header tricks, dev-only routes)
- Seed conventions discovered (helper methods, required setup patterns, common pitfalls)
- Frequently needed prerequisite state (e.g., "most QA scenarios need a published product — always seed one")
- Gotchas that caused seed failures (e.g., "User requires a confirmed email, not just an email address")
- App startup requirements (env vars, services that must be running)
- Common QA flows and how to navigate to them

**How to ask:** After presenting the QA results, say something like:
> "I learned some things about QA-ing this project during this session. Want me to save any of these to `QA.md` so future sessions can skip the discovery phase?"
Then list the specific things you'd propose saving.

**File format:** If `QA.md` doesn't exist yet, create it in the project root (or wherever the project keeps docs — follow existing conventions). Keep it concise and scannable — this is a reference for future QA sessions, not documentation. Use sections like `## Auth`, `## Seed Conventions`, `## Common Pitfalls`, `## Frequently Needed State`.

**If `QA.md` already exists**, propose additions or updates — don't overwrite what's already there. Ask before changing existing content.
</step>
