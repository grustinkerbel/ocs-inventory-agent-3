# OCS Inventory Agent: macOS Package Builder

`ocs-build.sh` builds a **signed, notarized macOS `.pkg`** of the [OCS Inventory Agent (Dart rewrite)](https://github.com/OCSInventory-NG/OCSInventory-Agent-Rework), ready for distribution through MDM tools such as Jamf.

It targets macOS's stock `/bin/bash` 3.2, so no Homebrew bash is needed.

## What it does

1. Fetches a Dart SDK (SHA-256 verified) or uses the system one
2. Fetches the `OCSInventory-Agent-Rework` source at a pinned ref (branch, tag, or commit SHA)
3. Applies small local fixes to the fresh checkout (currently the LaunchDaemon plist, see [Local source fixes](#local-source-fixes))
4. Runs `dart pub get` and `dart compile exe`
5. Signs every Mach-O with **Developer ID Application** (hardened runtime, secure timestamp)
6. Builds the package with `pkgbuild` / `productbuild`, signed with **Developer ID Installer**
7. Notarizes with `notarytool`, staples, and validates

A build manifest (SHA-256, source commit, Dart version, etc.) and the resolved `pubspec.lock` are written next to each package.

## Package design

The repository's `setup/macos` folder (`install.sh`, `uninstall.sh`, `com.ocsinventory.agent.plist`) is installed together with the signed binary, in the same folder:

```
/usr/local/share/ocsinventory-agent/setup/macos/
├── install.sh
├── uninstall.sh
├── com.ocsinventory.agent.plist
└── ocsinventory-cli          # signed binary
```

- The package installs **nothing** into `/usr/local/bin`. `install.sh` makes that single copy.
- **`install.sh` is shipped exactly as upstream wrote it.** The build never edits it and fails if the staged copy differs from the checkout.
- `URL`, `USERNAME`, `PASSWORD` (and optionally `SERVICE`, `NOW`) from the config are baked into the generated `postinstall` script as command-line arguments. The package's `postinstall` runs `install.sh -S -U <url> -u <user> -p <password> [-s] [-n]` non-interactively. (A `.pkg` cannot receive arguments at install time, so they are fixed at build time. Environment variables cannot be used because `install.sh` resets its variables on startup.)
- After `install.sh` finishes, `postinstall` verifies that `/usr/local/bin/ocsinventory-cli` is byte-identical to the signed binary in the package and passes `codesign --verify`.
- The build verifies that the binary appears exactly once in the payload and that the credentials appear only in the `postinstall` script.

## Requirements

- macOS with Xcode Command Line Tools (`xcode-select --install`), including `notarytool` (Xcode 13+)
- Apple Developer Program membership with these certificates, each with its private key:
  - **Developer ID Application**
  - **Developer ID Installer**
- Notarization credentials: a `notarytool` keychain profile or an App Store Connect API key
- Network access to GitHub and `storage.googleapis.com` (Dart SDK)
- Building `x86_64` on Apple silicon requires Rosetta 2:
  `softwareupdate --install-rosetta --agree-to-license`. Building `arm64` on an Intel host is not supported.

## Quick start

```bash
# 1. Create your config from the example and lock it down
cp ocs-build.conf.example ocs-build.conf
chmod 600 ocs-build.conf

# 2. Edit ocs-build.conf (identities, notary profile, OCS URL, ...)

# 3. One-time: store notary credentials
xcrun notarytool store-credentials "ocs-notary" \
    --apple-id "you@example.com" --team-id "<TEAMID>" \
    --password "<app-specific-password>"

# 4. Supply secrets via environment, never in the config file
export OCS_PASSWORD='your-ocs-account-password'
export OCS_P12_PASSWORD='...'      # only if importing .p12 certificates

# 5. Check everything first (tools, certs, notary credentials; builds nothing)
./ocs-build.sh --check

# 6. Build
./ocs-build.sh
```

Output lands in `~/ocs-build-out/` (configurable):

```
ocs-inventory-agent-<version>-arm64.pkg   # actual name: <last part of PKG_IDENTIFIER>-<version>-<arch>.pkg
*.pkg.manifest.txt
*.pkg.pubspec.lock
```

## Command-line options

| Option | Description |
|---|---|
| `--config FILE` | Config file (default: `ocs-build.conf` next to the script) |
| `--check` | Preflight only: tools, certificates, notary credentials. Builds nothing. |
| `--no-notarize` | Build and sign only (testing; the result will not pass Gatekeeper as a downloaded file) |
| `--clean` | Delete `WORK_DIR` before building |
| `-h`, `--help` | Show help |

## Configuration

All settings live in `ocs-build.conf` (see [`ocs-build.conf.example`](ocs-build.conf.example) for full comments). The file is **sourced as shell code**, so the script refuses to read it if it is group- or world-writable.

| Section | Key settings |
|---|---|
| Signing identities | `APP_SIGN_IDENTITY`, `INSTALLER_SIGN_IDENTITY`, `TEAM_ID` (from `security find-identity -v`) |
| Certificate location (pick one) | **A)** `APP_CERT_P12` / `INSTALLER_CERT_P12` imported into a temporary keychain; **B)** `SIGNING_KEYCHAIN`; **C)** login keychain (leave both empty) |
| Notarization (pick one) | **A)** `NOTARY_KEYCHAIN_PROFILE`; **B)** `NOTARY_API_KEY_PATH` / `NOTARY_API_KEY_ID` / `NOTARY_API_ISSUER` |
| Source / toolchain | `OCS_REPO_URL`, `OCS_REF` (**pin a tag or SHA for production**), `DART_SOURCE` (`download`/`system`), `DART_CHANNEL`, `DART_VERSION` |
| Package | `TARGET_ARCHS` (`arm64`, `x86_64`), `BINARY_NAME`, `PKG_IDENTIFIER`, `PKG_TITLE`, `PKG_VERSION` (`auto` reads `pubspec.yaml`), `ENTITLEMENTS_FILE` |
| Setup values | `SILENT`, `URL`, `USERNAME`, `PASSWORD`, `SERVICE`, `NOW` |
| Directories | `WORK_DIR`, `OUTPUT_DIR` |

### Secrets

Never put passwords or private keys in the config file. Secrets are read from environment variables:

- `OCS_PASSWORD` overrides `PASSWORD`
- `OCS_P12_PASSWORD` (name configurable via `P12_PASSWORD_ENV`) unlocks `.p12` files
- `SIGNING_KEYCHAIN_PASSWORD_ENV` names the variable holding a keychain's unlock password

### Setup values

| Key | Meaning |
|---|---|
| `SILENT=true` | The postinstall runs `install.sh` with the baked-in values as arguments (`-S -U -u -p`). `URL`, `USERNAME`, `PASSWORD` are required. |
| `SILENT=false` | The package only installs files; nothing is configured. |
| `SERVICE=true` | Passes `-s` to `install.sh` to register the LaunchDaemon. Without it, silent mode installs and configures the agent but **never schedules it**. |
| `NOW=true` | Passes `-n` to `install.sh` to run one inventory right after install. |

Validation rules enforced at build time:

- `URL` must start with `http://` or `https://`
- `URL`, `USERNAME`, `PASSWORD` must not contain `"`, `\`, or line breaks (the upstream `install.sh` writes them into `config.json` unescaped)
- With `NOW=true` they must also contain no whitespace or any of `* ? [ $ \``, because `install.sh` passes them unquoted to a command

### Entitlements

`dart_entitlements.plist` grants the hardened-runtime exceptions a Dart AOT executable typically needs:

- `com.apple.security.cs.allow-jit`
- `com.apple.security.cs.allow-unsigned-executable-memory`
- `com.apple.security.network.client`

Leave `ENTITLEMENTS_FILE` empty to sign without entitlements, and add them only if testing shows the hardened runtime blocks something.

## Re-installs

If an agent is already present, `install.sh` itself runs `uninstall.sh -S -D` first. This is a **hard delete** of the config directory, data directory, log file, binary, and LaunchDaemon. It then installs again with the baked-in values. The package's `postinstall` never uninstalls anything. `uninstall.sh` needs `realpath` (built into recent macOS; check your minimum supported version).

## Security notes

- With `SILENT=true`, the OCS URL, username, and password are stored **in clear text inside the `.pkg`**, in the `postinstall` script. `install.sh` itself contains no credentials.
- While `install.sh` runs on the target Mac, the password is briefly visible in the process list as a command-line argument.
- The package is **uploaded to Apple** for notarization.
- Anyone who can obtain the `.pkg` (for example from a Jamf distribution point) can read the credentials.
- **Use a dedicated, low-privilege OCS account.**
- Changing any value means rebuilding, re-signing, and re-notarizing.
- On the target Mac, `config.json` and the setup logs are restricted to root (`0600`; `install.sh`'s `--now` run writes the password into `install.log`), and `/var/log/ocsinventory-agent-pkg-install.log` records the install.
- The temporary keychain (when using `.p12` files) is deleted and your keychain search list is restored on exit.

## Local source fixes

Right after the source is fetched, and before anything is compiled or staged, the build applies small fixes to the checkout (`apply_source_fixes` in `ocs-build.sh`). The packaging code does not know about them; it just uses whatever is in the checkout, so you can tweak the code without touching the package build.

| Fix | What it does |
|---|---|
| LaunchDaemon plist ([upstream issue #175](https://github.com/OCSInventory-NG/OCSInventory-Agent-Rework/issues/175)) | Changes `<string>--service true</string>` to `<string>--service=true</string>` in `setup/macos/com.ocsinventory.agent.plist`, because upstream passes it as a single argument the agent cannot parse. |

The plist fix is idempotent. If the plist already uses `--service=true`, it does nothing. If neither form is found, it prints a warning and leaves the file unchanged. After patching it checks the result and runs `plutil -lint`.

To add another fix, write a function in the "Source" section of `ocs-build.sh` and call it from `apply_source_fixes`. Keep each fix idempotent.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `Config file ... is group/world-writable` | `chmod 600 ocs-build.conf` |
| `Signing identity not found` | Run `security find-identity -v` and copy the identity string exactly; ensure the private key is present |
| `Certificate identity still a placeholder` | Replace the `REPLACE_ME` values in the config |
| `notarytool rejected the credentials` | Re-run `xcrun notarytool store-credentials`, or check the API key settings |
| Notarization not `Accepted` | The script prints Apple's log automatically; check for unsigned or missing hardened-runtime Mach-O files |
| `x86_64 build on Apple silicon needs Rosetta` | `softwareupdate --install-rosetta --agree-to-license` |
| `SHA-256 mismatch` for the Dart SDK | Re-run with `--clean`; check for proxy interference |
| `Staged install.sh differs from the upstream copy` | Something modified `setup/macos/install.sh` between checkout and staging. The build requires it to be unmodified. |
| Warning: plist has no `--service` argument in the expected form | Upstream changed the plist. Review `setup/macos/com.ocsinventory.agent.plist` in the checkout and update `fix_plist_service_arg` or remove it. |
| `Credential placement check failed` | The password was found somewhere other than the `postinstall` script, for example because it is a short string that also appears in another file. |

## Repository layout

```
ocs-build.sh              # the build script
ocs-build.conf.example    # copy to ocs-build.conf and edit
dart_entitlements.plist   # optional hardened-runtime entitlements
```

Add `ocs-build.conf` to `.gitignore`. It holds your signing identities and, potentially, deployment details:

```
ocs-build.conf
```
