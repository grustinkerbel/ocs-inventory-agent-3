# OCS Inventory Agent: macOS Package Builder

`ocs-build.sh` builds a **signed, notarized macOS `.pkg`** of the [OCS Inventory Agent (Dart rewrite)](https://github.com/OCSInventory-NG/OCSInventory-Agent-Rework), ready for distribution through MDM tools such as Jamf.

It targets macOS's stock `/bin/bash` 3.2, so no Homebrew bash is needed.

## What it does

1. Fetches a Dart SDK (SHA-256 verified) or uses the system one
2. Fetches the `OCSInventory-Agent-Rework` source at a pinned ref (branch, tag, or commit SHA)
3. Runs `dart pub get` and `dart compile exe`
4. Signs every Mach-O with **Developer ID Application** (hardened runtime, secure timestamp)
5. Builds the package with `pkgbuild` / `productbuild`, signed with **Developer ID Installer**
6. Notarizes with `notarytool`, staples, and validates

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
- `URL`, `USERNAME`, `PASSWORD`, `SILENT` (and optionally `SERVICE`, `NOW`) from the config are injected into the **staged** copy of `install.sh` at build time. The source checkout is never modified, and command-line options passed to `install.sh` still win.
- The package's `postinstall` runs `install.sh` non-interactively, then verifies that `/usr/local/bin/ocsinventory-cli` is byte-identical to the signed binary in the package and passes `codesign --verify`.
- The build verifies that the binary appears exactly once in the payload and that the credentials appear only in `install.sh`.

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
| `SILENT=true` | The postinstall runs `install.sh` with the baked-in values. `URL`, `USERNAME`, `PASSWORD` are required. |
| `SILENT=false` | The package only installs files; nothing is configured. |
| `SERVICE=true` | Register the LaunchDaemon. Without it, silent mode installs and configures the agent but **never schedules it**. |
| `NOW=true` | Run one inventory right after install. |

Validation rules enforced at build time:

- `URL` must start with `http://` or `https://`
- `URL`, `USERNAME`, `PASSWORD` must not contain `"`, `\`, or line breaks (`install.sh` writes them into `config.json` unescaped)
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

- With `SILENT=true`, the OCS URL, username, and password are stored **in clear text inside the `.pkg`** (in `install.sh`, mode `0700 root:wheel` once installed).
- The package is **uploaded to Apple** for notarization.
- Anyone who can obtain the `.pkg` (for example from a Jamf distribution point) can read the credentials.
- **Use a dedicated, low-privilege OCS account.**
- Changing any value means rebuilding, re-signing, and re-notarizing.
- On the target Mac, `config.json` and the setup logs are restricted to root (`0600`), and `/var/log/ocsinventory-agent-pkg-install.log` records the install.
- The temporary keychain (when using `.p12` files) is deleted and your keychain search list is restored on exit.

## Older `install.sh` versions

Some older revisions of `install.sh` referenced `$plist_source_file`, a local variable of `register_service()`, from `check_installed_agent()`. Under `set -u`, that aborted silent mode whenever an existing agent was found, so re-installs failed while fresh installs worked.

Current `main` no longer has this problem: `check_installed_agent()` uses the global `PLIST_DST`. The build still runs `check_install_sh_layout` and prints a warning if the pinned `OCS_REF` contains the old pattern. If you see that warning, build from a newer ref or a commit that includes the fix.

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
