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

## Found the hard way (add to this list)

- GLP-1 Anchor (Aug 2026): purchases failed because the IAP init called a
  method that doesn't exist in the installed plugin version, and a try/catch
  swallowed the error (2.1(a)/(b)). Placeholder icon (2.3.8).
- Farkle (Sep 2026): the paywall's fallback price read the same as the real
  price, so the label proved nothing. Only the Subscribe-tap check showed
  the product loaded.
