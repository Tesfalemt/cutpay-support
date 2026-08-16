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
- Meet Play's current target API level requirement for new releases.
- Use Play App Signing and keep the upload key backed up somewhere you will
  still have it in two years.

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

## Before you can ship to production

If the developer account is a **personal/individual** account created after
13 November 2023, Google requires a closed test with a minimum number of
opted-in testers sustained over a continuous period before production access
is granted. Check the current thresholds in the Console — this is the single
most common reason a first release sits unshipped for weeks. Organization
accounts are not subject to it. Start the closed test early; it runs on
wall-clock time and cannot be shortened.
