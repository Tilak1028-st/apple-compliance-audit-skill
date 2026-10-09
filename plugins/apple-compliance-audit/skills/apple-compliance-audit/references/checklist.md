# Audit checklist with grep recipes

Run greps from the project root. Exclude build/vendor folders: `--exclude-dir={.git,Pods,node_modules,build,DerivedData,.build}`. Swift globs need quotes: `--include='*.swift'`. For React Native/Flutter, also search `ios/`, `*.ts`, `*.tsx`, `*.js`, `*.dart`. Always READ each hit before you flag it.

---

## 0. Upload requirements (check first)

These block the upload itself, before App Review sees the app. Confirm the current list on Apple's "Upcoming requirements" page every time, because dates change.
- **SDK:** since Apr 28, 2026, builds must use Xcode 26+ with the iOS 26 SDK. Run `xcodebuild -version`. For Expo/React Native/Flutter, check the CI image or EAS build image too.
- **Minimum target:** since Sep 9, 2026, iOS apps must target iOS 13 or later. `grep -rn 'IPHONEOS_DEPLOYMENT_TARGET' --include='*.pbxproj' .` (also `Podfile` `platform :ios`).
- **Age rating questions:** the new questionnaire had to be answered by Jan 31, 2026. Unanswered questions block update submissions. Check with asc (see `asc-commands.md`).
- **EU trader status (DSA):** without it the app is removed from the EU storefront. asc can't read it, so add it to "Check manually".

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

**1.1.6: fake functionality**
- "Prank" features, fake trackers or detectors, and simulated results presented as real (e.g. a "metal detector" or "heart rate" that is random). Read any feature whose output is generated with `random`/`Int.random`/`Double.random`: `grep -rnE '\.random\(|arc4random' --include='*.swift' .`

**1.6: data security (often Critical)**
- **Hardcoded secrets** for paid APIs (OpenAI, Gemini, Anthropic, AWS, Stripe secret keys) ship inside the binary and can be extracted. Critical when the key can run up a bill or read user data.
  `grep -rnE 'AIza[0-9A-Za-z_-]{35}|sk-(proj-|ant-)?[A-Za-z0-9_-]{20,}|AKIA[0-9A-Z]{16}|sk_live_[0-9A-Za-z]{20,}|ghp_[A-Za-z0-9]{36}|xox[baprs]-' --exclude-dir={.git,Pods,node_modules,build,DerivedData,.build} .`
  `grep -rniE '(api[_-]?key|secret|token|password)[A-Za-z_]*\s*[:=]\s*"[^"]{16,}"' --exclude-dir={.git,Pods,node_modules,build,DerivedData,.build} .` Also check `*.xcconfig`, `*.plist`, `.env*` files that are bundled.
  Public-by-design keys are fine: RevenueCat `appl_` keys, Firebase `GoogleService-Info.plist` (but then check that Firebase security rules exist), Supabase anon keys (check RLS is on).
  When you report a secret, mask it (first 4 characters + `...`). Never paste the full value.
- **App Transport Security:** `grep -rn -A4 'NSAppTransportSecurity' --include='*.plist' .` `NSAllowsArbitraryLoads = true` needs a real reason; plain `http://` URLs for user data are a finding.
- **Tokens and passwords in UserDefaults** instead of the Keychain: `grep -rniE '(UserDefaults|@AppStorage).*(token|password|secret|session|apiKey)' --include='*.swift' .`
- **Logging personal data:** `grep -rnE 'print\(|NSLog\(|os_log|Logger\(' --include='*.swift' .` and look for tokens, emails or health data in log lines that ship in release builds.

**1.3: Kids Category.** If the app is in the Kids Category: no external links or purchases without a parental gate, no third-party analytics or ads (see 5.1.4).

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

**2.1(a) / 2.1(b): ready for review**
- If the app has login, App Review needs a working demo account (or a demo mode) in the review notes, and the backend must be live during review.
- Every IAP and subscription the build uses must be submitted for review with the version and be visible and buyable in the sandbox. Check with asc: products in "Missing Metadata" or "Developer Action Needed" state are a finding.

**2.3.2: promoting IAP.** Description and screenshots must make clear which features need a purchase.

**2.3.3 / 2.3.4 / 2.3.9: screenshots and previews**
- They must show the app in use (not only a splash or login screen), use real app UI, and contain no real people's personal data or media you have no rights to. No other platforms' devices or icons (2.3.10).
- Use asc to list screenshot sets per device. A universal app needs iPad screenshots.

**2.4.1: iPad.** Read `TARGETED_DEVICE_FAMILY` in the pbxproj (`1` = iPhone, `1,2` = universal). iPhone-only apps still run on iPad and are reviewed there: check that layouts don't break, sheets/popovers don't crash (e.g. `UIActivityViewController` or action sheets without a `popoverPresentationController` source crash on iPad), and nothing assumes a phone-only feature.

**2.4.4: no restart or settings prompts.** The app must not ask users to restart the device or change unrelated system settings.

**2.5.5: IPv6.** Hardcoded IPv4 addresses break on IPv6-only networks: `grep -rnoE '([0-9]{1,3}\.){3}[0-9]{1,3}' --include='*.swift' --include='*.ts' --include='*.dart' .` (ignore `127.0.0.1`, `0.0.0.0` and version strings).

**2.5.6: web content.** Browsing must use WebKit (`WKWebView`/`SFSafariViewController`). Flag `UIWebView` (removed API).

**2.5.18: ads.** Ads must not appear in extensions, widgets, notifications or the keyboard, must be closable, must match the age rating, and need ATT before tracking. `grep -rniE 'GADBannerView|GoogleMobileAds|AppLovin|IronSource|UnityAds|AdMob|interstitial|rewardedAd' .`

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

**3.1.1(a): links to outside purchases.** Only allowed with Apple's StoreKit External Purchase entitlements, and only in the storefronts those entitlements cover. A "buy on our website" button or link without the entitlement is Critical. Rules differ by region (US, EU, others) and change often: confirm against the fetched guidelines.

**3.1.2(a) / 3.1.2(c): subscription basics**
- The shortest subscription period is 7 days. Read durations with asc.
- The paywall and description must say clearly what the subscriber gets. "Premium" alone is not enough.
- A subscription must give ongoing value. A one-time unlock sold as a subscription is a finding.

**3.1.3 / 3.1.4: allowed exceptions.** Reader apps, multiplatform services, physical goods, person-to-person services and hardware-unlocked features may use other payment methods. If the app relies on one, check the exact conditions in the fetched text.

**3.1.5: crypto (if present).** Wallets need an Organization account, no on-device mining, exchanges only where licensed, no crypto rewards for tasks.

**3.2.2: unacceptable business models.** Ad-stuffed apps, forcing users to rate/review/download other apps to unlock features, artificially raising prices, charging for features that are free elsewhere in iOS.

## 4. Design (Section 4)

**4.1: copycats.** Brand names or icons that aren't yours.

**4.2: minimum functionality.** Not a thin web wrapper.

**4.3: spam.** Run `asc apps list --limit 200`. Many near-identical template apps on the account ("AI X Identifier", "AI X Solver") is High risk for the whole account. Pass this to the owner as a decision.

**4.5.4: push and marketing notifications**
- `grep -rniE 'UNNotificationRequest|UNCalendarNotificationTrigger|UNTimeIntervalNotificationTrigger|engagement|promo|winback|win-back|trial' --include='*.swift' .`
- Marketing notifications (trial nudges, re-engagement, "new feature") need explicit opt-in consent wording AND an in-app opt-out. Using the reminder permission for marketing counts as a finding. Check that every claim in the copy is true.

**4.7: AI, chatbots, mini apps.** Moderation, a way to report content, no objectionable output.

**4.8: login.** Google or Facebook login requires an equivalent privacy-focused login such as Sign in with Apple.

**4.2.6: template apps.** Apps built from a commercial template or app generator are rejected unless submitted by the content provider. Check this together with 4.3.

**4.5.6: Apple emoji.** Don't ship Apple emoji images (e.g. as PNG assets or in a custom font) for use outside Apple platforms or as app content.

**4.9: Apple Pay (if used).** Disclose all purchase terms before the sale. Recurring payments must state length, what's provided, charges and how to cancel.

**4.10: monetizing built-in features.** Charging for capabilities iOS already provides for free (flashlight, camera, push notifications, iCloud storage, Screen Time API) is a finding. Common in utility and template apps.

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

**5.1.1(v) / 5.1.1(x): login and personal info**
- Don't require login for features that don't need an account. Look at the first screens: a forced sign-up wall before any core feature is a finding.
- Asking for name, email or phone must be optional unless the feature needs it, and must not be required to use the app.

**5.1.5: location**
- `grep -rnE 'CLLocationManager|requestWhenInUseAuthorization|requestAlwaysAuthorization|startUpdatingLocation' --include='*.swift' .`
- Use location only when it's directly relevant to a feature, explain why in the app and in the purpose string, and prefer "When In Use". "Always" needs a clear background reason.

**5.2.2 / 5.2.3: third-party content.** Showing content from other services must be allowed by their terms. Downloading or saving media from YouTube, Instagram, Apple Music, SoundCloud and similar is not allowed without permission.

**5.2.4 / 5.2.5: Apple trademarks and endorsement**
- Don't imply Apple endorses the app ("Apple approved", "Recommended by Apple") or use Apple product images or the Apple logo in the icon, screenshots or UI.
- Don't copy Apple app designs or the Activity rings look. Apps using WeatherKit must show the Apple Weather attribution.
  `grep -rniE 'apple (approved|recommended|certified)|WeatherKit|WeatherService' .`

**5.6.2 / 5.6.3 / 5.6.4: developer conduct**
- Developer name, support info and website must be accurate and current.
- No manipulating rankings, reviews or referrals (incentivized reviews, review-gating, fake download campaigns).
- High refund rates or many complaints count against the account. If the user has data on this, note it.

## Kids and age rating
- Use asc to read the age rating declaration on the RIGHT app-info record (the live one AND the "prepare for submission" one).
- The tiers are now 4+, 9+, 13+, 16+ and 18+ (17+ no longer exists). Every question in the updated questionnaire must be answered, including in-app controls and capabilities (UGC, messaging, web access, advertising).
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

## HIG basics (Low or Medium unless they break the app)
- **Dark Mode:** hardcoded colors that become unreadable in dark mode. `grep -rnE 'Color\(red:|UIColor\(red:|\.foregroundColor\(\.black\)|\.background\(\.white\)' --include='*.swift' .` then check in the simulator.
- **Dynamic Type:** fixed font sizes everywhere (`.font(.system(size:` with no scaling) make text unreadable at large sizes.
- **Accessibility:** icon-only buttons without `accessibilityLabel`, images that carry meaning with no label. `grep -rnE 'Image\(systemName:' --include='*.swift' .` and check the nearby code for labels.
- **Tap targets** smaller than 44x44 pt, and content hidden under the notch or home indicator.
- **Permissions:** ask in context, right before the feature needs it, not all at launch.

## Niche rules (check only if the app does this)
| If the app... | Check |
|---|---|
| has chat, a community or AI-generated content others see | 1.2, 1.2.1(a) creator content age gate, 4.7.x |
| is a mini-app, game streaming, plug-in or emulator host | 4.7.1 to 4.7.5 |
| has a keyboard or Safari extension | 4.4.1, 4.4.2 |
| is a VPN | 5.4: NEVPNManager, Organization account, no data sale |
| uses MDM or config profiles | 5.5 |
| has gambling, contests or lotteries | 5.3.1 to 5.3.4 |
| uses Apple Music, MusicKit or Game Center | 4.5.2, 4.5.3, 4.5.5 |
| uses ARKit | 4.2.1 |
| is a remote desktop client | 4.2.7 |
| reports crimes or tracks police | 1.7, 1.4.4 |
| promotes alcohol, tobacco, drugs or risky challenges | 1.4.3, 1.4.5 |
| blocks calls or SMS | 2.5.12 |
| uses face login | 2.5.13: LocalAuthentication only |
| ships on the Mac App Store | 2.4.5 |
| uses Matter | 2.5.17 |

