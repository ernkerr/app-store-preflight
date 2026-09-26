# Rejection checklist

The common App Review rejections for small indie apps, each with how to
detect it in an Expo / React Native or Capacitor repo. Report only what you
can point at in this repo. Guideline numbers are Apple's App Review
Guidelines.

## Broken or unfinished (2.1 App Completeness)

- **Purchases that don't work.** This is the most expensive one (GLP-1 was
  rejected twice for it). Read the IAP code path end to end: connection
  init, product fetch, purchase, finish transaction, restore. Red flags:
  - a `try/catch` that swallows an error before products load
  - a call to an API that doesn't exist in the installed library version
    (check the package's own types in `node_modules`)
  - a purchase that silently returns when no product loaded
  - no Restore button
  Then run the Subscribe-tap check in the simulator (SKILL.md Step 2).
- **Placeholder content:**
  `grep -rniE "lorem|todo|fixme|coming soon|placeholder|test user|asdf|xxx" app src --include=*.tsx --include=*.ts`
  (ignore code comments). Also look for demo/seed data shipping as real content.
- **Dead ends:** buttons with empty `onPress`, links to `example.com` or
  empty URLs (`grep -rn 'href=""\|url: ""\|: ""' src/config`), settings rows
  that do nothing.
- **iPad, even for iPhone-only apps:** Apple often reviews iPhone-only
  apps on an iPad (compatibility mode). The purchase flow must work there
  too, so run the Subscribe-tap check on an iPad simulator as well.
- **iPad:** if `supportsTablet` is true, Apple reviews on iPad. Run the
  smoke test on an iPad simulator too. Stretched layouts, clipped sheets and
  unreachable buttons get rejected. If iPad isn't worth supporting, set
  `supportsTablet: false` instead.
- **Crashes on launch or first run:** the smoke test from a clean install
  (`clearState: true`) covers it.

## Metadata (2.3)

- **2.3.8 placeholder icon:** open `assets/images/icon.png` (or the
  AppIcon set) and look at it. Template or default icons are rejected.
- **2.3.7 keywords/subtitle:** no competitor or trademarked names, no price
  words ("free"), and no repeating the app name. Check the listing doc.
- **2.3.3 screenshots** must show the app in use, not just a splash or
  marketing art alone.
- **Home-screen name vs store name:** if they differ (a store name was taken),
  say so in the App Review notes.

## Payments (3.1)

- **3.1.1:** digital unlocks must use In-App Purchase. No links or buttons to
  pay elsewhere (`grep -rniE "stripe|paypal|checkout|buy on (our )?web"`).
- **3.1.2 subscriptions:** the paywall must show:
  - the subscription title
  - its length
  - its price, and the price per period if there's a trial
  - that it auto-renews and how to cancel
  - working Privacy Policy and Terms of Use (EULA) links
  - a Restore Purchases button
  The listing description also needs the Privacy and Terms links. Read the
  paywall component and check each item.
- The price shown should come from StoreKit (`localizedPrice`), with the
  config value only as a fallback.

## Spam and minimum functionality (4.x)

- **4.3(a) spam / similar apps:** Apple rejects near-duplicate apps from one
  developer. That's a real risk for a family of score trackers built from one
  template. Make sure this app has game-specific features beyond the
  boilerplate (its own scoring engine, calculator, rules, stats), and that
  its name, icon, screenshots and description don't read like a reskin. Put
  what's specific to this app in the review notes.
- **4.2 minimum functionality:** a thin web wrapper (Capacitor apps) needs to
  feel native. It should work offline, use native navigation, and not look
  like a website.
- **4.8 login services:** if it offers Google/Facebook sign-in, it must also
  offer Sign in with Apple.

## Intellectual property (5.2)

- **5.2.1 third-party content:** lyrics, sheet music, movie posters, logos,
  brand names, or API data the app doesn't have documented rights to. Look for
  bundled content files and for APIs that return copyrighted material. Either
  attach proof of rights in App Review Information, or remove the content.
  Apple repeats this rejection until one of those happens.

## Privacy (5.1)

- **Privacy policy:** a live URL, linked in the app (paywall/settings) and in
  App Store Connect. `curl` it for a 200 and check it names this app.
- **Data collection answers must match the code.** List every SDK in
  `package.json` that phones home (analytics, crash reporting, ads, auth,
  backend, feedback forms like Web3Forms). Each one means the App Privacy
  answers can't be "Data Not Collected". Flag any mismatch with the listing
  or privacy policy.
- **Permission strings:** every permission the app requests needs a specific
  `NS*UsageDescription`. Unused permissions added by Expo plugins (camera,
  photos, location) should be removed. Check `app.json` → `ios.infoPlist` and
  plugin configs.
- **5.1.1(v):** apps with account creation must offer in-app account
  deletion.
- **5.1.2 tracking:** any tracking SDK needs App Tracking Transparency and the
  matching privacy answers.

## Data sources and licenses

- **Third-party data terms vs. a paid app.** Not an App Review rejection, but
  a launch blocker all the same. Read the terms of every data API the app
  calls. Many free movie/book/music APIs forbid commercial use, and a
  subscription or paid unlock is commercial. Also check per-key request
  quotas: a key shipped in the bundle is shared by every user.

## Content and safety

- **Health/medical apps (1.4.1):** include a visible medical disclaimer, make
  no diagnosis or dosing advice beyond what the user enters, and answer the
  regulated-medical-device question honestly (a tracker is "No").
- **Age rating** must match what's in the app (user-generated content, web
  views, mature themes).

## Build and binary

- `ITSAppUsesNonExemptEncryption` is set (false for HTTPS-only apps).
- The version is higher than any version Apple already approved.
- No debug-only UI or dev menus reachable in Release.
- `EXPO_PUBLIC_*` values the Release build needs are set in the build
  environment (EAS env), or the binary ships with fallback data.
- OTA updates (expo-updates) are fine for JS fixes. Don't use them to change
  what the app does after review (2.5.2).

- **Paywall perks must exist.** Check each perk bullet against the code. A
  data-source switch can quietly remove a paid feature (e.g. a streaming
  filter that needs provider data the new API doesn't have) (2.3.1, 3.1.2).
- **In-app unlock codes.** Hardcoded promo codes that unlock paid features
  bypass IAP (3.1.1), and anyone can read them out of the bundle. Use App
  Store Connect offer codes.
- **Developer hints in Release.** Strings like "API key not configured" or
  "demo catalog" read as unfinished (2.1). Gate them behind `__DEV__`.

## Found the hard way (add to this list)

- GLP-1 Anchor (Aug 2026): purchases failed because the IAP init called a
  method that doesn't exist in the installed plugin version, and a try/catch
  swallowed the error (2.1(a)/(b)). Placeholder icon (2.3.8).
- A karaoke app (Feb 2026): rejected for third-party lyrics (5.2.1), and
  for missing EULA and privacy links both in the app and in the description
  (3.1.2).
- GLP-1 Anchor (Aug 24, 2026): build 4 was still rejected under 2.1(b) ("we
  got one error message" on buy), reviewed on an iPad Air running an
  iPhone-only app.
- Farkle (Sep 2026): the paywall's fallback price read the same as the real
  price, so the label proved nothing. Only the Subscribe-tap check showed
  the product loaded.
- Movie tracker (Sep 2026): the smoke test found a modal whose Save button sat
  under the keyboard (the tap landed on a key, so nothing saved), and a
  cancelled Apple sign-in sheet reported as "Purchase failed". After the
  Subscribe-tap check, tap Cancel and confirm no error alert appears. Always
  type into every modal and tap Save with the keyboard still up.
- GLP-1 Anchor (Sep 2026, root cause of the Aug 24 rejection): with
  cordova-plugin-purchase 13.x on Apple, `store.owned()` returns true for the
  INITIATED placeholder transaction created when the payment sheet opens, so
  the paywall unlocked before payment. Gate entitlement on APPROVED/FINISHED
  transactions. In the Subscribe-tap check, confirm the paywall is still
  behind Apple's sheet, not the unlocked app.
- Capacitor apps (Sep 2026): without `viewport-fit=cover` in the viewport
  meta, every `env(safe-area-inset-*)` is 0 and headers sit under the
  Dynamic Island. Text glyphs used as icons (〜 ◎) can render as '?' boxes.
- Maestro can't see into an iPhone-only app's compatibility window on an
  iPadOS 26 simulator. For the iPad smoke test, build a sim-only copy with
  `TARGETED_DEVICE_FAMILY=1,2` on the xcodebuild command line (don't commit).
  Simulator builds have no App Store receipt, so a sign-in prompt at launch
  is expected there.
