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

The backup file added in Phase 7 does not change this answer. It is written on
a tap, handed to the share sheet, and goes wherever the barber sends it — the
app has no destination of its own and uploads nothing. Same category as the CSV
export, and declared the same way.

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

Re-checked after `expo-document-picker` was added for restore in Phase 7: it
declares no permissions. It goes through the system document picker, so the app
is handed a scoped URI for the one file the user chose rather than the right to
browse storage. The list above is unchanged.

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
| Android bundle | builds — 1356 modules, no unresolved imports | — |

`npx expo export --platform android` bundles cleanly — 1360 modules. That
proves every import resolves under Metro, which the jest suites do **not**
prove: they mock `src/db/database` and resolve differently.

Stronger than that now: the app has actually **run**. The web target boots,
migrates, and takes real writes — the `store/` screenshots were made by entering
cuts, bookings and a payment through the real forms. Running it that way found
two defects that had been in the repo since Phase 1: the web bundle could not
resolve `wa-sqlite.wasm`, and every transaction failed on web, which meant no
write of any kind could succeed there. Both fixed; neither affected Android.

It still does **not** prove the app boots on Android.

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

All of these now live in **`store/` in the app repo** (`Tesfalemt/client-tracker`).

| Asset | Requirement | State |
| --- | --- | --- |
| App icon | 512×512 PNG, 32-bit | **Ready** — `store/play-icon-512.png`. |
| Adaptive icon | foreground + background | **Ready** — in `assets/`, 512×512 each, plus a 432×432 monochrome. |
| Feature graphic | 1024×500 PNG/JPG, no alpha | **Ready** — `store/feature-graphic.png`. |
| Phone screenshots | 2–8, 16:9 or 9:16, 320–3840px each side | **Six, ready** — `store/screenshots/`, 1236×2196. Read the caveat. |
| Short description | ≤80 characters | Ready, below. |
| Full description | ≤4000 characters | Ready, below. |

**The screenshot caveat.** They are the real app — the actual screens, running
the actual code, against a real SQLite database, with five cuts, three bookings
and a matched Zelle payment entered through the real forms. They are not
mockups. But they were captured from the **web build**, not from an Android
device, so font rendering and spacing are close to Android without being
identical, and there is no status bar.

That is good enough to complete a listing. It is not good enough to ship
without looking: once `npx expo run:android` has been done, retake them on the
phone — four taps per screen — and replace them. Play rejects listings whose
screenshots do not depict the real app, and layout that drifts on a real screen
would be exactly that.

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
> • See who is overdue for a cut, and book them straight from the list
> • Send a reminder from your own number — you press send, never the app
> • Export any period as a CSV for your accountant
> • Back up everything to a single file, and restore it on a new phone
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

1. **Install the APK on a real phone.** There is now a signed one to install:
   open [build `cbe6d5ad`][build] on the phone and tap install — no Android
   SDK, no `expo run:android`, no cable. CutPay has been run on the web target
   (it boots, migrates and takes real writes) but **never on Android**, and
   only a device exercises the share sheet, document picker, dialer and
   messaging hand-offs. A green cloud build proves the app *compiles*; it does
   not prove it boots. Everything below is wasted effort if it does not.
2. **Retake the screenshots on the device** and replace the ones in `store/`.
   The feature graphic and icon are done and need no device.
3. **Confirm both listing URLs load** in a normal browser, signed out:
   `https://tesfalemt.github.io/cutpay-support/` and `/privacy.html`. A privacy
   URL that 404s is an automatic rejection.
4. **Create the app in the Play Console.** *All apps* → *Create app*. This is
   the step no tool can do for you — the Play Developer API has no method for
   it. Five fields:

   | Field | Value |
   | --- | --- |
   | App name | `CutPay` |
   | Default language | English (United States) |
   | App or game | App |
   | Free or paid | Free |
   | Declarations | tick both (developer programme policies, US export laws) |

   Note what it does **not** ask for: the package name. `com.cutpay.app` is
   bound by the *first bundle you upload*, not here — which is why `app.json`
   already carries it and must not change afterwards. Free→paid is a one-way
   door too; a free app can never be made paid.
5. **Build the bundle.** Run the Release workflow in the app repo with
   `profile: production` — proven working, and `EXPO_TOKEN` is already set.
   Leave `submit` unticked until step 6 is done.
6. **Fill the declarations** from *Data safety form* and *Declarations that are
   easy to get wrong* above. Financial features: none. Ads: none. Data
   collected: none.
7. **Complete the IARC content rating questionnaire.**
8. **Upload to a closed test track first** — and read *Before you can ship to
   production* below before assuming production is available to you.
9. **Submit for review.**

Steps 4, 6, 7 and 9 need a Console login and cannot be done from a repository.
Step 5 can now be done entirely from CI. Step 8's upload can too, once
`PLAY_SERVICE_ACCOUNT` exists — see *Getting `PLAY_SERVICE_ACCOUNT`* in the app
repo's README for how to make that key, and run the preflight workflow to
confirm it works before spending a build on finding out.

## What cannot be done from a Claude Code session

Recorded because it has been asked more than once, and the answer is not about
permission.

The session's network policy refuses the hosts this would need. Both fail at
the proxy's CONNECT, before any question of a login:

| Host | Result |
| --- | --- |
| `play.google.com:443` | `403 — policy denial` |
| `api.expo.dev:443` | `403 — policy denial` |

So the Play Console cannot be opened from here, and neither can EAS.

**The build half of that is now solved.** GitHub Actions runs on GitHub's
network, not this sandbox's, so the Release workflow in the app repo reaches
EAS perfectly well — and on 20 August it produced the first real Android
build of CutPay, [`cbe6d5ad`][build], a signed preview APK, in nineteen
minutes. Switching the profile from `preview` to `production` produces the
`.aab` the same way. An earlier version of this file said no bundle could be
produced at all; that was true of the container and never true of CI, and it
is corrected here rather than quietly deleted.

[build]: https://expo.dev/accounts/cutpay/projects/cutpay/builds/cbe6d5ad-3c8c-4985-ad1e-7d8d32102007

What remains genuinely impossible from here is the **Console** — creating the
app, and the human declarations. Granting permission does not change that: a
sandbox network policy is not a consent prompt. And no API fixes it either;
the Play Developer API has no method that creates an app, so even a service
account with full publishing rights cannot make the first one. It is a browser,
a Google login, and about ten minutes.

## Before you can ship to production

If the developer account is a **personal/individual** account created after
13 November 2023, Google requires a closed test with a minimum number of
opted-in testers sustained over a continuous period before production access
is granted. Check the current thresholds in the Console — this is the single
most common reason a first release sits unshipped for weeks. Organization
accounts are not subject to it. Start the closed test early; it runs on
wall-clock time and cannot be shortened.
