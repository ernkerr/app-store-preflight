# app-store-preflight

A [Claude Code](https://claude.com/claude-code) **skill** that answers "is this
iOS app ready to submit?" before anything is built or uploaded.

1. **Repo scan:** the common App Review rejections (broken purchases,
   placeholder content, subscription disclosures, privacy mismatches, spam /
   similar apps, missing account deletion) plus a quick security pass (secrets
   in the bundle or git, insecure endpoints), typecheck and tests.
2. **Simulator smoke test:** drives the app with Maestro, including a
   Subscribe tap that proves the subscription loads from App Store Connect, and
   watches the logs for errors and crashes.
3. **Your walkthrough:** leaves the app open for a 5-minute look at the small
   stuff automated checks miss, then fixes what you flag.

It ends with a verdict. If the app is ready, it hands off to
[app-store-connect-setup](https://github.com/ernkerr/app-store-connect-setup),
which captures and styles screenshots
([app-store-screenshots](https://github.com/ernkerr/app-store-screenshots)) and
submits. Each run's lessons go back into `references/rejection-checklist.md`.

## Install

```bash
git clone https://github.com/ernkerr/app-store-preflight ~/.claude/skills/app-store-preflight
```

Needs Xcode + the iOS Simulator, and [Maestro](https://maestro.mobile.dev)
(`brew install openjdk` + `curl -fsSL https://get.maestro.mobile.dev | bash`).

## Use

From the app's repo:

> is this app ready for submission?
