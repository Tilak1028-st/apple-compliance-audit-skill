# App Store Connect via `asc` (read-only)

Use `--profile <name>` on every call. Don't `asc auth switch` unless the user asks. Profile names are often different from what the user calls them (e.g. the user says "Maths for Kids" but the profile is `MathForKids`), so check `asc auth status` and pick the closest match.

When unsure of a subcommand, run `asc <cmd> --help`. Output is JSON. Save it to a scratch file, then parse it with `python3 -I`. Piping `asc` straight into `json.load(sys.stdin)` sometimes fails because of auth/status noise, so redirect stderr: `asc ... > out.json 2> err.txt`.

## Discovery
```
asc auth status
asc apps list --limit 200                         # every app on the account (needed for the 4.3 check)
asc apps list --bundle-id com.example.app         # get the App ID
asc apps info list --app APP_ID                   # app-info records: live (READY_FOR_SALE) + draft (PREPARE_FOR_SUBMISSION)
asc versions list --app APP_ID                    # version IDs, versionString, appStoreState
```

## Metadata
```
# Version text: description, keywords, what's new, support/marketing URL, promo text
asc localizations list --version VERSION_ID --locale en-US
# App-info text: name, subtitle, privacyPolicyUrl. Pass --app-info when there are 2 records!
asc localizations list --app APP_ID --type app-info --app-info APP_INFO_ID
asc app-info get --app APP_ID
```
The live record and the draft record can differ (e.g. the privacy URL was changed on the draft only). Report both.

## Age rating
```
asc age-rating view --app-info-id APP_INFO_ID     # check BOTH records; edits land on the draft
```
Fields to read: `medicalOrTreatmentInformation`, `healthOrWellnessTopics`, `userGeneratedContent`, `messagingAndChat`, `unrestrictedWebAccess`, `advertising`, `ageRatingOverride*`.

## Monetization
```
asc subscriptions groups list --app APP_ID
asc subscriptions list --group-id GROUP_ID
asc subscriptions pricing summary --app APP_ID
asc subscriptions offers win-back list --subscription-id SUB_ID
asc iap list --app APP_ID                        # consumables / non-consumables (often empty)
asc eula view --app APP_ID                       # empty = Apple standard EULA
```

## Other
- Screenshots: list the sets per device type through the version localization's `appScreenshotSets` (see `asc screenshots --help`). Don't download them unless asked.
- App Review contact and notes: shown on the version (see `asc review --help`). Empty notes on a version with new features is a 2.3.1(a) finding.

## Known gaps: write "Check manually"
- **App Privacy nutrition label:** not in the official API. `asc web privacy pull` uses private endpoints and a web login. Don't use it; ask the user for screenshots.
- **Developer account type** (Individual vs Organization): ask the user.
- **RevenueCat or other remote paywalls:** dashboard only, so ask for screenshots.
- **CloudKit usage:** CloudKit Console → container → Production → Usage / Telemetry (screenshots).

## Writes (ONLY after the user approves that specific change)
Prefer giving the user exact text to paste in the App Store Connect UI. After any change, verify it with the read commands above.
