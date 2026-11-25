# LEDGR Build Plan v2

> A privacy-first Android expense tracker that builds your ledger from wallet SMS, app notifications and email receipts. Local-only: no account, no cloud, and no internet permission in the core app.

**Team:** FA24-BCS-013 and FA24-BCS-023 · CSC303, COMSATS Wah
**Stack:** Flutter (Android only), a little Kotlin, drift + SQLCipher. **Cost:** $0.
**Toolchain (checked Nov 25, 2026):** Flutter 3.47.3 stable, Dart 3.13.3, Android build-tools 37. Use the same Flutter version on both laptops.
**Replaces** [LEDGR_RECOMMENDED_IMPLEMENTATION_PLAN.md](LEDGR_RECOMMENDED_IMPLEMENTATION_PLAN.md) (v1, kept next to this file for reference). Same product with about half the moving parts, built so there is a working demo at the end of every milestone.

---

## 1. Scope and order

The full proposal, one channel at a time. After **M3** (about day 16) you have a complete, demo-ready project. M4 to M6 add the remaining channels if time allows.

| Provider | Read from | Milestone |
|---|---|---|
| JazzCash, Easypaisa, your bank | SMS inbox | M2 |
| SadaPay, NayaPay | App notifications (they mostly don't send transaction SMS; **VERIFY**) | M4 |
| Emailed receipts | Paste box first, then share sheet, then IMAP | M2, M5 |
| Wallet app screens | Accessibility service (stretch) | M6 |

**Each provider is read from exactly one channel.** The same payment then can't arrive twice through different channels, so no cross-channel dedupe is needed.

---

## 2. What changed from v1, and why

| v1 | v2 | Why |
|---|---|---|
| Kotlin SMS receiver → encrypted native queue → WorkManager drain → boot receiver | Read the SMS inbox for wallet senders when the app opens or resumes, and on pull-to-refresh | The inbox already stores every message durably, so nothing is lost when the app is killed, the phone reboots or a battery manager steps in. Multipart SMS arrive joined. Importing past months is the same query. Only limit: an SMS deleted before LEDGR next opens is missed. |
| Background sync for every channel | Sync when the app opens, for every channel | The ledger is only seen with the app open, so the user can't tell the difference. Removes WorkManager, foreground services and the per-brand battery-settings screen. |
| sha256 fingerprints with ±5-minute buckets | One channel per provider, plus a unique `(source, sourceRef)` index | No duplicates, and two real Rs 100 canteen payments a minute apart are never merged by mistake. |
| Parser templates as JSON assets | A Dart list in `parser.dart` | Still data, not logic. Tests run in plain Dart without loading assets. |
| Riverpod + go_router | `StreamBuilder` on drift `.watch()` streams; `Navigator` + `NavigationBar` | The course doesn't require them; two fewer libraries to learn and explain in the viva. |
| Two build flavors to prove "no internet" | No `INTERNET` permission at all until IMAP (M5b) | Flutter only adds `INTERNET` to debug and profile builds. Check the release APK (Section 6). |
| Computed balance + reconciliation screen | Show the balance quoted in the wallet's latest message, with "as of" | It's the provider's own figure, so it can't drift. Wallets whose messages never quote a balance (and Cash) use opening balance + transactions. |
| Regex category rules with priorities | Merchant → category map learned from your corrections, plus seeded keywords | Nobody writes regex on a phone. |
| Custom PIN pad and lock edge cases | `local_auth` with the phone's own PIN/pattern fallback | Android handles lockouts and changed fingerprints. |
| `FLAG_SECURE` always on | Release builds only | It blacks out screenshots, screen recordings and screen mirroring: your report screenshots and a projected demo. |
| Budget rows per month + local notifications | One monthly limit per category, warning shown in the app | No notification permission or background work needed. |
| Timestamps parsed from message text | The time the SMS arrived | Usually within seconds; parsing five providers' date formats is a rabbit hole. |
| `Money.tryParse` (v1 Appendix G) | `parseMinor` (Appendix B) | v1 returns `null` for `"Rs.500"` and `"Rs. 1,250.50"`: the dot in "Rs." survives the cleanup, so the whole-number part comes out empty. Confirmed by running both. |

**Dropped:** root detection (easy to bypass; v1 already made it warning-only), R8 obfuscation and split debug info (`flutter build apk --release` already shrinks the app), CI, widget/integration/migration tests, the 50,000-row performance test (one person makes about 100 transactions a month), a light theme (the design is dark), separate wireframe/MoSCoW/taxonomy documents (the Assignment-2 dashboard and the corpus cover them), and the `crypto`, `csv`, `mocktail`, `workmanager`, `flutter_foreground_task`, `flutter_riverpod` and `go_router` packages.

**Later, only if you decide you need them:** PDF report export, budget alerts while the app is closed (a periodic job calling the same `sync()`), live updates without pull-to-refresh, IMAP IDLE, receipt OCR, backup/restore, Urdu UI.

---

## 3. Architecture

```
SMS inbox ───────────── readSms(afterMs, senders) ──┐
Notification listener ── queue file ── drain ───────┤
Share sheet / paste / IMAP fetch ───────────────────┤
                                                    ▼
      sync()   Dart · runs on launch, resume and pull-to-refresh
      parse → categorise → write, in one drift transaction per batch
        ├─ payment                         → transactions (+ wallet balance from the message)
        ├─ OTP or promo                    → dropped, never stored
        └─ no template matched, has "Rs"   → review_items
                                                    ▼
      drift + SQLCipher ───── .watch() ─────► StreamBuilder screens
```

### Files

```
android/app/src/main/kotlin/.../MainActivity.kt    readSms channel, FLAG_SECURE (Appendix A)
android/app/src/main/kotlin/.../NotifListener.kt   M4
android/app/src/main/kotlin/.../ScreenReader.kt    M6
lib/main.dart           app, lock gate, bottom navigation
lib/db.dart             drift tables, encrypted open, seed data
lib/money.dart          parseMinor, formatMinor (Appendix B)
lib/parser.dart         providers, templates, parse() (Appendix C)
lib/categorise.dart     merchant map, then keywords
lib/sync.dart           read each channel → parse → write
lib/screens/            dashboard, transactions, edit, review, analytics, budgets, wallets, sync, lock
lib/theme/              app_colors.dart, app_theme.dart (from Assignment 2)
test/corpus.dart        anonymised messages with expected results
test/parser_test.dart   checks every corpus case, prints parse rate per provider
```

### Data (drift)

| Table | Columns | Notes |
|---|---|---|
| `wallets` | id, name, provider?, openingBalanceMinor, lastBalanceMinor?, lastBalanceAt? | One per provider, created on its first message; Cash seeded |
| `categories` | id, name, icon, excludeFromTotals | Food, Transport, Bills, Shopping, Mobile, Transfers (excluded), Income, Other |
| `transactions` | id, walletId, direction (debit/credit), amountMinor, counterparty?, categoryId?, occurredAt, source (sms/notif/email/screen/manual), sourceRef?, rawText?, note? | Unique `(source, sourceRef)` |
| `review_items` | id, source, sourceRef, sender, body, receivedAt | Unique `(source, sourceRef)`; removed when added or ignored |
| `merchant_categories` | merchant (key, lowercase), categoryId | Written by "Always use this category" |
| `budgets` | categoryId (key), monthlyLimitMinor | Same limit every month |
| `settings` | key, value | e.g. `smsCursor` |

- Money is `int` paisa. `double` only when formatting for display.
- Store times in UTC; show local time.
- The SMS cursor is the SMS `date`, not `_id`, and an SMS's `sourceRef` is `"<_id>:<date>"`: Android doesn't promise an `_id` is never reused after deletes.
- Keep `schemaVersion` at 1 until the first release build. Until then, change the schema and reinstall; the inbox import rebuilds SMS history (manual test entries are lost, which is fine).

---

## 4. Verify on day 1

These decide whether the plan works on your phones. Check them before building anything else.

- [ ] `READ_SMS` can be granted on **both** phones when installed with `flutter run` or `adb install`. An APK sent over WhatsApp or opened from a file manager may be blocked by Android's "restricted settings" or Play Protect, so always install over USB.
- [ ] The exact sender `address` each provider's SMS has in the inbox (a shortcode or a name). Copy it from the spike's output; the shortcodes in v1 were never confirmed.
- [ ] What SadaPay and NayaPay actually send you (SMS, notifications, email), and their package names: `adb shell pm list packages | findstr /i "sada naya"`.
- [ ] drift + SQLCipher: follow drift's current encryption docs rather than v1's Appendix F. The `sqlite3` package's native setup changed in 2025 (build hooks).
- [ ] `local_auth` needs `MainActivity` to extend `FlutterFragmentActivity` (Appendix A already does).

---

## 5. Milestones

Every milestone ends with an app you could demo. With Claude Code writing most of the code, the slow parts are human: collecting real messages, testing on two phones, and understanding the code well enough to explain it in the viva. Give Claude Code one milestone at a time, and have whoever didn't build a feature review it.

**Member A (FA24-BCS-023):** Kotlin, database, security, device testing. **Member B (FA24-BCS-013):** corpus, parser, screens, analytics, IMAP. Swap if you like.

| Days | ≈ Finish | Milestone | Done when |
|---|---|---|---|
| 1–2 | Nov 27 | **M0** Spike + corpus | The app lists your real wallet SMS on both phones; anonymised corpus committed |
| 3–6 | Dec 1 | **M1** Manual ledger | You can log a day of spending by hand; data survives a restart; the DB file is encrypted |
| 7–12 | Dec 7 | **M2** SMS channel, done properly | Corpus ≥ 95% parsed with 0 OTPs/promos turned into transactions; a real payment appears after pull-to-refresh |
| 13–16 | Dec 11 | **M3** Analytics, budgets, export, hardening | **Complete project.** Everything on the Assignment-2 dashboard works with real data, from a release APK |
| 17–20 | Dec 15 | **M4** SadaPay/NayaPay notifications | A SadaPay or NayaPay payment appears after sync |
| 21–24 | Dec 19 | **M5** Email | An emailed receipt becomes a transaction |
| 25–28 | Dec 23 | **M6** Screen reading (stretch) + report | Works on one wallet app, or the report explains why it can't |

Dates assume you start Nov 26. If your deadline comes first, stop after M3: it is a complete project on its own.

### M0: Spike + corpus (days 1–2)
- Done Nov 25: Flutter project created (`flutter create --platforms=android --org pk.edu.comsats --empty`) and pushed to [github.com/alinoor4/ledgr](https://github.com/alinoor4/ledgr). The repo is **public**, so only anonymised messages may ever be committed. Keep raw exports in `private/`, which git ignores.
- Bring in the Assignment-2 theme (`app_colors.dart`, `app_theme.dart` → `lib/theme/`) and dashboard (→ `lib/screens/`).
- **A:** add `readSms` (Appendix A), `READ_SMS` in the manifest, and a throwaway screen listing what it returns. Run it on both phones.
- **B:** write `test/corpus.dart`: 20+ real messages per SMS provider, anonymised (names, phone and account numbers, transaction IDs replaced; wording and layout untouched). Cover what you actually receive: sent, received, bill, top-up, ATM, failed, reversal, fee, plus OTPs and promos that must be ignored.
- **Both:** turn on Notification history (Settings → Notifications, Android 11+) and screenshot SadaPay/NayaPay notifications as they arrive. M4 needs about 20.

### M1: Manual ledger (days 3–6)
- drift schema (Section 3), opened with SQLCipher; random 256-bit key in `flutter_secure_storage`. Check that the DB file's first 16 bytes are not `SQLite format 3\0`. If SQLCipher takes more than a day, ship plain SQLite for now (Android already encrypts app storage on disk), fix the dashboard copy, and return to it after M3.
- Lock gate: `local_auth` on launch, and on resume after 60 s in the background.
- The five tabs from Assignment 2 (Dashboard, Analytics, Budgets, Wallets, Sync) on live data through `StreamBuilder`, with an empty state on every list.
- Add, edit and delete a transaction (amount, wallet, direction, category, date, note). Seed the categories and a Cash wallet.

### M2: SMS channel, done properly (days 7–12)
- `parser.dart` (Appendix C): for each provider, its senders, an optional balance pattern, and templates of kind `debit`, `credit` or `ignore`.
- `parser_test.dart` runs the whole corpus, prints the parse rate per provider, and fails below 95% or when any ignored message becomes a transaction. These numbers go in the report.
- `sync()` for SMS: request `READ_SMS` with `permission_handler` → read since the cursor → parse → insert transactions or review items → update the wallet's last balance → move the cursor. All of it in one drift transaction, so a crash can't leave a half-finished import.
- First run asks "Import the last 6 months?" (the same query with an older cursor), so analytics has history on day one.
- Review queue in the Sync tab: unmatched messages that mention an amount. Tap one to open the add form prefilled with the amount found, or Ignore it. "Retry all" re-parses the queue after you fix a template.
- "Paste a message" box: runs every provider's templates on the pasted text and opens a prefilled add form. It lets you test the parser on the phone, and handles emailed receipts until M5.
- Categoriser: merchant map first, then seeded keywords (foodpanda → Food, careem/indrive/bykea → Transport, k-electric/sngpl/ptcl → Bills, daraz → Shopping), else Other. Changing a category offers "Always for <merchant>".
- Permission screen, one plain paragraph: LEDGR reads only SMS from these senders, and the filter runs inside Android's own query, so other messages never reach the app.

### M3: Analytics, budgets, export, hardening (days 13–16)
- Analytics: month picker; spend by category as horizontal bars; spend vs income for the last 6 months. Transfers are excluded from totals. Built-in widgets are enough (`LinearProgressIndicator`, sized `Container`s); add `fl_chart` only if you want the Assignment-2 chart look.
- Budgets: monthly limit per category, a progress bar, and a warning colour at 80% and 100%.
- Wallets: quoted balance with "as of"; tap for that wallet's transactions.
- CSV export through `share_plus`, with a note that the file is not encrypted. Quote fields by hand (one line) instead of adding the `csv` package.
- Hardening: `allowBackup="false"` plus data extraction rules; `FLAG_SECURE` in release builds only; never log message bodies; an About screen saying what LEDGR reads, with `showLicensePage`.
- Change the dashboard copy to match what ships (it currently promises email and screen reading).
- Build the release APK, install it on both phones and run the Section 6 checks now, not the night before the demo.

### M4: SadaPay/NayaPay notifications (days 17–20)
- `NotificationListenerService` with an allow-list of the package names from Section 4. Every other notification is dropped before it's read.
- The listener runs while LEDGR is closed, so it appends title + text to a small queue file in app storage, and `sync()` drains it. Plain text is acceptable: it stays in private app storage only until the next sync, and the same text was already in the notification shade.
- `sourceRef` = notification key + post time. Check on a real phone that wallet apps don't re-post the same notification.
- Templates and corpus cases for SadaPay/NayaPay, written from the notification screenshots.
- Sync tab: how to grant and revoke access, including the "Allow restricted settings" step Android 13+ may require for sideloaded apps (**VERIFY**).

### M5: Email (days 21–24)
- **5a, no internet:** accept Android's share sheet (`text/plain`), so "Share → LEDGR" from Gmail opens the same prefilled add form.
- **5b, IMAP:** `enough_mail`; app password in `flutter_secure_storage`; on `sync()`, fetch new mail from allow-listed sender addresses; save the last UID only after the DB write succeeds. Gmail needs 2-Step Verification and an app password (**VERIFY** current rules). This adds the `INTERNET` permission and ends the "provably offline" claim; if that claim matters for your grade, ship IMAP as a separate build flavor.

### M6: Screen reading (stretch, days 25–28)
- `AccessibilityService` limited by `packageNames` to one wallet app. Read the "payment successful" screen and send the text through the same pipeline (`source = screen`).
- Many banking apps block accessibility services or mark screens `FLAG_SECURE`. If yours does, record that; it's a valid result.
- Write the channel comparison for the report: effort, reliability and privacy of SMS vs notifications vs email vs screen.

---

## 6. Testing and checks

- `parser_test.dart` over the corpus: the test that matters, and the headline number in your report.
- Small unit tests: `parseMinor`, the categoriser, and `sync()` importing the same SMS twice → one row (in-memory drift).
- Emulator: `adb emu sms send 8558 "<message>"` injects a test SMS (**VERIFY** whether named senders work). Use it to test and rehearse without spending money.
- Release APK permissions: `aapt2 dump permissions build/app/outputs/flutter-apk/app-release.apk` (`aapt2` is in `Android/Sdk/build-tools/37.0.0/`). Until M5b, `INTERNET` must not appear. This output is your privacy proof.
- On both phones at each milestone: grant the permission, force-stop and reboot then sync, check the lock.

---

## 7. Demo and report

- Demo from a phone or emulator loaded with corpus messages. Never show your real finances on a projector. Install over USB.
- Live moment: someone sends Rs 10 to your JazzCash during the demo, you pull to refresh, and it appears, categorised. Fallback: `adb emu sms send` on the emulator.
- Record a backup video from a debug build (release builds record as black).
- Report: corpus size and parse rate per provider, false positives, the release APK's permission list, and the channel comparison from M6. Section 2 of this plan is your design-decisions chapter.

---

## 8. Risks

| Risk | Mitigation |
|---|---|
| SMS permission blocked on your phones | Day-1 spike on both phones; install over USB |
| Real message formats differ from samples, or change | Corpus test, review queue, paste box, Retry all |
| SadaPay/NayaPay invisible until M4 | Paste box covers them meanwhile |
| Real financial data leaks (git, screenshots, slides) | Anonymise before committing; demo device with fake data |
| SQLCipher setup eats days | One-day timebox, then plain SQLite until after M3 |
| Release build behaves differently from debug | Release APK on both phones at the end of M3 |
| Deadline arrives mid-plan | Every milestone is demo-able; M3 is a complete project |

---

## Appendix A: `MainActivity.kt`

Not compiled yet; M0 is where it gets run.

```kotlin
package pk.edu.comsats.ledgr   // keep your project's package

import android.content.pm.ApplicationInfo
import android.os.Bundle
import android.provider.Telephony
import android.view.WindowManager
import io.flutter.embedding.android.FlutterFragmentActivity
import io.flutter.embedding.engine.FlutterEngine
import io.flutter.plugin.common.MethodChannel

class MainActivity : FlutterFragmentActivity() {   // local_auth needs a FragmentActivity

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // Release only: FLAG_SECURE also blacks out screenshots, recordings and screen mirroring.
        if ((applicationInfo.flags and ApplicationInfo.FLAG_DEBUGGABLE) == 0) {
            window.addFlags(WindowManager.LayoutParams.FLAG_SECURE)
        }
    }

    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        super.configureFlutterEngine(flutterEngine)
        MethodChannel(flutterEngine.dartExecutor.binaryMessenger, "ledgr/native")
            .setMethodCallHandler { call, result ->
                when (call.method) {
                    "readSms" -> result.success(readSms(
                        call.argument<Number>("afterMs")!!.toLong(),
                        call.argument<List<String>>("senders")!!))
                    else -> result.notImplemented()
                }
            }
    }

    // Wallet SMS newer than afterMs, oldest first. The sender filter runs inside Android's
    // query, so other messages never reach LEDGR.
    // shortcut: runs on the main thread; move it to a background thread if the 6-month import stutters.
    private fun readSms(afterMs: Long, senders: List<String>): List<Map<String, Any?>> {
        if (senders.isEmpty()) return emptyList()
        val where = "date > ? AND address IN (${senders.joinToString(",") { "?" }})"
        val args = arrayOf(afterMs.toString()) + senders
        val out = mutableListOf<Map<String, Any?>>()
        contentResolver.query(
            Telephony.Sms.Inbox.CONTENT_URI, arrayOf("_id", "address", "body", "date"),
            where, args, "date ASC"
        )?.use { c ->
            while (c.moveToNext()) out += mapOf(
                "id" to c.getLong(0), "sender" to c.getString(1),
                "body" to c.getString(2), "date" to c.getLong(3))
        }
        return out
    }
}
```

## Appendix B: `money.dart`

Checked with `dart run --enable-asserts` against the cases below.

```dart
import 'package:intl/intl.dart';

/// "Rs. 1,250.50" → 125050 paisa. Reads the first number in [s]; null if there is none.
int? parseMinor(String s) {
  final m = RegExp(r'(\d[\d,]*)(?:\.(\d{1,2}))?').firstMatch(s);
  if (m == null) return null;
  return int.parse(m[1]!.replaceAll(',', '')) * 100 + int.parse((m[2] ?? '').padRight(2, '0'));
}

final _fmt = NumberFormat('#,##0.00');
String formatMinor(int minor) => 'Rs ${_fmt.format(minor / 100)}'; // double for display only

// test/money_test.dart
// expect(parseMinor('Rs. 1,250.50'), 125050);   // v1 returned null
// expect(parseMinor('Rs.500'), 50000);           // v1 returned null
// expect(parseMinor('Rs 1,250/-'), 125000);
// expect(parseMinor('Rs 0.5'), 50);
// expect(parseMinor('Rs.'), null);
```

## Appendix C: parser shape

Checked against synthetic messages: a payment SMS ending "never share your PIN/OTP" parses as a debit, an OTP is ignored, a promo returns `null`.

```dart
class Template {
  Template(this.id, this.kind, String pattern) : re = RegExp(pattern, caseSensitive: false);
  final String id;
  final String kind; // 'debit' | 'credit' | 'ignore'
  final RegExp re;   // named groups: amount (required for debit/credit), counterparty (optional)
}

class Provider {
  Provider(this.key, this.channel, this.senders, this.templates, {String? balance})
      : balance = balance == null ? null : RegExp(balance, caseSensitive: false);
  final String key;
  final String channel;           // 'sms' | 'notif' | 'email' | 'screen': exactly one per provider
  final List<String> senders;     // SMS addresses, package names or email addresses, copied from real messages
  final List<Template> templates; // tried in order
  final RegExp? balance;          // named group: balance
}

class Parsed {
  Parsed(this.kind, this.amountMinor, {this.counterparty, this.balanceMinor});
  final String kind;
  final int amountMinor;
  final String? counterparty;
  final int? balanceMinor;
}

/// null: no template matched (goes to review if it mentions an amount).
/// kind 'ignore': OTP or promo, dropped.
Parsed? parse(Provider p, String body) {
  final text = body.replaceAll(RegExp(r'\s+'), ' ');
  for (final t in p.templates) {
    final m = t.re.firstMatch(text);
    if (m == null) continue;
    if (t.kind == 'ignore') return Parsed('ignore', 0);
    // Throws if the template has no `amount` group, so the corpus test catches it.
    final amount = parseMinor(m.namedGroup('amount')!);
    if (amount == null) continue;
    final bal = p.balance?.firstMatch(text)?.namedGroup('balance');
    return Parsed(t.kind, amount,
        counterparty: m.groupNames.contains('counterparty') ? m.namedGroup('counterparty')?.trim() : null,
        balanceMinor: bal == null ? null : parseMinor(bal));
  }
  return null;
}

// Illustrative only: write every pattern against your anonymised corpus, never from memory.
final providers = [
  Provider('jazzcash', 'sms', ['SENDER_FROM_SPIKE'], [
    // Keep ignore patterns narrow: payment SMS often end with "never share your PIN/OTP".
    Template('jc.otp', 'ignore', r'^your otp'),
    Template('jc.sent', 'debit', r'sent rs\.? ?(?<amount>[\d,]+(?:\.\d{1,2})?) to (?<counterparty>.+?) on'),
  ], balance: r'bal(?:ance)?:? rs\.? ?(?<balance>[\d,]+(?:\.\d{1,2})?)'),
];
```
