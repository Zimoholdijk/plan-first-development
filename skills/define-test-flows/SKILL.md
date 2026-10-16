---
name: define-test-flows
description: "Record the user journeys a product must never break, with the user doing the clicking, and turn each one into an end-to-end test that every code review re-runs. Walks the user through recording a flow with Playwright's recorder (codegen), tidies the recording without changing the path, adds the failure cases the user confirms by watching, and lists the flow in context/critical-flows.md. Use after a feature is implemented, when setting up a project, or any time the user wants to add or redo a critical flow."
disable-model-invocation: false
argument-hint: "[optional: the flow to record, e.g. 'reassigning a task' or 'signing up']"
---

# Define Test Flows

You are helping the user record a critical flow: $ARGUMENTS (if empty, ask which flow).

A critical flow is a journey the product depends on, such as signing up, creating a task, or reassigning one. The user records it by clicking through the real app, because the user knows which path real people take. Your job is to turn that recording into a reliable test, add the failure cases the user confirms, and list the flow so every code review re-runs it.

## Rules

1. **The user's clicks are the source of truth.** Never invent, reorder, or shortcut the recorded path. If the path looks wrong, ask; if it is wrong, re-record it.
2. **Encode what should happen, not just what did.** A recording captures whatever the app did at that moment, bugs included. Before the test is final, the user confirms what they should see at each key point and at the end.
3. **One flow at a time.** Record, review, tidy, add failure cases, and register one flow before starting the next.
4. **Every added variant is watched.** A failure case you draft is only kept after the user watches it run in a visible browser and says it is right.
5. **Deterministic tests only.** No fixed sleeps, no reliance on data left by another test, no real third-party services. If a step cannot be made reliable, say so instead of keeping a flaky test.
6. **Plain language with the user.** Describe steps as a person would ("clicks New task, types a title, presses Save"), not as code.

## Step 1: Get oriented

Read, without dumping it on the user:

1. `.claude/CLAUDE.md`, especially any **Testing Reality** section, and `context/overview.md`.
2. `context/critical-flows.md` if it exists: the flows already recorded, so you don't record a duplicate.
3. The project's end-to-end setup: `playwright.config.*`, an `e2e/` or `tests/` folder, and how the suite is run.

If the project has a browser UI but no Playwright setup, tell the user what you propose to add (a `playwright.config` with the base URL, and an `e2e/flows/` folder) and add it only after they agree.

If the project has no browser UI (a command-line tool, a library, an API), a flow is a sequence of commands or calls. The user runs them in their own terminal and pastes each command with its output; you write the test in the project's own test runner. Steps 4 to 8 apply the same way.

## Step 2: Choose the flow

Ask, one question at a time:

1. **Which flow?** In one sentence, what does the user accomplish? ("A member reassigns a task and the new assignee sees it.")
2. **Where does it start?** The page or URL the user begins on.
3. **Does it need a signed-in user?** If yes, which kind of account. Tests use a dedicated test account, never a real person's.

Confirm the app is running and reachable at its base URL. If the project's CLAUDE.md says the user manages dev servers, do not start one; otherwise ask once before starting it.

## Step 3: The user records it

Recording happens on the user's own machine, because Playwright's recorder opens a real browser window. Give the exact commands, filled in for this project:

- **If the flow needs a signed-in user,** record the sign-in once and save it:
  `npx playwright codegen --save-storage=e2e/.auth/<account>.json <base-url>`
  The user signs in with the test account, then closes the window. Make sure `e2e/.auth/` is in `.gitignore`.
- **Record the flow:**
  `npx playwright codegen -o e2e/flows/<flow-slug>.spec.ts [--load-storage=e2e/.auth/<account>.json] <base-url><start-path>`
  For a project whose tests are not TypeScript, add the matching `--target` (for example `--target=python-pytest`).

Tell the user how to record well:

- Click through exactly as a real user would, at a normal pace.
- At each key moment, use the recorder's toolbar to mark what should be on screen: a message appears, a list shows the new item, a field holds a value.
- If they click something by mistake, close the window and record again rather than editing the file.
- Close the browser window when the flow is done; the file is then saved.

If this session cannot reach the user's machine (a cloud session), the user records locally and commits or pastes the file.

## Step 4: Review the recording with the user

Read the recorded file and describe it back as numbered plain steps:

> 1. Opens the task list.
> 2. Clicks "New task", types "Draft report", presses Save.
> 3. Sees "Draft report" at the top of the list.

Then ask:

1. **Is this the path a real user takes?** If not, re-record.
2. **What should the user see at the end?** Add a check for it if the recording has none.
3. **Did anything happen during recording that is actually wrong?** If the app misbehaved, write the check for what *should* happen, tell the user the test will fail until the bug is fixed, and record the bug in the feature's `progress.md` or the project's Deferred Items.

## Step 5: Tidy the recording without changing the path

Keep every step in the same order, on the same targets. Change only how it is written:

- Replace fragile element references (CSS classes, nth-child, generated IDs) with sturdy ones: `getByRole`, `getByLabel`, `getByText`.
- Replace typed test data that would collide on a second run with unique values, and add setup and cleanup so the test runs in any order and any number of times.
- Use the saved sign-in (`storageState`) instead of replaying the login steps, unless signing in is the flow itself.
- Remove fixed waits; rely on Playwright's auto-waiting checks (`toBeVisible`, `toHaveURL`, `toHaveText`).
- Name the test after what it proves ("member reassigns a task and the new assignee sees it"), not how.

Tell the user in one or two sentences what changed. If a step cannot be made reliable without changing the path, ask before changing it.

## Step 6: Add the failure cases the user confirms

Ask: **what should happen when this goes wrong?** Offer two or three likely cases for this flow, such as a required field left empty, a signed-out user, or someone without permission. For each case the user wants:

1. Draft the variant as its own test, starting from the recorded path.
2. Run it in a visible browser (`npx playwright test e2e/flows/<flow-slug>.spec.ts --headed -g "<test name>"`) while the user watches.
3. Keep it only if the user says it shows the right behavior. Otherwise fix the expectation with them or drop the case.

## Step 7: Run the flow

Run the flow's file headless (`npx playwright test e2e/flows/<flow-slug>.spec.ts`) and show the exact command and result. Everything passes except checks the user agreed in Step 4 will fail until a known bug is fixed.

## Step 8: Register the flow

Add one row to `context/critical-flows.md`, creating the file with this header if it does not exist:

```markdown
# Critical flows

Journeys this product must never break. Each was recorded by a person clicking through the app (`/define-test-flows`), and every code review re-runs these tests.

| Flow | What it proves | Test file | Added |
|------|----------------|-----------|-------|
```

The row: the flow's short name, one plain sentence on what it proves (including its failure cases), the test file path, and today's date.

## Step 9: Wrap up

Tell the user, briefly:

- **Done:** the flow recorded, its failure cases, and the test result.
- **Decided for you:** anything you chose without asking, such as a tidy-up that needed judgment.
- **Next:** another flow to record, or nothing.
