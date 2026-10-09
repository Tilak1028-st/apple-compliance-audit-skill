# Apple Compliance Audit: a Claude Code skill

Find what could get your iOS app rejected or removed **before** Apple does.

This skill turns Claude Code into a strict App Review auditor. It reads your code, your App Store Connect metadata and your live privacy policy, checks them against Apple's **current** rules, and writes a report with every issue ranked by severity. Then it fixes only the issues you pick.

Works with SwiftUI, UIKit, React Native, Flutter and Expo apps.

## What it checks

- **App Review Guidelines**: safety, performance, business, design and legal (sections 1 to 5)
- **Third-party AI consent** (5.1.2(i), added Nov 2025): personal data sent to OpenAI, Gemini, Claude and others
- **Apple Developer Program License Agreement**: §3.3.3 and Schedule 2
- **Privacy**: privacy manifests, required-reason APIs, the App Privacy label, purpose strings, account deletion
- **Privacy policy vs reality**: compares every claim in your live policy with your real data flows
- **Subscriptions and paywalls**: renewal terms, restore, manipulative copy, duplicate groups
- **Health and medical apps**: disclaimers, accuracy claims, health data in iCloud or analytics
- **Metadata**: name, subtitle, description claims, age rating, review notes
- **4.3 spam risk**: near-identical apps on the same account

Every finding cites the guideline number plus the exact `file:line` or metadata field.

## How it works

1. **Fetches the official sources fresh** every time, because the guidelines change
2. **Reads your app**: Info.plist, entitlements, dependencies, purchases, AI calls, analytics, notifications
3. **Reads App Store Connect** using the [`asc` CLI](https://github.com/rudrankriyam/App-Store-Connect-CLI), read-only
4. **Writes `APPLE_AUDIT.md`** with Critical, High, Medium and Low findings, then stops
5. **Fixes only what you approve**, builds after each change, and records decisions in the report
6. **Helps with release**: version bump, App Review notes, What's New text

The audit is read-only. Nothing in your code or App Store Connect changes until you say so.

## Install

### Option 1: as a plugin (recommended)

In Claude Code, run:

```
/plugin marketplace add Tilak1028-st/apple-compliance-audit-skill
/plugin install apple-compliance-audit@apple-compliance-audit
```

### Option 2: as a plain skill

```bash
git clone --depth 1 https://github.com/Tilak1028-st/apple-compliance-audit-skill.git /tmp/aca
cp -R /tmp/aca/plugins/apple-compliance-audit/skills/apple-compliance-audit ~/.claude/skills/
```

## Requirements

- [Claude Code](https://claude.com/claude-code)
- **Optional but recommended:** the `asc` CLI for App Store Connect checks
  ```bash
  brew install asc
  asc auth login --name "MyApp" --key-id "KEY_ID" --issuer-id "ISSUER_ID" --private-key /path/to/AuthKey.p8
  ```
  Without it, the skill still audits your code and marks App Store Connect items as "Check manually".
- Xcode, if you want Claude to build and verify the fixes

## Usage

Open your app's project folder in Claude Code and say any of these:

- "Run an Apple audit"
- "Will Apple reject this?"
- "Check App Review guidelines before I submit"

You can also run it directly with `/apple-compliance-audit` (plain skill) or `/apple-compliance-audit:apple-compliance-audit` (plugin).

## What's inside

```
plugins/apple-compliance-audit/skills/apple-compliance-audit/
├── SKILL.md                    # the workflow: audit, report, fix, release
└── references/
    ├── checklist.md            # what to check, with grep recipes
    ├── asc-commands.md         # read-only App Store Connect commands
    ├── fix-recipes.md          # tested fixes (AI consent popup, privacy manifest, more)
    └── gotchas.md              # traps that cost real time
```

The fix recipes and gotchas come from real fixes shipped in a live App Store app.

## Contributing

Found a new rejection reason, a guideline change or a better fix? Pull requests and issues are welcome. Helpful additions:

- New checklist items with the guideline number and a grep recipe
- Fix recipes you have shipped
- Gotchas that cost you a rejection

## Disclaimer

This project is not affiliated with or endorsed by Apple. It is a helper, not legal advice, and it cannot guarantee approval. App Review makes the final call.

## License

MIT © Tilak Shakya
