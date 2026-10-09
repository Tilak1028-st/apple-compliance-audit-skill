# Fix recipes (tested in a shipped App Store app, Oct 2026)

Adapt names, URLs and the AI provider to the app. Match the project's style (ObservableObject vs @Observable, comment density). Build after every recipe.

---

## A. Third-party AI consent popup (5.1.2(i), Critical)

**What it does:** asks once before the first AI request, stores the choice, and offers a Settings toggle to withdraw it. Declining sends nothing and uses no credit.

**Popup copy** (example for an AI pill-identifier app; short copy is fine, and naming the proxy in the popup isn't needed, but it must be in the privacy policy):
> **Share with AI?**
> To identify pills and answer questions, your photo, typed text and questions are sent to Google Gemini.
> [Allow] [Privacy Policy] [Not Now]

`Core/System/AIConsentManager.swift`:
```swift
import UIKit

@MainActor
enum AIConsentManager {
    /// UserDefaults key. Settings binds its toggle to this via @AppStorage.
    static let consentKey = "aiDataSharingConsent"
    private static let privacyPolicyURL = URL(string: "https://example.com/privacy")!
    /// The prompt currently on screen, so concurrent callers share one popup.
    private static var pendingRequest: Task<Bool, Never>?

    static var hasConsent: Bool { UserDefaults.standard.bool(forKey: consentKey) }

    /// Returns true when sharing is allowed, showing the popup first if needed.
    static func requestConsentIfNeeded() async -> Bool {
        if hasConsent { return true }
        if let pendingRequest { return await pendingRequest.value }
        let request = Task { await presentPrompt() }
        pendingRequest = request
        let granted = await request.value
        pendingRequest = nil
        if granted { UserDefaults.standard.set(true, forKey: consentKey) }
        return granted
    }

    private static func presentPrompt() async -> Bool {
        guard let presenter = topViewController() else { return false }
        return await withCheckedContinuation { continuation in
            var resumed = false
            func finish(_ granted: Bool) {
                guard !resumed else { return }
                resumed = true
                continuation.resume(returning: granted)
            }
            let alert = UIAlertController(
                title: "Share with AI?",
                message: "To <do X>, your <data> is sent to <Provider>.",
                preferredStyle: .alert
            )
            let allow = UIAlertAction(title: "Allow", style: .default) { _ in finish(true) }
            alert.addAction(allow)
            alert.addAction(UIAlertAction(title: "Privacy Policy", style: .default) { _ in
                UIApplication.shared.open(privacyPolicyURL)
                finish(false)
            })
            alert.addAction(UIAlertAction(title: "Not Now", style: .cancel) { _ in finish(false) })
            alert.preferredAction = allow
            presenter.present(alert, animated: true)
            // If UIKit refused the presentation, don't leave the caller waiting forever.
            Task { @MainActor in
                try? await Task.sleep(for: .seconds(1))
                if alert.presentingViewController == nil { finish(false) }
            }
        }
    }

    /// Front-most view controller, so the popup also appears above sheets and covers.
    private static func topViewController() -> UIViewController? {
        let scene = UIApplication.shared.connectedScenes
            .compactMap { $0 as? UIWindowScene }
            .first { $0.activationState == .foregroundActive }
        var top = scene?.keyWindow?.rootViewController
        while let presented = top?.presentedViewController, !presented.isBeingDismissed {
            top = presented
        }
        return top
    }
}
```

**Call sites:** add the guard at EVERY entry point, right after the credit/premium check and before any loading animation or network call:
```swift
guard await AIConsentManager.requestConsentIfNeeded() else {
    hasError = true
    errorMessage = "<Feature> needs your permission to send <data> to <Provider>. Tap Try Again to allow it."
    return
}
```
For throwing flows, add an error case (e.g. `case aiConsentDeclined`) and show a specific message for it. Don't fall into the generic "no match" error. Find every entry point by grepping for the service singletons (`XService.shared.`).

**Settings toggle:**
```swift
@AppStorage(AIConsentManager.consentKey) private var hasAIConsent = false
// Row: "Share with AI" + Toggle bound to hasAIConsent, with a caption:
// "Photos, typed text and questions are sent to <Provider> to <purpose>. Turn off to stop sharing. You'll be asked again next time."
```

---

## B. Privacy manifests (DPLA 3.3.3(B))

App target (`<App>/Resources/PrivacyInfo.xcprivacy`). Keep only the API categories and data types you actually found:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>NSPrivacyTracking</key><false/>
	<key>NSPrivacyTrackingDomains</key><array/>
	<key>NSPrivacyAccessedAPITypes</key>
	<array>
		<dict>
			<key>NSPrivacyAccessedAPIType</key><string>NSPrivacyAccessedAPICategoryUserDefaults</string>
			<key>NSPrivacyAccessedAPITypeReasons</key><array><string>CA92.1</string><string>1C8F.1</string></array>
		</dict>
		<dict>
			<key>NSPrivacyAccessedAPIType</key><string>NSPrivacyAccessedAPICategoryFileTimestamp</string>
			<key>NSPrivacyAccessedAPITypeReasons</key><array><string>C617.1</string></array>
		</dict>
	</array>
	<key>NSPrivacyCollectedDataTypes</key>
	<array>
		<!-- one dict per App Privacy label entry -->
		<dict>
			<key>NSPrivacyCollectedDataType</key><string>NSPrivacyCollectedDataTypeUserID</string>
			<key>NSPrivacyCollectedDataTypeLinked</key><true/>
			<key>NSPrivacyCollectedDataTypeTracking</key><false/>
			<key>NSPrivacyCollectedDataTypePurposes</key>
			<array>
				<string>NSPrivacyCollectedDataTypePurposeAppFunctionality</string>
				<string>NSPrivacyCollectedDataTypePurposeAnalytics</string>
			</array>
		</dict>
	</array>
</dict>
</plist>
```
Data type constants include `...Health`, `...PhotosorVideos`, `...OtherUserContent`, `...UserID`, `...DeviceID`, `...PurchaseHistory`, `...ProductInteraction`, `...OtherDiagnosticData`.

Widget or extension: its own file with only its APIs (often UserDefaults `1C8F.1`) and an empty `NSPrivacyCollectedDataTypes`.

**Verify:**
- `plutil -lint` on both files.
- After building, run `find <Build>/Products/... -name PrivacyInfo.xcprivacy`. It must show inside the `.app` and the `.appex`.
- File-system-synchronized Xcode groups (Xcode 16+) pick new files up automatically. Older projects need the file added to the target's Copy Bundle Resources.

---

## C. Remove a marketing notification campaign (4.5.4 / 5.6)

1. Delete the scheduler and every call to it (onboarding, purchase-state updates, app init).
2. Keep the notification permission request if reminders need it: `_ = try? await UNUserNotificationCenter.current().requestAuthorization(options: [.alert, .sound, .badge])` (add `import UserNotifications`).
3. Remove the campaign's branch from the notification delegate.
4. Clean up existing installs on launch:
```swift
private static func removeLegacyCampaignNotifications() {
    Task {
        let center = UNUserNotificationCenter.current()
        let pending = await center.pendingNotificationRequests().map(\.identifier).filter { $0.hasPrefix("engagement_") }
        if !pending.isEmpty { center.removePendingNotificationRequests(withIdentifiers: pending) }
        let delivered = await center.deliveredNotifications().map(\.request.identifier).filter { $0.hasPrefix("engagement_") }
        if !delivered.isEmpty { center.removeDeliveredNotifications(withIdentifiers: delivered) }
    }
}
```
Alternative if the owner wants to keep marketing: fix false copy, add a "Tips & offers" toggle (default off or explicit opt-in), and schedule only when it's on.

---

## D. Remove iCloud/CloudKit sync (5.1.3(ii), DPLA 3.3.3(D))

1. Measure first: CloudKit Console → Usage (unique users, iPad vs iPhone share). Show the user and let them decide.
2. Commit pending unrelated work, create a branch, and build + save the "before" `.app`.
3. Change the `ModelContainer` to local only (`cloudKitDatabase: .none`), keeping the in-memory last resort. Same default store, so data stays.
4. Delete the CloudKit warm-up (dummy insert/delete) and any debug schema initializer. Keep harmless one-time data migrations.
5. Entitlements: remove `com.apple.developer.icloud-container-identifiers` and `com.apple.developer.icloud-services`. Info.plist: remove `remote-notification` from `UIBackgroundModes` if only CloudKit used it. Edit with `plistlib` and check the diff is clean.
6. Upgrade test (before → after) on simulator or device: data kept, new save persists after force-quit, reminders fire.
7. Follow-ups: remove the "iCloud Sync" line from the description, update the privacy policy, and update memory if a CloudKit schema-deploy rule exists.

---

## E. Honest loading copy (2.3.1 / 5.6)

Replace claims of work that never happens. Real examples:

| Before | After |
|---|---|
| Matching with medicine database | Identifying your pill… |
| AI verifying medicine identity | Preparing results… |
| Cross-referencing drug databases | Looking for interactions with AI… |
| Generating safety report | Preparing your report… |

If timers play BEFORE the request, run them DURING it instead and hold on the last step:
```swift
let stepProgress = Task { @MainActor in
    for step in LoadingStep.allCases.dropFirst() {
        try? await Task.sleep(for: .seconds(1.5))
        guard !Task.isCancelled else { return }
        currentLoadingStep = step.rawValue
    }
}
await performRealRequest()
stepProgress.cancel()
```

---

## F. Health data in analytics (5.1.1(ii), 5.1.3)

Keep feature-use events and drop health details: taken/skipped status, missed counts, dose schedules, pill counts, streak lengths, which reminder button was pressed (log `opened` vs `action_button`), and anything from system captures (Visual Intelligence labels). Tell the user which metrics they'll lose. Then update the privacy policy's analytics section ("From version x.y.z, analytics do not include…").

---

## G. Metadata text

**Medical disclaimer for the description** (short version, fits tight limits):
```
MEDICAL DISCLAIMER
<App> uses AI and can make mistakes. For information only, not medical advice. Always confirm with a pharmacist or doctor before taking any medication. In an emergency, call your local emergency number.
```

**Common rewrites** (offer them, and let the owner decide):
- "The Fastest X on the App Store" → "Identify X in Seconds"
- "fast and accurate" → "fast, AI-powered"
- "Make sure your medication is safe today!" → "Learn more about your medications today."
- "helps doctors, nurses, and pharmacists verify medications" → "helps patients and caregivers quickly look up medications"
- "with no limits" → remove it if there's a server cap

**App Review notes template:**
```
What's new in <version>:
- <Each new feature/permission, specifically. E.g. "Before any photo or question is sent to our AI provider (<Provider>), the app asks for permission in a 'Share with AI?' popup. Users can withdraw it in Settings > Share with AI.">
- <Removed features, e.g. "iCloud sync was removed. All data now stays on the device.">
- Privacy manifests added; privacy policy updated: <url>

How to test: <steps>. No login is required. <Free tier details.>
```

**What's New:** 2-4 short, friendly bullets, no technical words, and no implication that things were unsafe before. Example:
```
What's New:
• Faster results when you scan a pill.
• We now ask before sharing your photos or questions with our AI. You can change this anytime in Settings.
• Small fixes and improvements.
```
