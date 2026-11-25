# Secure Manager — Android (Kotlin)

Native Android port of the desktop **Secure Manager** in this repository. Same
data model, same encryption, same features — no web view and no server.

Version 2.0 replaces the original XML/views frontend with a completely rebuilt
**Jetpack Compose + Material 3** UI: dark gradient theme, animated navigation
drawer, searchable record cards, bottom-sheet details, dynamic forms, charts and
file-picker integration. The encrypted backend is unchanged.

## Build & install

```bash
cd android
./gradlew assembleDebug
adb install app/build/outputs/apk/debug/app-debug.apk
```

Or from the project root:


Requirements: JDK 17+, Android SDK platform 37, build-tools 36+.
Gradle 9.6 is downloaded automatically by the wrapper (already cached here).

Run the crypto compatibility tests with `./gradlew testDebugUnitTest` — they
check that a token created by the Python `core/encryption.py` decrypts in Kotlin
(and vice versa), plus file-blob and strength-rule parity.

## What is included

| Section | Notes |
|---|---|
| Dashboard | Live counts for every vault category |
| Emails | Account + authorized apps/devices (nested list) |
| Passwords | Encrypted credentials, strength label, generator & history |
| Cards | Encrypted number/CVV/PIN |
| Wallets | Encrypted private key / seed phrase / passphrase |
| Documents | Encrypted attachments (pick from device, download back) |
| Health, Projects, Notes, Goals, Contacts, Alerts | Full add/edit/delete, favorites & progress |
| Security report | Strength/age breakdown + recommendations |
| Settings | Display name, password change with **full re-encryption**, data reset, account deletion |

The drawer (☰) switches sections, the **＋** button adds entries, and tapping an
entry opens a detail sheet with copy/reveal/edit/delete and section-specific
actions.  locks the vault and returns to the login screen.

## Security model (identical to the Python app)

- Login password hashed with **bcrypt** (`org.mindrot:jbcrypt`). Python writes
  `$2b$` hashes and jbcrypt expects `$2a$`, so `BcryptCompat` normalizes the
  prefix before verifying — desktop accounts work unchanged.
- Per-user random 32-byte salt, key derived with **PBKDF2-HMAC-SHA256**, 480,000 iterations.
- Records encrypted with **Fernet (AES-128-CBC + HMAC-SHA256)**; text values are
  double-base64 encoded and attachments use Fernet's ASCII token string — exactly
  the desktop format, so the same vault can be opened by both apps.
- Keys live only in memory for the session; the master password is never stored.
- Changing the master password decrypts and re-encrypts every field and attachment.
- All SQLite files are app-private, backups/cloud sync are disabled, screenshots
  are blocked with `FLAG_SECURE`, and the manifest requests **no INTERNET permission**.

## Data compatibility

Android stores the same SQLite files under
`/data/data/com.securemanager.app/files/databases/`:

```
users.db  emails.db  passwords.db  cards.db  wallets.db  documents.db
health.db  projects.db  notes.db  goals.db  phones.db  alerts.db
```

You can copy the desktop `~/.local/share/SecurePasswordManager/databases/` folder there
with `adb push` (root/debug app required) and log in with the same username and
master password.

## Project layout

```
android/app/src/main/java/com/securemanager/app/
├── SecureApp.kt             application setup
├── crypto/Crypto.kt         Fernet/PBKDF2/generator (matches core/encryption.py)
├── data/Model.kt            sections/fields/preferences
├── data/VaultDb.kt          SQLite schema + generic queries
├── data/Vault.kt            auth, CRUD, re-encryption, reports
└── ui/
    ├── theme/Theme.kt       dark Material 3 palette, gradients, typography
    ├── components/          gradient buttons, cards, secrets, form editors
    ├── sections/            dashboard, generator, reports, settings, CRUD
    ├── AuthActivity.kt      Compose login / registration host
    ├── AuthScreen.kt        login UI
    ├── MainScreen.kt        drawer, top bar, FAB, section routing
    └── MainActivity.kt      Compose host + document picker / SAF bridge
```

All PBKDF2/bcrypt work and bulk vault decryption run on `Dispatchers.IO`, so
unlocking even a large copied vault never blocks the UI thread.
