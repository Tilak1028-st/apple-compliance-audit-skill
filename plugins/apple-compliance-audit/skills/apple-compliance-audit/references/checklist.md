# Audit checklist with grep recipes

Run greps from the project root. Exclude build/vendor folders: `--exclude-dir={.git,Pods,node_modules,build,DerivedData,.build}`. Swift globs need quotes: `--include='*.swift'`. For React Native/Flutter, also search `ios/`, `*.ts`, `*.tsx`, `*.js`, `*.dart`. Always READ each hit before you flag it.

---

## 1. Safety (Section 1)

**1.1 / 1.2 / 4.7: user-generated content, chat, and AI output**
- Any chat, community or AI feature needs filtering, a way to report content and (for UGC) blocking.
- Look for LLM safety settings that are switched off:
  `grep -rnE 'threshold: *\.none|BLOCK_NONE|HarmBlockThreshold|safetySettings' --include='*.swift' --include='*.ts' .`
- No "report this answer" option on AI chat counts as a Medium finding.

**1.4.1: medical and health apps**
- Disclaimers ("not medical advice", "consult a doctor/pharmacist") in the app AND in the App Store description.
- Accuracy claims: `grep -rniE 'accurate|verified|guarantee|100%|clinically|doctor.approved|FDA|#1|fastest' --include='*.swift' .` Also check notification copy, onboarding, paywall and the description.
- A "Verified" or "Confirmed" label on AI output is High: nothing actually verifies it.
- If the app has regulatory clearance, it needs a link to it in review notes.

**1.4.2: dosage**
- `grep -rniE 'dosage|dose|mg|typicalDosage|recommended dos' --include='*.swift' .`
- A dose calculator is High unless it comes from an approved entity. AI-written "typical dosage" text is Medium.

**1.5: contact**
- The support URL must load and contain contact info. Check for an in-app contact option (mailto or form).

## 2. Performance (Section 2)

**2.1: completeness**
- `grep -rniE 'lorem|TODO|FIXME|coming soon|placeholder text|test data|dummy' --include='*.swift' .` Read each hit: many are code comments, not UI.
- Dead buttons and broken links: check every URL string (`grep -rnoE 'https?://[^" )]+' --include='*.swift' .`) and fetch the important ones.

**2.3.1: hidden or misleading behavior**
- Remote switches that change features after review: `grep -rniE 'remoteConfig|featureFlag|isEnabled\(|killSwitch|RemoteConfig' .`
- **Fake progress steps** claiming work that never happens, like "Matching with medicine database" or "Cross-referencing drug databases" when the app makes one LLM call. Look at loading views for step arrays and `Task.sleep` timers that run before the real request.
- App Review notes must describe new features specifically (check with asc).

**2.3.7 / 2.3.8 / 2.3.10 / 2.3.12: metadata**
- Name and subtitle ≤ 30 chars. No prices, competitor or trademark names, other platforms, or keyword stuffing.
- Description: no "#1"/"fastest"/"best" claims you can't prove, no wrong free-tier promises (compare with the code's credit or limit logic), no "unlimited" if there's a server cap.
- What's New must mention significant changes, including removed features.

**2.5.1 / 2.5.2: private APIs and downloaded code**
- `grep -rnE 'NSClassFromString|dlopen|dlsym|NSSelectorFromString|valueForKey: *"_|value\(forKey: "_|JSContext|evaluateJavaScript|codepush|CodePush' .`

**2.5.4: background modes**
- Read `UIBackgroundModes` in Info.plist. Each mode needs a real use. `remote-notification` is fine for CloudKit or real push, and unneeded once CloudKit is removed.

**2.5.14 / DPLA 3.3.3(A): recording indicators**
- Camera or mic capture needs a visible indicator. A camera preview counts.

## 3. Business (Section 3)

**3.1.1: payments**
- `grep -rniE 'stripe|paypal|checkout|buy on (our )?web|external.*purchase|SFSafariViewController.*(pay|buy)' .`
- Credits bought with IAP must not expire, and there must be a restore mechanism.

**3.1.2: subscriptions (and DPLA Schedule 2 §3.8)**
- The paywall must show: title, length, full renewal price (most prominent), trial length plus the price after the trial, auto-renew wording, Terms (EULA) and Privacy links, Restore, and a visible close button.
- Remote paywalls (RevenueCat `PaywallView`, Superwall, Adapty) are configured in a dashboard, so add them to "Check manually".
- Restore: `grep -rn 'restorePurchases' .` Check that the success and failure messages are honest ("No purchases found" instead of "Restore Failed").
- Multiple subscription groups selling the same thing (often left over from price experiments) is a 3.1.2(b) risk. Check whether the paywall is hidden for users who already have the entitlement.
- **5.6 / 3.1.2(a) manipulation:** false urgency ("your reminders will pause tomorrow" when they won't), fake countdowns, hidden or delayed close buttons, "free" without trial terms, and a "no ads" claim when the free tier has no ads.

**3.2.2 / 5.6.1: ratings**
- `grep -rnE 'requestReview|SKStoreReviewController|AppStore.requestReview|write-review' .`
- Only the system prompt may be used. No rewards for ratings, and nothing gated behind a review.

## 4. Design (Section 4)

**4.1: copycats.** Brand names or icons that aren't yours.

**4.2: minimum functionality.** Not a thin web wrapper.

**4.3: spam.** Run `asc apps list --limit 200`. Many near-identical template apps on the account ("AI X Identifier", "AI X Solver") is High risk for the whole account. Pass this to the owner as a decision.

**4.5.4: push and marketing notifications**
- `grep -rniE 'UNNotificationRequest|UNCalendarNotificationTrigger|UNTimeIntervalNotificationTrigger|engagement|promo|winback|win-back|trial' --include='*.swift' .`
- Marketing notifications (trial nudges, re-engagement, "new feature") need explicit opt-in consent wording AND an in-app opt-out. Using the reminder permission for marketing counts as a finding. Check that every claim in the copy is true.

**4.7: AI, chatbots, mini apps.** Moderation, a way to report content, no objectionable output.

**4.8: login.** Google or Facebook login requires an equivalent privacy-focused login such as Sign in with Apple.

## 5. Legal and privacy (Section 5)

**5.1.1(i): privacy policy**
- It must be linked in the app (Settings) AND in App Store Connect, and it must be accurate. Compare EVERY claim against the real data flows: analytics with user IDs, iCloud/CloudKit, AI providers and proxies, RevenueCat, crash reporting, Visual Intelligence or other system captures. Two different policy URLs (ASC vs app) is a finding on its own.

**5.1.1(ii) / 5.1.2: consent and data use**
- Analytics with a persistent ID (`identifierForVendor`, custom UUID) and health-related events (dose taken/missed, symptoms, cycle data) count as health data in analytics. Strip health details or get consent.
- `grep -rn -A4 'log(event\|signal(\|track(\|logEvent(' --include='*.swift' .` Read every parameter.

**5.1.1(iii): data minimization**
- Photos: `grep -rnE 'PHPhotoLibrary.requestAuthorization|requestAuthorization\(for: \.readWrite\)' .` Full library access while using `PHPickerViewController` or `PhotosPicker` (which need no permission) is a finding. Note: some owners keep it on purpose after past picker problems. Ask before changing it.
- Contacts, location and others: only ask when a feature needs it.

**5.1.1(iv): purpose strings**
- `grep -n 'UsageDescription' Info.plist *.pbxproj`. Strings must be specific and honest. Remove unused ones.

**5.1.1(v): account deletion**
- If accounts exist (including auto-created "guest" accounts on a backend), deletion must be possible in the app.

**5.1.1(ix): regulated fields**
- Healthcare, finance and similar apps should come from an Organization account. Ask the owner, because asc can't read the account type.

**5.1.2(i): third-party AI (added Nov 2025). Often Critical.**
- Find every LLM call: `grep -rniE 'openai|gemini|anthropic|claude|aiproxy|generativelanguage|chat/completions|vertex|bedrock|groq|mistral' .`
- Personal data (photos, health questions, chat history) sent to a third-party AI needs CLEAR disclosure plus EXPLICIT permission in the app BEFORE the first send, and a way to withdraw it. A privacy policy alone is not enough.

**5.1.3: health data**
- (i) No health data used for ads or marketing, and disclose what health data is collected.
- (ii) "may not store personal health information in iCloud." With DPLA 3.3.3(D), that covers CloudKit with medication or health records. Check `cloudKitDatabase: .private`, `NSPersistentCloudKitContainer`, `CKContainer`, and the iCloud entitlements. Measure real sync use (CloudKit Console → Usage) before you recommend keeping or removing it.

**5.2: intellectual property**
- `grep -rn '®\|™' --include='*.swift' --include='*.plist' .` plus the App Store name. ® is only allowed for a registered trademark. Check image assets named after other products (e.g. "ChatGPT Image..."). Fonts and sounds need licenses.

**ATT: tracking**
- `grep -rnE 'advertisingIdentifier|ATTrackingManager|ASIdentifierManager|AdSupport' .` Tracking SDKs (Meta, AppsFlyer, Adjust, ad networks) need the ATT prompt before tracking.

**Privacy manifest (DPLA 3.3.3(B))**
- `find . -name '*.xcprivacy' -not -path './.git/*' -not -path '*/Pods/*'`. Each app target AND each extension (widgets!) that uses required-reason APIs needs its own.
- Required-reason APIs to look for:
  - UserDefaults: `UserDefaults|@AppStorage`. Codes: CA92.1 for the app's own data, 1C8F.1 for an App Group `UserDefaults(suiteName:)`.
  - File timestamps: `creationDate|modificationDate|contentModificationDateKey|creationDateKey|attributesOfItem|fileModificationDate`. Codes: C617.1 for files inside the app/group/CloudKit container, DDA9.1 for display only, 3B52.1 for user-picked files.
  - System boot time: `systemUptime|mach_absolute_time`. Codes: 35F9.1, 8FFB.1.
  - Disk space: `volumeAvailableCapacity|systemFreeSize|statfs`. Codes: E174.1, 85F4.1.
  - Active keyboards: `activeInputModes`.
- Third-party SDK manifests: look in DerivedData `SourcePackages/checkouts/**/PrivacyInfo.xcprivacy` (or Pods). Confirm they're present.
- `NSPrivacyCollectedDataTypes` should match the App Privacy label.

## Kids and age rating
- Use asc to read the age rating declaration on the RIGHT app-info record (the live one AND the "prepare for submission" one).
- A medical or health app with "Medical/Treatment Information = NONE" or "Health or Wellness Topics = false" is High.
- If the app targets kids: no third-party analytics or ads, and parental gates are needed.

## App Privacy label
`asc` can't read it through the official API (`asc web privacy pull` needs an unofficial web login, so don't use it). Ask the user for screenshots, or go through the data-collection screens with them. "Data Not Collected" is wrong for any app using analytics with IDs, RevenueCat, or uploads to an AI. Typical mapping:

| Data | Type | Linked | Purposes |
|---|---|---|---|
| Analytics user ID (IDFV-based) | User ID + Device ID | Yes | App Functionality, Analytics |
| RevenueCat purchase history | Purchases | Yes | App Functionality, Analytics |
| Feature events | Product Interaction | Yes | Analytics |
| Error diagnostics in analytics | Other Diagnostic Data | Yes | Analytics |
| Photos sent to an AI | Photos or Videos | No if no ID is attached | App Functionality |
| Questions or text sent to an AI | Other User Content | No if no ID is attached | App Functionality |
| Health or medication data (AI or analytics) | Health | Yes if analytics carries it | App Functionality (+ Analytics) |

Tracking: No for all of these, unless ads or attribution SDKs are present.
