---
name: app-store-preflight
description: >-
  Check whether an iOS app is ready to submit to the App Store, before anything
  is built or uploaded. Scans the repo for things Apple rejects (broken
  purchases, placeholder content, missing subscription disclosures, privacy
  mismatches, missing account deletion, thin functionality) and for security
  problems (secrets in the bundle or git), runs the typecheck and tests,
  smoke-tests the app in the iOS Simulator with Maestro, then hands the
  simulator to the user for a short walkthrough of the small stuff (wording,
  empty states, symbols) and fixes what they find. Ends with a verdict and,
  if ready, hands off to app-store-connect-setup. USE THIS whenever the user
  asks "is this app ready for submission?", "is it ready for the App Store?",
  "preflight", "pre-submission check", "will Apple reject this?", "check the
  app before we submit", or before any new version or update is submitted.
---

# App Store preflight

Catches, before the build, what App Review or the user would otherwise catch
after it. Everything found here is a cheap code fix. Found after upload, it
costs a new build, and found by Apple, it costs 1–2 days per rejection.

**Where it sits:**
`app-store-preflight` (this skill) → verdict → `app-store-connect-setup`
(which captures screenshots and styles them with `app-store-screenshots`, then
fills in App Store Connect). The submission skill runs this one first unless
a passing record exists for the current commit (Step 6).

## Ground rules

- **Report, then fix with consent.** Scan and smoke-test on your own. Fix
  only what the user approves in the Step 4 question round. Obvious
  one-liners the user can see in the report (a typo, a missing link) can be
  batched into that approval.
- **Evidence for every finding:** file:line, the guideline number, and why it
  would fail. No generic advice. If it isn't in this repo, it isn't a
  finding.
- **Don't print secrets.** If you find a key, report its file:line and type,
  masked (`sk-…a1b2`).
- **Run from the app repo** and follow its CLAUDE.md (typecheck/test/commit
  rules).

## Step 1 — Repo scan (automated)

Work through `references/rejection-checklist.md`. It lists each common
rejection with how to detect it in an Expo/React Native or Capacitor repo.
Also:

- **Build health:** the repo's typecheck and test commands (e.g. `npm run
  typecheck`, `npm test`). A failure is a blocker.
- **Security quick pass:**
  - Secrets baked into the app: every `EXPO_PUBLIC_*` / `VITE_*` value ships
    inside the binary. Fine for public API keys meant for clients, a blocker
    for anything secret (service-role keys, private API tokens, webhook
    secrets).
  - Secrets committed to git: `git log -p -S` for common key prefixes (`sk_`,
    `sk-`, `AKIA`, `-----BEGIN`, `service_role`), plus `.env*` files that
    aren't gitignored.
  - `http://` endpoints (ATS will block them unless exempted) and any ATS
    `NSAllowsArbitraryLoads`.
  - For a deeper audit, suggest gstack `/cso`. Don't run it unasked.
- **Known repo traps:** read the repo's CLAUDE.md "do not regress" notes and
  check each one still holds (e.g. pinned dependency versions, patches
  applied).

## Step 2 — Simulator smoke test (automated)

1. Build a **Release** simulator build (dev builds add launcher and LogBox
   noise): `npx expo run:ios --configuration Release --device <UDID>` for Expo;
   Capacitor: `npm run build && npx cap sync ios`, then build the App scheme
   for the simulator.
2. Drive the main flow with Maestro (setup and label tips:
   `app-store-connect-setup/references/maestro-capture.md`). If the repo has
   `.maestro/screenshots.yaml`, run it. It already covers the main flow. Also
   do the **Subscribe-tap check**: Apple's sign-in prompt = product loaded,
   and "Purchase failed" = blocker. Tap Cancel; never enter credentials.
3. Capture the app's log during the run
   (`xcrun simctl spawn <UDID> log show --start <t> --predicate 'process ==
   "<App>"'`). Red flags: JS errors, `console.warn` from IAP/network code,
   crash reports in `~/Library/Logs/DiagnosticReports/<App>-*.ips`. A crash
   inside `XCTAutomationSupport` is Maestro's runner, not the app.
4. Look at every screenshot the flow produced. Look for clipped text, overlapping
   elements, placeholder strings, raw numbers where the design uses a symbol.

## Step 3 — The user's walkthrough (5–10 minutes)

Leave the app open in the simulator on a realistic state (the smoke test's
end state works: a game in progress). Bring the Simulator to the front.
Then ask the user to tap around, with a short list of what to look for. Keep
it to ~6 items tailored to this app, like:

- wording that reads wrong or sounds machine-written
- empty states (no games yet, no players)
- symbols vs numbers (a bust shown as 0 instead of a skull)
- anything cut off, misaligned, or hard to tap
- the paywall: does the price and wording feel right?
- anything you'd expect to be there that isn't

Collect their notes in their words. Don't argue taste. If a note is
ambiguous, ask one question. If they have nothing, that's a valid result.

This happens in the same sitting as any other questions, so the user can
walk away afterwards. If they're already gone, skip it and mark it as
"not done" in the report rather than inventing their verdict.

## Step 4 — Report and approve fixes (one question round)

Show a short report:

| Severity | Meaning |
|---|---|
| **Blocker** | Apple will reject, or it's broken/insecure. Fix before the build. |
| **Risk** | Could be rejected depending on the reviewer. Recommend fixing. |
| **Polish** | The user's walkthrough notes and small UX issues. |

Each line: what, where (file:line or screen), the guideline, and the proposed
fix. Then one AskUserQuestion: which to fix now (default: all blockers and
risks, plus the polish they raised). Anything that publishes something or
changes pricing gets named explicitly.

## Step 5 — Fix, re-verify

Make the approved fixes. Re-run typecheck/tests, and re-run the Maestro flow if
UI changed. Show the user before/after screenshots of any visual fix. Commit
per the repo's rules, with messages that describe the actual change.

## Step 6 — Verdict and hand-off

Write the record the submission skill checks:

```bash
mkdir -p .preflight && cat > .preflight/last-run.json <<EOF
{"commit": "$(git rev-parse HEAD)", "date": "$(date -u +%FT%TZ)", "verdict": "ready|not-ready",
 "blockers_open": 0, "walkthrough": "done|skipped"}
EOF
```

Commit it with the fixes (it's small and useful history). Then:
- **Ready:** say so in one line with what was fixed, and ask "Submit now?".
  On yes, invoke `app-store-connect-setup`.
- **Not ready:** list the open blockers and stop.

## Step 7 — Learn

Same rule as the submission skill: this skill lives in a git repo, and chat
transcripts are deleted after 30 days. When a run finds something the
checklist didn't cover, or Apple later rejects for something preflight
passed, add it to `references/rejection-checklist.md` (generic wording, no
personal IDs), then commit and push the skill repo. Tell the user in one line.
