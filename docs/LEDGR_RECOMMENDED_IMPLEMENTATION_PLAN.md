# LEDGR: Recommended Implementation Plan (Flutter + Kotlin, Local-Only, $0 Cost)

> A privacy-first Android expense tracker that builds your ledger automatically from SMS, email receipts, and wallet notifications. No cloud, no account, no manual logging as the normal case.

**Stack:** Dart/Flutter, Kotlin (native Android), Drift + SQLCipher, Riverpod
**Cost goal:** $0/month. No backend, no paid services. (Only cost if you later publish: the one-time Google Play developer fee, **VERIFY** current amount.)
**Target:** Working MVP in 10 weeks, 2 people (COMSATS CSC303, FA24-BCS-023 and FA24-BCS-013)
**Source:** Semester Project Proposal, September 22, 2026. This plan is the version I think is best: it keeps your vision and changes the order and a few technical choices to reduce the risk of ending the semester with a demo that doesn't work.

---

## Table of Contents

1. [Assumptions & Guiding Principles](#1-assumptions--guiding-principles)
2. [Platform Constraints & Compliance (read this first)](#2-platform-constraints--compliance-read-this-first)
3. [Architecture](#3-architecture)
4. [Final Tech Stack](#4-final-tech-stack)
5. [Phase 0: Requirements & Message Corpus](#phase-0-requirements--message-corpus-weeks-1-2)
6. [Phase 1: Project Setup, Database & Security Shell](#phase-1-project-setup-database--security-shell-weeks-2-3)
7. [Phase 2: Parser Engine](#phase-2-parser-engine-weeks-3-4)
8. [Phase 3: SMS Ingestion (Native)](#phase-3-sms-ingestion-native-weeks-4-5)
9. [Phase 4: Core UI](#phase-4-core-ui-weeks-4-7)
10. [Phase 5: Email Ingestion (IMAP)](#phase-5-email-ingestion-imap-weeks-6-7)
11. [Phase 6: Analytics, Budgets & Export](#phase-6-analytics-budgets--export-weeks-7-8)
12. [Phase 7: Third Channel (Notification Listener / Accessibility)](#phase-7-third-channel-notification-listener--accessibility-weeks-8-9)
13. [Phase 8: Hardening, Testing & Release](#phase-8-hardening-testing--release-weeks-9-10)
14. [Phase 9: Future Scope](#phase-9-future-scope-post-semester)
15. [Testing, Security, Performance, Legal](#testing-security-performance-legal)
16. [Risks & Mitigations](#risks--mitigations)
17. [Week-by-Week Checklist](#week-by-week-checklist)
18. [Appendix: Code & Config Snippets](#appendix-code--config-snippets)

---

## 1. Assumptions & Guiding Principles

**Assumptions**
- Two students, about 10 weeks, no budget.
- **Android only.** iOS does not allow reading SMS or other apps' screens. Flutter is chosen for fast UI and good libraries, not for cross-platform reach.
- The demo runs from a sideloaded APK (not the Play Store). See Section 2 for why this matters.
- Real SMS and email samples come from your own phones and accounts plus synthetic test data. Never commit real personal financial data to GitHub.
- Wallets in scope: JazzCash, Easypaisa, SadaPay, NayaPay, plus one or two commercial banks. Start with the two or three you personally use, so you can verify against reality.

**Principles**
1. **Local-only, provably.** No analytics, no crash reporters that upload data, no cloud backend. This is the project's identity.
2. **Ingestion is a pipeline of small, pure steps.** `raw event → normalise → parse → dedupe → categorise → write`. Each step is a plain Dart function with tests.
3. **The parser is data plus a tiny engine.** Templates live in JSON files; adding a bank means adding a template and test cases, not code.
4. **Never lose data.** Store the raw payload for every event, make writes idempotent, and keep unparsed messages in an "Needs review" queue instead of dropping them.
5. **Manual entry is a first-class fallback.** Parsers will miss messages (templates change, new merchants). Users must be able to add and fix transactions in two taps.
6. **Money is an integer.** Store minor units (paisa) as `int`. Never use `double` for amounts.
7. **Native only where required.** Kotlin for the SMS receiver, notification listener, and accessibility service. Everything else is Dart.
8. **Least permission, enabled progressively.** Ask for each permission when the user turns on the matching channel, with a plain-language explanation.
9. **Accuracy is the brand.** If the dashboard balance disagrees with the wallet app, users stop trusting it. Reconcile using balances found in messages.

### 1.1 Where this plan deliberately differs from the proposal

| Proposal says | This plan does | Why |
|---|---|---|
| "Zero manual data entry" | Automatic first, **manual add/edit always available** | Regex templates will miss some messages. Without a manual fallback, one miss makes the ledger wrong forever. |
| Accessibility screen-reading as one of three equal channels | **Notification listener** as the preferred third channel; accessibility is a stretch goal | Accessibility is the most fragile and most policy-restricted option, some banking apps block or misbehave when an accessibility service is active, and screens marked `FLAG_SECURE` can't be read. Notification listening gets much of the same data with far less risk. **VERIFY** on your target apps. |
| `telephony` package for SMS | **Own small Kotlin `BroadcastReceiver`** plus a method/event channel | The receiver is about 60 lines, you control multipart handling and queuing, and you avoid depending on a package that may be unmaintained. **VERIFY** package status; if it is healthy, you can still use it. |
| Email via IMAP IDLE "always on" | **Periodic WorkManager sync** as the baseline; IDLE only while the app is open | Android Doze and OEM battery managers kill long-lived sockets. Periodic sync is reliable; IDLE is a bonus. |
| Phases: UI design weeks 3-4, database weeks 5-7, ingestion after | **Corpus and parser first, UI in parallel** | The parser is the riskiest and most valuable part. Find out in week 4, not week 9, whether it works. |
| (Not mentioned) | Cross-channel **deduplication** | The same payment often arrives by SMS and email (and notification). Without dedup the ledger double-counts. |
| (Not mentioned) | Review queue, reconciliation, test corpus | These are what make an auto-tracker trustworthy and give you a strong evaluation story. |

---

## 2. Platform Constraints & Compliance (read this first)

> Android permission rules and store policies change often. Before building each channel, re-check the current Android docs and Google Play policy. Items marked **VERIFY** are the ones most likely to have moved.

### 2.1 Capability and permission reality check

| Capability | API / permission | Gotchas (VERIFY) |
|---|---|---|
| Read incoming SMS | `RECEIVE_SMS` (runtime permission) | A receiver declared in the manifest fires even when the app is not running, so **the SMS listener does not need a foreground service**. On Android 13+, sideloaded apps may need the user to allow "restricted settings" in App info before the permission can be granted. |
| Read SMS history (initial backfill) | `READ_SMS` | Same restrictions. Needed only if you want to import old messages; make it optional. |
| Background email sync | WorkManager periodic work | Minimum period is 15 minutes. Needs `INTERNET`. |
| Live email (IDLE) | Foreground service + `INTERNET` | Android 14+ requires a declared foreground service type; some types have time limits on newer Android versions. Use only while the app is open or as an opt-in. |
| Wallet app notifications | `NotificationListenerService` (user enables in system settings) | Receives **all** notifications. Filter by an allow-list of package names immediately and discard everything else without logging or storing it. |
| Screen reading | `AccessibilityService` | Google Play restricts this API heavily. Some banking apps detect it and refuse to work. `FLAG_SECURE` screens may return nothing. Treat as stretch goal. |
| Biometric / PIN | `local_auth` | Use `BiometricPrompt` with device-credential fallback. |
| Battery / autostart | OEM settings | Xiaomi, Oppo, Vivo, Infinix, Tecno and similar phones (very common in Pakistan) can kill background work. Add an onboarding step with per-brand guidance (see dontkillmyapp.com). |

### 2.2 Distribution decision

- **For the course:** build a signed release APK and install it directly. SMS and accessibility permissions are fine for your own devices and for a demo.
- **If you publish later:** Google Play restricts SMS permissions (generally to default SMS apps and a short list of approved exceptions) and tightly controls `AccessibilityService`. **VERIFY** the current policies and the permission-declaration process before investing in publishing. Plan to publish elsewhere (GitHub Releases, F-Droid-style distribution) or to remove the restricted channels from a Play build.

### 2.3 The privacy contract (what LEDGR promises, and how to prove it)

- No analytics SDKs, no ads, no crash uploaders.
- All data in an AES-256 encrypted SQLite file (SQLCipher). The key is random, generated on first run, and stored via Android Keystore-backed `flutter_secure_storage`.
- `android:allowBackup="false"` and data-extraction rules that exclude the app from cloud backup, otherwise the "local-only" claim is weakened.
- `FLAG_SECURE` on the app window to hide content from screenshots and the recent-apps preview.
- **Optional, strong:** two Gradle product flavors. `core` has **no `INTERNET` permission at all** (SMS and notification channels only), `email` adds `INTERNET` for IMAP. Publishing the `core` APK's manifest is a verifiable privacy claim.
- Documented in an in-app "What LEDGR does with your data" screen and a short `PRIVACY.md`.

---

## 3. Architecture

### 3.1 Ingestion pipeline

```
 ┌────────────────┐  ┌──────────────────┐  ┌───────────────────────┐
 │ SMS receiver   │  │ IMAP sync        │  │ Notification listener │   (Accessibility = stretch)
 │ (Kotlin)       │  │ (Dart/WorkManager)│  │ (Kotlin)              │
 └───────┬────────┘  └────────┬─────────┘  └───────────┬───────────┘
         └──────────────┬─────┴────────────────────────┘
                        ▼
              RawEvent {source, sender, body, receivedAt}      ← always stored first (raw_events)
                        ▼
              Normaliser  (trim, unify digits/currency, strip footers)
                        ▼
              ParserRegistry → pick templates by sender/package → first match wins
                        ▼
        ┌───────────────┴───────────────┐
   ParsedTxn                        no match → status = needs_review
        ▼
   Deduper (fingerprint, ±5 min window across channels)
        ▼
   Categoriser (merchant rules → category)
        ▼
   LedgerWriter (single Drift transaction; updates wallet balance snapshot)
        ▼
   Riverpod streams → Dashboard / Transactions / Analytics
```

### 3.2 Layers and folder structure

```
ledgr/
├── android/app/src/main/kotlin/.../
│   ├── sms/SmsReceiver.kt, PendingQueue.kt
│   ├── notif/WalletNotificationListener.kt
│   ├── a11y/BankScreenService.kt            (stretch)
│   └── Bridge.kt                             (MethodChannel / EventChannel)
├── lib/
│   ├── core/            money.dart, result.dart, clock.dart, logger.dart
│   ├── data/
│   │   ├── db/          tables.dart, app_database.dart, daos/
│   │   ├── security/    key_store.dart, app_lock.dart
│   │   └── native/      native_bridge.dart
│   ├── domain/
│   │   ├── ingest/      raw_event.dart, normaliser.dart, parser_engine.dart,
│   │   │                dedupe.dart, categoriser.dart, ingest_service.dart
│   │   ├── models/      wallet.dart, txn.dart, budget.dart
│   │   └── reports/     cashflow.dart, category_totals.dart
│   ├── features/        lock/, dashboard/, transactions/, analytics/, sync_settings/, review/
│   └── main.dart
├── assets/parsers/      easypaisa.json, jazzcash.json, sadapay.json, nayapay.json, banks/
├── test/
│   ├── corpus/          *.jsonl (message → expected fields), golden test runner
│   └── unit/ widget/
├── integration_test/
└── .github/workflows/ci.yml
```

### 3.3 Data model (Drift)

| Table | Key columns | Notes |
|---|---|---|
| `wallets` | id, name, kind (wallet/bank/cash), providerKey, openingBalanceMinor, lastKnownBalanceMinor, lastKnownBalanceAt | `lastKnown*` comes from balances found in messages and powers reconciliation |
| `transactions` | id, walletId, direction (debit/credit/transfer), amountMinor, currency, counterparty, categoryId, occurredAt, source, rawEventId, fingerprint (unique), note, status (confirmed/needs_review), createdAt | `fingerprint` unique index gives idempotent writes |
| `raw_events` | id, source (sms/email/notif/a11y/manual), sender, body, receivedAt, processedAt, templateId, parseStatus | Keep raw text so templates can be fixed and replayed. Purge after N days by user setting. |
| `categories` | id, name, icon, kind (expense/income), isDefault | Seed Dining, Transport, Utilities, Shopping, Transfers, Income, Other |
| `category_rules` | id, pattern, matchType (contains/regex), categoryId, priority | User-editable; "always categorise X as Y" creates a rule |
| `budgets` | id, categoryId, month (yyyy-mm), limitMinor | One row per category per month |
| `sync_state` | channel, lastUid/lastRunAt, enabled | IMAP last seen UID per account |
| `settings` | key, value | Currency, theme, retention days |

### 3.4 Money and time rules
- Amount = `int` paisa. `Money` value class with `format()` and `parse()`.
- Store timestamps as UTC epoch milliseconds; display in device timezone. Message time beats receive time when parseable.

### 3.5 Idempotency and dedupe
- Fingerprint = `sha256(walletId | direction | amountMinor | reference-or-(counterparty + 5-minute bucket))`.
- If a bank/wallet message has a transaction ID, use that in the fingerprint (most reliable).
- When two events fall into the same window with the same amount, merge them: keep the richer record and link both raw events.

### 3.6 SMS to Dart bridge (decision record)

| Option | Pros | Cons |
|---|---|---|
| **A. Native queue, Dart drains it (recommended for MVP)** | No lost events, simple, works when the app isn't running | Parsing happens when Dart next runs (app open or a WorkManager tick) |
| B. Receiver starts a headless Flutter engine and parses immediately | Instant | More complex, harder to debug, memory cost |

Start with A: the receiver appends the SMS to a small app-private queue (encrypted with a Keystore key) and posts a WorkManager job; Dart drains the queue, writes `raw_events`, and runs the pipeline. Upgrade to B only if the demo needs instant updates while the app is closed.

---

## 4. Final Tech Stack

| Layer | Choice | Notes |
|---|---|---|
| Language | Dart (UI, logic), Kotlin (native services) | Strict analysis options on |
| Framework | Flutter, Android target only | Material 3 |
| State | **Riverpod** | Providers wrap DAO streams |
| Navigation | go_router | Lock screen as a redirect guard |
| Database | **drift** + **sqlcipher_flutter_libs** | AES-256 at rest; migrations via drift |
| Key storage | **flutter_secure_storage** | Keystore-backed |
| App lock | **local_auth** | Biometric + device PIN fallback |
| SMS | Kotlin `BroadcastReceiver` (or `telephony` fork if healthy, **VERIFY**) | Permission via `permission_handler` |
| Email | **enough_mail** (IMAP) + **workmanager** | Periodic sync; IDLE optional |
| Foreground work | **flutter_foreground_task** | Only for IDLE or long sync |
| Charts | **fl_chart** | Pie, line, bar |
| Export | `csv`, `pdf`, `printing`, `share_plus` | CSV and PDF reports |
| Crypto | `crypto` (hashes) | Fingerprints |
| Root check | `flutter_jailbreak_detection` or equivalent (**VERIFY** maintenance) | Warning only, not a hard block |
| Testing | `flutter_test`, `mocktail`, `integration_test` | Golden corpus tests |
| Tooling | VS Code, Android Studio, `very_good_analysis` or `flutter_lints` | |
| CI/CD | GitHub Actions | Analyze, test, build APK artifact |
| Version control | Git + GitHub, trunk-based with short feature branches | |

---

## Phase 0: Requirements & Message Corpus (Weeks 1-2)

**Goal:** know exactly what messages look like before writing any parser.

### Tasks
- [ ] **Collect real samples** from your own phones: SMS, email receipts, wallet notifications. Aim for **30+ per provider** and cover: money sent, money received, bill payment, mobile top-up, ATM withdrawal, failed transaction, reversal, fee, and balance inquiry.
- [ ] **Anonymise** before committing: replace names, account numbers, phone numbers, and transaction IDs with realistic fakes. Keep the structure identical.
- [ ] **Record sender identifiers** (shortcodes and names) for each provider from your own inboxes. The shortcodes listed in the proposal (3737, 8558, 8274) must be confirmed against real messages, not assumed.
- [ ] **Write a message taxonomy:** one table per provider listing message types and the fields each contains (amount, direction, counterparty, reference, balance, timestamp, fee).
- [ ] **Define the normalised transaction schema** (the `ParsedTxn` fields above) and what "needs review" means.
- [ ] **Requirements list** with priority (Must/Should/Could): lock, dashboard, SMS channel, manual add, categories, budgets, analytics, export, email channel, notification channel, accessibility.
- [ ] **Define success metrics** you will report in the final presentation: parse rate (% of corpus parsed correctly), false-positive rate (non-transaction SMS wrongly parsed), dedupe accuracy, ingest latency, and app cold-start time.
- [ ] **Low-fidelity wireframes** for the five screens in the proposal plus a "Needs review" screen.

### Deliverables
- Anonymised corpus (`test/corpus/*.jsonl`) with expected outputs
- Message taxonomy document
- Requirements list with MoSCoW priorities
- Wireframes

---

## Phase 1: Project Setup, Database & Security Shell (Weeks 2-3)

### 1.1 Repo and tooling

```bash
flutter create --platforms=android --org pk.edu.comsats ledgr
cd ledgr
flutter pub add flutter_riverpod go_router drift sqlcipher_flutter_libs path_provider path \
  flutter_secure_storage local_auth crypto fl_chart intl permission_handler workmanager
flutter pub add --dev drift_dev build_runner flutter_lints mocktail
dart run build_runner build
```

Pin the versions that resolve at install time and commit `pubspec.lock`.

### 1.2 Tasks
- [ ] Folder structure from Section 3.2; lint rules enabled; CI running analyze + test on every push.
- [ ] Drift schema for all tables in 3.3 with a first migration and a migration test.
- [ ] **Encrypted database open** using a random 256-bit key stored in `flutter_secure_storage` (see appendix). Add a test or debug check that the DB file is not readable as plain SQLite.
- [ ] `Money` class and tests (parse "Rs. 1,250.50" → 125050).
- [ ] **Lock screen:** `local_auth` with PIN fallback; lock on app background after a timeout; `FLAG_SECURE`.
- [ ] Manifest hardening: `allowBackup=false`, data extraction rules, no `INTERNET` in the `core` flavor if you adopt flavors.
- [ ] Seed default categories and wallets.

### Deliverables
- App boots to a lock screen, then an empty dashboard, with an encrypted database
- CI green; DB migration test passing

---

## Phase 2: Parser Engine (Weeks 3-4)

**The most important phase. Do it test-first.**

### 2.1 Template format (JSON)

```json
{
  "id": "provider.sent.v1",
  "senders": ["PLACEHOLDER_SENDER"],
  "kind": "debit",
  "pattern": "(?i)sent\\s+rs\\.?\\s*(?<amount>[\\d,]+(?:\\.\\d{1,2})?)\\s+to\\s+(?<counterparty>.+?)\\s+on",
  "fields": { "amount": "amount", "counterparty": "counterparty" },
  "balancePattern": "(?i)balance[:\\s]+rs\\.?\\s*(?<balance>[\\d,]+(?:\\.\\d{1,2})?)"
}
```

> The pattern above is **illustrative only**. Real message wording varies by provider and changes over time. Write each pattern against your anonymised corpus, never from memory.

### 2.2 Tasks
- [ ] `Normaliser`: collapse whitespace, normalise currency markers (Rs, Rs., PKR), convert Eastern Arabic digits if present, strip promotional footers.
- [ ] `ParserEngine`: load templates from assets, index by sender/package, try templates in priority order, return `ParsedTxn` or `null`.
- [ ] **Non-transaction filter:** OTP codes, promotions, and balance-only messages must **not** create transactions (separate "ignore" templates make this explicit and testable).
- [ ] **Corpus test runner:** reads every `*.jsonl` line and asserts the expected fields. CI fails if parse rate drops below your threshold.
- [ ] Extract balance when present and update `wallets.lastKnownBalance*`.
- [ ] `Deduper` and `Categoriser` with tests (see 3.5 and Appendix I).
- [ ] `IngestService.process(RawEvent)` orchestrating the pipeline with a single DB transaction per event.
- [ ] Replay tool: a debug screen or CLI that re-runs the pipeline on stored raw events after you fix a template.

### Deliverables
- Parser passing the corpus for at least two providers (target 95%+ on the corpus)
- Zero OTP or promo messages creating transactions in the test set
- Documented process for adding a new provider in under 30 minutes

---

## Phase 3: SMS Ingestion (Native) (Weeks 4-5)

### Tasks
- [ ] Kotlin `SmsReceiver` for `SMS_RECEIVED`; concatenate multipart messages; filter to the sender allow-list early (see appendix C).
- [ ] `PendingQueue` in app-private storage, encrypted with a Keystore AES-GCM key; atomic append and drain.
- [ ] Bridge: `MethodChannel` to drain the queue and `EventChannel` to push events when the Dart side is alive.
- [ ] WorkManager job to drain the queue periodically and on boot.
- [ ] Permission onboarding screen: explain what is read, that non-wallet SMS is ignored, and how to handle the Android 13+ "restricted settings" step for sideloaded installs. **VERIFY** on a real device.
- [ ] **Backfill** (optional): read the last N days of the inbox once with explicit consent to seed history.
- [ ] Device test matrix: at least one Samsung and one Xiaomi/Infinix/Tecno-class phone, Android 12 through the newest version you can test.
- [ ] Battery/autostart guidance screen with per-OEM instructions.

### Deliverables
- Send yourself a real payment (or replay an SMS on a test device) and see it appear in the ledger with the app closed
- Kill-and-reboot test: no events lost

---

## Phase 4: Core UI (Weeks 4-7)

Built in parallel with Phases 2-3 by the other team member.

### 4.1 Screens

| Screen | Contents |
|---|---|
| Lock | Branding, biometric button, PIN pad |
| Dashboard | Wallet carousel, total balance, month spend summary, recent activity with source icons (SMS/email/notification/manual) |
| Transactions | Filter by wallet, category, date, source; search; swipe to edit or delete |
| Transaction detail | Parsed fields, raw payload view, category picker, merchant editor, notes, "Always categorise this merchant as…" rule |
| Needs review | Queue of unparsed or low-confidence events with "Add manually" and "Ignore" actions |
| Manual add | Amount, wallet, direction, category, date, note (two-tap fast path) |
| Wallets | Create, rename, set opening balance, reconcile against last-known balance |
| Sync settings | Channel toggles, permission status, IMAP accounts, last sync times |

### Tasks
- [ ] Theme, dark mode, and an empty-state design for every list.
- [ ] Riverpod providers over Drift streams so the UI updates live when an ingest completes.
- [ ] Accessibility basics: text scaling, contrast, semantic labels.
- [ ] Widget tests for the dashboard, transaction detail, and manual add.

### Deliverables
- Fully usable manual ledger (even before any ingestion works) with live updates from ingestion

---

## Phase 5: Email Ingestion (IMAP) (Weeks 6-7)

### Tasks
- [ ] Account setup screen: host, port, SSL, username, **app password**. **VERIFY** what current Gmail and other providers require (many now require app passwords or OAuth; OAuth for a published app adds verification work). Never store the password outside `flutter_secure_storage`.
- [ ] Windowed sync: remember the last seen UID per account; fetch only newer messages from allow-listed sender addresses; extract the plain-text part (fall back to stripped HTML).
- [ ] Run on a WorkManager periodic schedule (minimum 15 minutes). Optional IDLE while the app is open or behind an opt-in foreground service. **VERIFY** foreground-service type and limits for your target Android version.
- [ ] Email templates for SadaPay/NayaPay receipts, tested against anonymised samples.
- [ ] Handle failures cleanly: bad credentials, timeouts, TLS errors, and flaky networks with exponential backoff and a visible status.
- [ ] Cross-channel dedupe test: the same payment arriving by SMS and email produces one transaction.

### Deliverables
- A SadaPay or NayaPay email receipt becomes a transaction without opening the app
- Dedupe test passing

---

## Phase 6: Analytics, Budgets & Export (Weeks 7-8)

### Tasks
- [ ] Reports as pure functions over transactions: `categoryTotals(range)`, `cashFlowByDay/Week/Month(range)`, `walletBalances()`.
- [ ] Charts: category pie, cash-flow line, monthly trend bar, wallet balance cards. Date-range selector.
- [ ] Budgets: set a monthly limit per category; progress bars; warning at 80% and 100% via a local notification.
- [ ] Reconciliation view: computed balance vs last-known balance from messages, with a "difference" explanation and one-tap "adjust".
- [ ] **Export:** CSV (all fields) and PDF monthly report; share through the system share sheet. Warn the user that exported files are not encrypted.
- [ ] Performance check with a synthetic dataset of 50,000 transactions.

### Deliverables
- Analytics screen matching the proposal, with budgets and export working

---

## Phase 7: Third Channel (Notification Listener / Accessibility) (Weeks 8-9)

### 7.1 Notification listener (recommended)
- [ ] `NotificationListenerService` with an **allow-list of wallet/bank package names**. Anything else is dropped immediately and never stored.
- [ ] Extract title and text; send through the same `RawEvent` pipeline with `source = notif`.
- [ ] Settings screen: explain clearly that granting access lets Android show LEDGR all notifications, that LEDGR ignores everything except the allow-list, and how to revoke.
- [ ] Dedupe against SMS for the same payment.

### 7.2 Accessibility service (stretch, only if 7.1 is done and time remains)
- [ ] `AccessibilityService` scoped by `packageNames` in its config to your target banking apps only.
- [ ] Detect "transaction successful" screens by view IDs/text from the node tree; send the text through the same pipeline with `source = a11y`.
- [ ] Test against each target app. Document any app that blocks the service or hides content.
- [ ] Add a clear in-app disclosure and consent screen. **VERIFY** current Play policy before any Play submission.

### Deliverables
- At least one wallet's notifications ingested end to end
- A short written comparison (reliability, privacy, effort) of SMS vs notification vs accessibility; this is a good evaluation chapter

---

## Phase 8: Hardening, Testing & Release (Weeks 9-10)

### Tasks
- [ ] Biometric/PIN edge cases: enrolment changed, too many failures, app killed while locked.
- [ ] Root/emulator warning (non-blocking), with honest wording about its limits.
- [ ] Edge-case parsing test sweep: amounts with commas, decimals, negative balances, multi-line messages, Urdu or Roman-Urdu text if relevant, very long messages, duplicate delivery, out-of-order delivery.
- [ ] Release build: R8 shrink/obfuscation, `--split-debug-info`, `--obfuscate`; verify the parser asset files still load in release (a common obfuscation bug).
- [ ] Signing config and keystore backup (store the keystore **outside** the repo).
- [ ] Security review against the checklist below.
- [ ] Demo script with pre-seeded safe data plus one live SMS replay; record a backup video in case something fails on the day.
- [ ] Final report and presentation: problem, architecture diagram, ingestion pipeline, parse-rate metrics, privacy claims, limitations, future work.

### Deliverables
- Signed release APK, demo video, final report, slides, README with build instructions

---

## Phase 9: Future Scope (post-semester)

- On-device OCR for paper receipts (`google_mlkit_text_recognition`), feeding the same pipeline with `source = ocr`.
- Encrypted local peer-to-peer or file-based backup and restore for device migration (passphrase-derived key, e.g. Argon2/scrypt; never reuse the DB key directly).
- Parser template updates distributed as signed JSON files (so new formats don't need an app update), verified before use.
- Split bills, recurring-transaction detection, and subscription tracking.
- Optional Urdu UI.

---

## Testing, Security, Performance, Legal

### Testing strategy
| Level | What | Tooling |
|---|---|---|
| Unit | Money, normaliser, parser, dedupe, categoriser, reports | `flutter_test` |
| Corpus (golden) | Every anonymised message → expected fields | JSONL runner in CI |
| Integration | Ingest → DB → provider updates; migrations | `integration_test`, in-memory Drift |
| Widget | Dashboard, add/edit, review queue | `flutter_test` |
| Device | SMS receipt with app killed, reboot, Doze, OEM battery managers | Manual matrix |
| Security | Encrypted DB check, backup flags, permission scope | Manual checklist |

### Security checklist
- [ ] DB encrypted; key random; key in Keystore-backed storage; no keys in source.
- [ ] `allowBackup=false` and extraction rules set; `FLAG_SECURE` enabled.
- [ ] No sensitive data in logs (disable verbose logging in release; never log message bodies).
- [ ] Notification listener and accessibility service restricted to allow-listed packages; everything else discarded.
- [ ] IMAP credentials only in secure storage; TLS enforced; certificate errors never ignored.
- [ ] Exported files clearly labelled as unencrypted.
- [ ] Debug/test corpus contains only anonymised data.
- [ ] Dependencies reviewed and pinned; lockfile committed.

### Performance targets
- Ingest of a single event under 200 ms from queue drain to ledger write.
- Dashboard opens under 1 s with 50,000 transactions (use indexes on `occurredAt`, `walletId`, `categoryId`).
- Background work does not drain noticeable battery (check Android's battery stats during the test week).

### Legal and ethics
- Read the raw text of other people's messages only for allow-listed senders; document this.
- Local-only design keeps personal data on the device, but **VERIFY** current Pakistani data-protection requirements if you ever distribute beyond the course.
- Do not publish real transaction data, real account numbers, or real phone numbers anywhere, including screenshots in the report.
- Dependencies keep their licences; add a licences screen.

---

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Message formats change or differ from your samples | High | High | Corpus tests, "Needs review" queue, replay tool, manual entry fallback |
| OEM battery managers kill background work | High | Medium | Manifest receiver (survives app death), WorkManager, onboarding guidance, device test matrix |
| Android 13+ restricted settings block permission on sideloaded APK | Medium | High | Document the steps in onboarding; test on real devices early (week 4) |
| Accessibility service blocked by banking apps or by policy | High | Medium | Notification listener as the primary third channel; accessibility is stretch |
| Gmail/IMAP authentication requirements change | Medium | Medium | App-password path, clear error states, email is one channel of three |
| Double counting across channels | High | High | Fingerprint + time-window dedupe with tests |
| Wrong balances erode trust | Medium | High | Opening balance, reconciliation against message balances, adjust action |
| Scope too big for 10 weeks | High | High | Cut order below; manual ledger plus SMS plus analytics is a complete project |
| Sensitive samples leak into git | Medium | High | Anonymise before commit; `.gitignore` for raw exports; pre-commit check |
| Release obfuscation breaks assets/reflection | Medium | Medium | Test the release APK from week 8, not the night before |

**Scope-cut ladder (cut from the top first):**
1. Accessibility service (7.2)
2. IMAP IDLE (keep periodic sync)
3. Root detection
4. PDF export (keep CSV)
5. Notification listener
6. Email channel
7. Never cut: encrypted DB, SMS channel, parser corpus tests, manual add, dashboard, categories, basic analytics.

---

## Week-by-Week Checklist

Suggested split: **Member A (FA24-BCS-023):** native Kotlin, ingestion, security. **Member B (FA24-BCS-013):** corpus, parser templates, UI, analytics. Swap if you prefer; pair on the parser engine design.

| Week | Goals | Done when |
|---|---|---|
| 1 | Collect and anonymise samples; message taxonomy; requirements list | Corpus has 20+ messages per provider |
| 2 | Wireframes; repo, CI, folder structure; Drift schema; encrypted DB | App boots; DB encrypted; CI green |
| 3 | `Money`, normaliser, parser engine skeleton; lock screen; seed data | First provider parsed in tests; lock works |
| 4 | Templates for 2+ providers; dedupe; categoriser; Kotlin SMS receiver started; dashboard UI started | Corpus parse rate reported |
| 5 | Native queue + bridge; WorkManager drain; permission onboarding; transactions list + manual add | Real SMS appears with app closed |
| 6 | Review queue; wallet management; IMAP account setup and sync | First email-sourced transaction |
| 7 | Email templates; cross-channel dedupe; analytics reports and charts | Dedupe test green; charts render |
| 8 | Budgets and alerts; reconciliation; CSV/PDF export; notification listener | Budget alert fires; export opens |
| 9 | Notification channel finished; (stretch) accessibility; edge-case parsing sweep; security hardening | Security checklist ticked |
| 10 | Release build tested; demo script; report; slides; backup demo video | Signed APK, report, and demo ready |

---

## Appendix: Code & Config Snippets

> Sketches to start from. Check each against the current docs for the package versions you install.

### A. `AndroidManifest.xml` fragments

```xml
<uses-permission android:name="android.permission.RECEIVE_SMS" />
<!-- Optional backfill only: -->
<uses-permission android:name="android.permission.READ_SMS" />
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
<!-- 'email' flavor only: -->
<uses-permission android:name="android.permission.INTERNET" />

<application
    android:allowBackup="false"
    android:dataExtractionRules="@xml/data_extraction_rules"
    android:fullBackupContent="false" ...>

    <receiver
        android:name=".sms.SmsReceiver"
        android:exported="true"
        android:permission="android.permission.BROADCAST_SMS">
        <intent-filter android:priority="999">
            <action android:name="android.provider.Telephony.SMS_RECEIVED" />
        </intent-filter>
    </receiver>

    <service
        android:name=".notif.WalletNotificationListener"
        android:exported="true"
        android:label="LEDGR notification reader"
        android:permission="android.permission.BIND_NOTIFICATION_LISTENER_SERVICE">
        <intent-filter>
            <action android:name="android.service.notification.NotificationListenerService" />
        </intent-filter>
    </service>
</application>
```

### B. Secure window (Kotlin, in `MainActivity`)

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    window.setFlags(WindowManager.LayoutParams.FLAG_SECURE, WindowManager.LayoutParams.FLAG_SECURE)
}
```

### C. SMS receiver with early filtering (Kotlin sketch)

```kotlin
class SmsReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action != Telephony.Sms.Intents.SMS_RECEIVED_ACTION) return
        val parts = Telephony.Sms.Intents.getMessagesFromIntent(intent) ?: return

        // Multipart messages arrive as several PDUs from the same sender.
        val bySender = parts.groupBy { it.originatingAddress ?: "" }
        for ((sender, msgs) in bySender) {
            if (!SenderAllowList.isAllowed(context, sender)) continue   // drop non-wallet SMS now
            val body = msgs.joinToString("") { it.messageBody ?: "" }
            val ts = msgs.first().timestampMillis
            PendingQueue.append(context, QueuedEvent("sms", sender, body, ts))
        }
        DrainWorker.enqueue(context)   // WorkManager one-time job
    }
}
```

`SenderAllowList` reads the sender IDs the user enabled in settings (stored natively so the receiver doesn't need Dart). `PendingQueue.append` writes an encrypted, length-prefixed record to an app-private file with an atomic rename or file lock.

### D. Dart side of the bridge

```dart
class NativeBridge {
  static const _method = MethodChannel('ledgr/native');
  static const _events = EventChannel('ledgr/events');

  Future<List<RawEvent>> drainQueue() async {
    final list = await _method.invokeListMethod<Map>('drainQueue') ?? [];
    return list.map((m) => RawEvent(
      source: m['source'] as String,
      sender: m['sender'] as String,
      body: m['body'] as String,
      receivedAt: DateTime.fromMillisecondsSinceEpoch(m['ts'] as int, isUtc: true),
    )).toList();
  }

  Stream<RawEvent> get live => _events.receiveBroadcastStream().map((m) { /* same mapping */ });
}
```

### E. Drift schema (excerpt)

```dart
class Wallets extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get name => text()();
  TextColumn get kind => text()();                       // wallet | bank | cash
  TextColumn get providerKey => text().nullable()();     // e.g. 'sadapay'
  IntColumn get openingBalanceMinor => integer().withDefault(const Constant(0))();
  IntColumn get lastKnownBalanceMinor => integer().nullable()();
  DateTimeColumn get lastKnownBalanceAt => dateTime().nullable()();
}

class Transactions extends Table {
  IntColumn get id => integer().autoIncrement()();
  IntColumn get walletId => integer().references(Wallets, #id)();
  TextColumn get direction => text()();                  // debit | credit | transfer
  IntColumn get amountMinor => integer()();
  TextColumn get currency => text().withDefault(const Constant('PKR'))();
  TextColumn get counterparty => text().nullable()();
  IntColumn get categoryId => integer().nullable().references(Categories, #id)();
  DateTimeColumn get occurredAt => dateTime()();
  TextColumn get source => text()();                     // sms | email | notif | a11y | manual
  IntColumn get rawEventId => integer().nullable().references(RawEvents, #id)();
  TextColumn get fingerprint => text().unique()();
  TextColumn get note => text().nullable()();
  TextColumn get status => text().withDefault(const Constant('confirmed'))();

  @override
  List<Set<Column>> get customIndexes => []; // add indexes on (occurredAt), (walletId, occurredAt) via @TableIndex
}
```

### F. Opening the encrypted database (Dart sketch)

```dart
import 'dart:io';
import 'dart:math';
import 'package:drift/drift.dart';
import 'package:drift/native.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:path/path.dart' as p;
import 'package:path_provider/path_provider.dart';
import 'package:sqlcipher_flutter_libs/sqlcipher_flutter_libs.dart';
import 'package:sqlite3/open.dart';

Future<String> _dbKeyHex() async {
  const store = FlutterSecureStorage();
  var key = await store.read(key: 'db_key_v1');
  if (key == null) {
    final r = Random.secure();
    key = List.generate(32, (_) => r.nextInt(256).toRadixString(16).padLeft(2, '0')).join();
    await store.write(key: 'db_key_v1', value: key);
  }
  return key;
}

LazyDatabase openEncrypted() => LazyDatabase(() async {
  open.overrideFor(OperatingSystem.android, openCipherOnAndroid);
  await applyWorkaroundToOpenSqlCipherOnOldAndroidVersions();

  final keyHex = await _dbKeyHex();
  final dir = await getApplicationDocumentsDirectory();
  final file = File(p.join(dir.path, 'ledgr.db'));

  return NativeDatabase.createInBackground(file, setup: (db) {
    // Fail loudly if SQLCipher is not actually loaded.
    assert(db.select('PRAGMA cipher_version;').isNotEmpty, 'SQLCipher not available');
    db.execute("PRAGMA key = \"x'$keyHex'\";");
  });
});
```

### G. Money, template parser, and corpus test

```dart
class Money {
  final int minor;                         // paisa
  const Money(this.minor);

  static Money? tryParse(String s) {
    final cleaned = s.replaceAll(RegExp(r'[^\d.]'), '');
    if (cleaned.isEmpty) return null;
    final parts = cleaned.split('.');
    final whole = int.tryParse(parts[0]);
    if (whole == null) return null;
    final frac = parts.length > 1 ? parts[1].padRight(2, '0').substring(0, 2) : '00';
    return Money(whole * 100 + int.parse(frac));
  }
}

class Template {
  final String id, kind;
  final List<String> senders;
  final RegExp pattern;
  final RegExp? balance;
  Template(this.id, this.kind, this.senders, this.pattern, this.balance);

  factory Template.fromJson(Map<String, dynamic> j) => Template(
    j['id'], j['kind'], List<String>.from(j['senders']),
    RegExp(j['pattern']),                                  // use (?i) inline for case-insensitive
    j['balancePattern'] == null ? null : RegExp(j['balancePattern']),
  );
}

class ParsedTxn {
  final String templateId, direction;
  final int amountMinor;
  final String? counterparty;
  final int? balanceMinor;
  ParsedTxn(this.templateId, this.direction, this.amountMinor, this.counterparty, this.balanceMinor);
}

class ParserEngine {
  final List<Template> templates;
  ParserEngine(this.templates);

  ParsedTxn? parse(RawEvent e) {
    final text = Normaliser.normalise(e.body);
    for (final t in templates.where((t) => t.senders.contains(e.sender))) {
      final m = t.pattern.firstMatch(text);
      if (m == null) continue;
      final amount = Money.tryParse(m.namedGroup('amount') ?? '');
      if (amount == null) continue;
      final bal = t.balance?.firstMatch(text)?.namedGroup('balance');
      return ParsedTxn(t.id, t.kind, amount.minor,
          m.groupNames.contains('counterparty') ? m.namedGroup('counterparty')?.trim() : null,
          bal == null ? null : Money.tryParse(bal)?.minor);
    }
    return null;                                           // → needs_review
  }
}
```

```dart
// test/corpus_test.dart  (each JSONL line: {"sender":..., "body":..., "expect": {...} | null})
void main() {
  final engine = ParserEngine(loadTemplatesSync());
  final lines = Directory('test/corpus').listSync().whereType<File>()
      .expand((f) => f.readAsLinesSync()).where((l) => l.trim().isNotEmpty);

  var total = 0, ok = 0;
  for (final line in lines) {
    final c = jsonDecode(line) as Map<String, dynamic>;
    total++;
    final got = engine.parse(RawEvent.test(c['sender'], c['body']));
    final exp = c['expect'] as Map<String, dynamic>?;
    final match = exp == null
        ? got == null
        : got != null && got.amountMinor == exp['amountMinor'] && got.direction == exp['direction'];
    if (match) { ok++; } else { print('MISS: ${c['body']}'); }
  }
  test('corpus parse rate', () => expect(ok / total, greaterThanOrEqualTo(0.95)));
}
```

### H. Fingerprint and dedupe

```dart
import 'dart:convert';
import 'package:crypto/crypto.dart';

String fingerprint({
  required int walletId, required String direction, required int amountMinor,
  String? reference, String? counterparty, required DateTime occurredAt,
}) {
  final bucket = occurredAt.toUtc().millisecondsSinceEpoch ~/ (5 * 60 * 1000);
  final tail = reference ?? '${(counterparty ?? '').toLowerCase().trim()}|$bucket';
  return sha256.convert(utf8.encode('$walletId|$direction|$amountMinor|$tail')).toString();
}
```

Bucket edges can split two events that are seconds apart. For production-quality dedupe, also check for an existing transaction with the same wallet, direction, and amount within ±5 minutes before inserting.

### I. Categoriser (rules first, then default)

```dart
int? categorise(String? counterparty, List<CategoryRule> rules, int fallbackId) {
  final text = (counterparty ?? '').toLowerCase();
  for (final r in rules..sort((a, b) => b.priority.compareTo(a.priority))) {
    final hit = r.matchType == 'regex' ? RegExp(r.pattern, caseSensitive: false).hasMatch(text)
                                       : text.contains(r.pattern.toLowerCase());
    if (hit) return r.categoryId;
  }
  return fallbackId;                                       // 'Other' or 'Transfers'
}
```

### J. IMAP sync skeleton (`enough_mail`)

```dart
Future<List<RawEvent>> syncImap(ImapAccount a, String password, int lastUid) async {
  final client = ImapClient(isLogEnabled: false);
  try {
    await client.connectToServer(a.host, a.port, isSecure: true);
    await client.login(a.username, password);
    await client.selectInbox();

    // Fetch only messages newer than the last seen UID, then filter by allow-listed sender.
    final seq = MessageSequence.fromRangeToLast(lastUid + 1, isUidSequence: true);
    final res = await client.uidFetchMessages(seq, 'BODY.PEEK[]');
    final out = <RawEvent>[];
    for (final m in res.messages) {
      final from = m.from?.first.email ?? '';
      if (!a.allowedSenders.contains(from)) continue;
      out.add(RawEvent(
        source: 'email', sender: from,
        body: '${m.decodeSubject() ?? ''}\n${m.decodeTextPlainPart() ?? ''}',
        receivedAt: m.decodeDate() ?? DateTime.now().toUtc(),
      ));
    }
    return out;
  } finally {
    await client.logout().catchError((_) {});
  }
}
```

Verify method names against the current `enough_mail` docs. Persist the highest UID you processed in `sync_state` after the DB write succeeds, never before.

### K. Notification listener (Kotlin sketch)

```kotlin
class WalletNotificationListener : NotificationListenerService() {
    private val allowed = setOf("PLACEHOLDER.WALLET.PACKAGE")   // fill in from real devices

    override fun onNotificationPosted(sbn: StatusBarNotification) {
        if (sbn.packageName !in allowed) return                  // drop everything else immediately
        val extras = sbn.notification.extras
        val text = listOfNotNull(
            extras.getCharSequence(Notification.EXTRA_TITLE),
            extras.getCharSequence(Notification.EXTRA_BIG_TEXT) ?: extras.getCharSequence(Notification.EXTRA_TEXT)
        ).joinToString("\n")
        PendingQueue.append(this, QueuedEvent("notif", sbn.packageName, text, sbn.postTime))
        DrainWorker.enqueue(this)
    }
}
```

### L. GitHub Actions CI

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with: { channel: stable, cache: true }
      - run: flutter pub get
      - run: dart run build_runner build --delete-conflicting-outputs
      - run: flutter analyze
      - run: flutter test
      - run: flutter build apk --debug
      - uses: actions/upload-artifact@v4
        with: { name: debug-apk, path: build/app/outputs/flutter-apk/app-debug.apk }
```

### M. Release build commands

```bash
flutter build apk --release --obfuscate --split-debug-info=build/symbols
# Keep build/symbols outside the repo; you need it to de-obfuscate stack traces.
```

---

## Final Notes

- **Re-verify** Android permission behaviour, Play policy, and IMAP provider requirements at the start of Phases 3, 5, and 7. They change.
- The strongest version of this project is not "three channels", it is **one channel that is demonstrably correct**, with measured parse rate, dedupe, reconciliation, and an honest write-up of limits.
- Keep a `DECISIONS.md` in the repo recording every major choice and why; it becomes half of your final report.
