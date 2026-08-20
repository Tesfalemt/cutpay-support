# CutPay — Google Play release checklist

The support and privacy pages in this repo are the two URLs the Play listing
points at. This file records what the Play Console declarations must say so
they stay consistent with `privacy.html`. If the policy changes, change this
too.

Verify each item in the Console before submitting — Play's forms and wording
drift over time, so treat this as the intent, not a transcript of the UI.

## Where the app itself lives

The app is **`Tesfalemt/client-tracker`** — the repo name predates the rename
to CutPay. Identifiers as they stand:

| Field | Value |
| --- | --- |
| Package name | `com.cutpay.app` |
| Display name | CutPay |
| Version / versionCode | `0.1.0` / `1`, both in `app.json` |

The package name is **permanent from the first publish**. Change it before
that upload or never.

## URLs the listing needs

| Field | Value |
| --- | --- |
| Privacy policy | `https://tesfalemt.github.io/cutpay-support/privacy.html` |
| Support / website | `https://tesfalemt.github.io/cutpay-support/` |
| Support email | `uaeutemie5@icloud.com` |

Both must load publicly, over HTTPS, with no login and no redirect to a
sign-in wall. GitHub Pages is enabled and deploying from `main`, so these go
live when a change merges — a privacy policy URL that 404s is an automatic
rejection.

## Data safety form

The policy states CutPay collects nothing. The form must say the same thing,
or the mismatch is grounds for rejection.

- **Does your app collect or share any of the required user data types?** No.
- **Data collected:** none.
- **Data shared:** none.
- **Encrypted in transit:** not applicable — nothing is transmitted.
- **Users can request data deletion:** not applicable — no account, no
  server-side data. Uninstalling removes everything.

The client names, phone numbers and amounts the barber types in are **not**
"collected" in Play's sense: collection means transmitted off the device.
Nothing here leaves the device except by an explicit user action (share sheet,
messaging app, dialer), which Play treats as user-initiated, not collection.

## Declarations that are easy to get wrong

- **Financial features — declare none.** CutPay is a written record. It does
  not move money, connect to a bank, or process payments. Do not tick any
  financial-features box on the strength of the name or the word "pay";
  declaring them pulls the app into a compliance regime it does not belong in
  and will stall the review.
- **App access.** There is no login, so declare that all functionality is
  available without special access. Reviewers need to be able to open the app
  and use it cold.
- **Ads.** No ads, no advertising ID.
- **Target audience.** Adults — it is a tool for working barbers. Not
  directed at children, which matches the Children section of the policy.
- **Content rating.** Complete the IARC questionnaire honestly; a utility
  with no user-generated content, no ads and no purchases rates at the lowest
  tier.

## Android build notes

- **Do not request `SEND_SMS`.** Reminders must open the user's messaging app
  via an intent with the text pre-filled, exactly as the policy describes.
  Holding the SMS permission requires a Permissions Declaration and is
  routinely refused for apps that only need to hand off a draft. The intent
  approach needs no permission at all.
- Same for contacts, camera, microphone and location — the policy states the
  app touches none of them, so the manifest must not request them.
- Meet Play's current target API level requirement for new releases. Verified:
  the build targets API 36 — see *Verified against the build*.
- Use Play App Signing and keep the upload key backed up somewhere you will
  still have it in two years.

## Permissions the build actually requests

Know this before the listing goes up, because the permission list is public and
a reader will compare it against a policy that says there is no server.

**Read this from a real merged manifest, not from `node_modules`.** An earlier
version of this file audited the dependency tree and concluded `expo-file-system`
was the only source of permissions. That was wrong, and it was wrong in the
direction that matters: the Expo bare template writes two more into
`android/app/src/main/AndroidManifest.xml`, which is not a dependency and so
never showed up in that audit. To see the truth:

```bash
npx expo prebuild --platform android --no-install
grep uses-permission android/app/src/main/AndroidManifest.xml
rm -rf android          # generated; EAS runs its own prebuild
```

As of the current build that yields:

| Permission | Source | Ships? |
| --- | --- | --- |
| `INTERNET` | `expo-file-system` | **Yes.** CutPay's own code makes no network calls. |
| `WRITE_EXTERNAL_STORAGE` | `expo-file-system` | Yes, capped `maxSdkVersion="32"` — ignored on Android 13+. |
| `READ_EXTERNAL_STORAGE` | `expo-file-system` | Same cap. |
| `SYSTEM_ALERT_WINDOW` | Expo bare template | **Blocked** — see below. |
| `VIBRATE` | Expo bare template | **Blocked** — see below. |

`SYSTEM_ALERT_WINDOW` is the one that would have caused trouble. It shows on the
listing as **"Display over other apps"** — a strange thing for an offline ledger
to want, and squarely at odds with a privacy policy whose whole claim is that
the app touches nothing. Nothing in the codebase draws an overlay or vibrates;
neither permission has a single caller. Both are now removed via
`android.blockedPermissions` in `app.json`, which emits `tools:node="remove"` so
the manifest merger strips them from every variant.

One consequence to know: that also removes `SYSTEM_ALERT_WINDOW` from **debug**
builds, because a remove in the main manifest beats the debug source set's own
declaration. Modern React Native renders its dev menu in-app rather than through
an overlay, so this should be invisible — but it is the first thing to check if
the dev menu ever misbehaves.

React Native's own `main` manifest declares no permissions at all, and confines
its `SYSTEM_ALERT_WINDOW` to the debug source set, where it belongs. `expo-sharing`
declares none either — it ships a `SharingFileProvider` in its manifest, merged by
autolinking, so file sharing needs no config-plugin entry.

Do **not** add `expo-file-system` to `plugins` in `app.json`. Its config plugin
adds the storage permissions *uncapped*, which is strictly worse. The module
works without it.

**The policy stays accurate.** A permission is permission to act, not evidence
of acting: the CSV is written to the app's own cache and handed to the share
sheet, and nothing is uploaded. The Data safety form is unaffected too — Play
scopes that to data *transmitted off the device*, and none is.

If you would rather production not carry `INTERNET` at all, the mechanism is the
same `android.blockedPermissions`. Be careful: blocking it breaks the Metro
connection in development builds, so it cannot simply be left on.

## Verified against the build

Facts below were read from a real `expo prebuild`, not inferred:

| Item | Value | Play's requirement |
| --- | --- | --- |
| `targetSdkVersion` | **36** | Meets the current floor for new apps. |
| `minSdkVersion` | 24 | No floor; covers Android 7+. |
| `applicationId` | `com.cutpay.app` | Permanent from first publish. |
| `versionCode` / `versionName` | `1` / `0.1.0` | Bump `versionCode` for every upload. |
| Android bundle | builds — 1346 modules, no unresolved imports | — |

`npx expo export --platform android` bundles cleanly. That proves every import
resolves under Metro, which the jest suites do **not** prove: they mock
`src/db/database` and resolve differently. It does **not** prove the app boots.

One known warning, harmless: `userInterfaceStyle: Install expo-system-ui in your
project to enable this feature`. `app.json` asks for a dark UI but that setting
is a no-op without `expo-system-ui`. Every screen paints its own dark background,
so the app looks right regardless — this is worth fixing only if the window
background flashes light on launch, and adding a native module is not something
to do on the way into a first submission.

## Producing the bundle

`eas.json` in the app repo defines the profiles. From that repo:

```bash
npx expo run:android          # run it locally first — see the warning below
eas build --platform android --profile preview      # installable APK, for testers
eas build --platform android --profile production   # .aab, for Play
```

`appVersionSource` is `local`, so `versionCode` in `app.json` is the single
source of truth — bump it by hand for every upload. Play rejects a bundle whose
versionCode it has already seen.

Submitting from the CLI needs a Google Cloud service-account key with the Play
Developer API enabled and access granted in the Console. `eas.json` expects it
at `./play-service-account.json`, which is gitignored. That key grants publish
rights to the whole app — never commit it, and never paste it into a chat.

```bash
eas submit --platform android --profile production
```

**Run the app before building anything.** Every screen was written and unit
tested but the app has never been launched on a device or emulator, so
`npx expo run:android` is the first real check that it boots at all.

## Store listing assets

Play will not let the listing be completed without these. What the repo has
today, and what it does not:

| Asset | Requirement | State |
| --- | --- | --- |
| App icon | 512×512 PNG, 32-bit | **Have it** — `assets/icon.png` is 1024×1024; Play accepts it and downscales, or export a 512 copy. |
| Adaptive icon | foreground + background | **Have it** — 512×512 each, plus a 432×432 monochrome for themed icons. |
| Feature graphic | 1024×500 PNG/JPG, no alpha | **Missing.** Required for every listing. |
| Phone screenshots | 2–8, 16:9 or 9:16, 320–3840px each side | **Missing.** Minimum of two is enforced. |
| Short description | ≤80 characters | Draft below. |
| Full description | ≤4000 characters | Draft below. |

Screenshots have to come from the app actually running. There is no way around
that and no way to fake it honestly — they are what a reviewer and a buyer look
at, and Play rejects listings whose screenshots do not depict the real app. Run
`npx expo run:android`, seed a few cuts and bookings by hand, and capture at
least: Today with money on it, Log a cut, Schedule, and Money.

### Short description (80 max)

> Know who paid and who didn't. A private, offline ledger for barbers.

That is 68 characters.

### Full description

> CutPay is a ledger for independent barbers. Log a cut in about five seconds,
> see what you made today, and see exactly who walked out without paying.
>
> **The problem it solves.** Money arrives under the wrong name — a wife's Zelle
> account, a nickname, a bare phone number — hours or days after the cut. By
> Friday nobody knows who still owes what. CutPay learns the names: match a
> payment to a cut once, and every future payment under that name finds the
> right client on its own.
>
> **What it does**
> • Log a cut in seconds, walk-in or regular
> • See today's takings and exactly who still owes you
> • Match payments that arrived under an unfamiliar name
> • Book clients in ahead, and see what is still to come today
> • Send a reminder from your own number — you press send, never the app
> • Export any period as a CSV for your accountant
>
> **It is a written record, not a payment app.** CutPay does not connect to
> Zelle, Venmo, Cash App, or any bank. It never touches money and never sees
> your accounts. You type in what you were paid.
>
> **Private by construction.** No account, no server, no analytics, no ads, no
> tracking. Everything lives in one file on your phone and works with no
> signal. Delete the app and the data goes with it.

Check both against the app before pasting them in. A description that promises
something the build does not do is a rejection, and the wording above was
written from the current feature set.

## The submission runbook

In order. Steps 1–3 are the ones that cannot be rushed.

1. **Run the app on a real device.** `npx expo run:android`. It has still never
   been launched — see the warning under *Producing the bundle*. Everything
   below is wasted effort if it does not boot.
2. **Capture the screenshots and build the feature graphic** while it is
   running.
3. **Confirm both listing URLs load** in a normal browser, signed out:
   `https://tesfalemt.github.io/cutpay-support/` and `/privacy.html`. A privacy
   URL that 404s is an automatic rejection.
4. **Create the app in the Play Console** — name CutPay, package
   `com.cutpay.app`. The package name is permanent from here.
5. **Build the bundle.** `eas build --platform android --profile production`,
   or run the Release workflow in the app repo. Needs `EXPO_TOKEN`.
6. **Fill the declarations** from *Data safety form* and *Declarations that are
   easy to get wrong* above. Financial features: none. Ads: none. Data
   collected: none.
7. **Complete the IARC content rating questionnaire.**
8. **Upload to a closed test track first** — and read *Before you can ship to
   production* below before assuming production is available to you.
9. **Submit for review.**

Steps 4 onward need Play Console credentials and cannot be done from a
repository. `eas submit` needs `play-service-account.json`, which is gitignored
and deliberately not in this repo.

## Before you can ship to production

If the developer account is a **personal/individual** account created after
13 November 2023, Google requires a closed test with a minimum number of
opted-in testers sustained over a continuous period before production access
is granted. Check the current thresholds in the Console — this is the single
most common reason a first release sits unshipped for weeks. Organization
accounts are not subject to it. Start the closed test early; it runs on
wall-clock time and cannot be shortened.
