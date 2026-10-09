# Gotchas (learned the hard way: read before fixing or advising)

## App Store Connect
- **"Remove from Sale" on a subscription STOPS RENEWALS for existing subscribers.** Apple's dialog says "Existing subscribers won't be able to renew." Never tell the user it only stops new purchases. To retire a leftover price-test group, leave it on sale and stop showing it in the paywall (set the winning offering as default in RevenueCat). Only remove it when its subscriber count is close to zero.
- **The description limit is 4,000 characters, and App Store Connect counts more than Python's `len()`.** Newlines and `•` cost extra (a 3,951-char draft showed as -45 over). Leave about 100 characters of headroom, and offer specific cuts when the user is over.
- **There are two app-info records** while a new version is in preparation. Age rating, name, subtitle and privacy URL edits land on the draft record and go live with that version. Read both.
- **The App Privacy label publishes immediately** when the user clicks Publish, not with the next version. That's fine as long as it matches the live version.
- **Age rating:** a medical app needs Medical/Treatment Information = Frequent and Health or Wellness Topics = Yes. A disclaimer does not make the content go away. The rating number usually doesn't change if it's already 16+/17+.
- **Wording:** "Your data now stays private" implies it wasn't private before. Use neutral phrasing ("iCloud sync has been turned off. Your data stays on this iPhone.").

## Links in chat
- Put each URL on its own line, with nothing after it. The user's terminal glued "…agreement/. It covers…" onto the link and the page 404'd.

## Fetching pages
- WebFetch caches for 15 minutes, so a page just deployed will look old. Verify with `curl -s "<url>?nc=$(date +%s)" | sed 's/<[^>]*>/ /g'` and search the text with python (ugrep can fail on long UTF-8 regexes).

## SwiftData, CloudKit and storage
- **With an App Group entitlement, SwiftData's default store lives in the App GROUP container** (`Shared/AppGroup/<id>/Library/Application Support/default.store`), not in the app's data container. Use `xcrun simctl get_app_container <udid> <bundle> groups` to find it.
- **Don't run `sqlite3` on a path that doesn't exist.** It creates an empty file. Use `sqlite3 -readonly`, and check the file exists first.
- **Switching `cloudKitDatabase: .private(...)` to `.none` opens the SAME default store,** so existing data stays. A device test with real synced data confirmed: data kept, new saves persist after relaunch, reminders still fire. Still do the before/after upgrade test every time.
- **When removing CloudKit, also remove:** the iCloud entitlements (`icloud-container-identifiers`, `icloud-services`), `UIBackgroundModes: remote-notification` (if nothing else uses push), any schema warm-up that inserts dummy records, and debug schema initializers. Update the description ("iCloud Sync" bullets) and the privacy policy ("from version x.y.z data stays on device", plus how to delete old iCloud data: Settings > [name] > iCloud > Manage Storage > <App>).
- **Old iCloud copies stay in the user's iCloud** after removal. The app can't delete them once CloudKit is gone.

## Notifications
- When removing a scheduled marketing campaign, also clean up existing installs: on launch, remove pending AND delivered requests with the campaign's identifier prefix. Use the async `pendingNotificationRequests()` / `deliveredNotifications()` APIs, because callback closures in MainActor-default modules can trip isolation checks.
- Keep the notification permission request if reminders need it. Don't delete it along with the campaign.

## Analytics
- Removing health details from analytics loses metrics (adherence rate etc.). Tell the user what they'll lose before doing it.

## Consent popup placement
- A SwiftUI `.alert` on the root view can't show while a sheet or full-screen cover is up. Present a `UIAlertController` from the top-most view controller (see `fix-recipes.md`), and guard against a failed presentation so the caller never hangs.
- Ask BEFORE spending credits or running fake loading steps, and never deduct a credit when the user declines.

## Process
- Commit unrelated pending changes separately before starting a risky change, so the two can be reverted independently.
- The user often tests on their own physical device. When they say they're installing on device, tell them which branch Xcode is on, what to note BEFORE installing (counts of their data), and the key check (add something new, force-quit, relaunch, confirm it's still there).
- If the user wants something reverted mid-task, revert only that part (`git restore --staged --worktree <paths>`), keep the rest, and record it in the report as an owner decision.
- Peer sessions (e.g. a website repo session) need the user's approval in THEIR terminal; a relayed "yes" doesn't count. Tell the user to answer there.
