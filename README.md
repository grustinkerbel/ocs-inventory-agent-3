# OCS Inventory Agent: macOS Package Builder

`ocs-build.sh` builds a **signed, notarized macOS `.pkg`** of the [OCS Inventory Agent (Dart rewrite)](https://github.com/OCSInventory-NG/OCSInventory-Agent-Rework), ready for distribution through MDM tools such as Jamf.

It targets macOS's stock `/bin/bash` 3.2, so no Homebrew bash is needed.

## What it does

1. Fetches a Dart SDK (SHA-256 verified) or uses the system one
2. Fetches the `OCSInventory-Agent-Rework` source at a pinned ref (branch, tag, or commit SHA)
3. Runs `dart pub get` and `dart compile exe`, passing the package version to the binary as `-DAPP_VERSION`
4. Signs every Mach-O with **Developer ID Application** (hardened runtime, secure timestamp)
5. Builds the package with `pkgbuild` / `productbuild`, signed with **Developer ID Installer**, optionally with an installer background logo
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
- With `SILENT=true`, the package's `postinstall` passes `URL`, `USERNAME`, `PASSWORD`, and optionally `SERVICE`/`NOW` as command-line arguments to the **unmodified** `install.sh`; the source checkout is never modified. With `SILENT=false`, `postinstall` installs the package files and exits without running setup or `install.sh`.
- After setup runs, `postinstall` verifies that `/usr/local/bin/ocsinventory-cli` is byte-identical to the signed binary in the package and passes `codesign --verify`.
- The build verifies that the binary appears exactly once in the payload and that the password appears only in the generated `postinstall` script.

### Local source fix: LaunchDaemon plist

Right after fetching the source, the build patches `setup/macos/com.ocsinventory.agent.plist` in the checkout: `<string>--service true</string>` becomes `<string>--service=true</string>`. Upstream issue #175: the original passes `--service true` as one argv element, which the agent cannot parse. The patch is idempotent and skipped if upstream has already fixed it, so the plist in the package may differ from the repository's copy. `install.sh` itself is never patched.

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

# 2. Edit ocs-build.conf (identities, notary profile, OCS URL, optional logo ...)

# 3. One-time: store notary credentials
xcrun notarytool store-credentials "ocs-notary" \
    --apple-id "you@example.com" --team-id "<TEAMID>" \
    --password "<app-specific-password>"

# 4. Supply secrets via environment, never in the config file
export OCS_PASSWORD='your-ocs-account-password'
export OCS_P12_PASSWORD='...'      # only if importing .p12 certificates

# 5. Check everything first (tools, certs, notary credentials, logo files; builds nothing)
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
| `--check` | Preflight only: tools, certificates, notary credentials, logo files. Builds nothing. |
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
| Package | `TARGET_ARCHS` (`arm64`, `x86_64`), `BINARY_NAME`, `PKG_IDENTIFIER`, `PKG_TITLE`, `PKG_VERSION` (`auto` reads `pubspec.yaml`; also passed to the binary as `APP_VERSION`), `ENTITLEMENTS_FILE` |
| Installer logo (optional) | `LOGO_FILE`, `LOGO_DARK_FILE`, `LOGO_ALIGNMENT`, `LOGO_SCALING` |
| Setup values | `SILENT`, `URL`, `USERNAME`, `PASSWORD`, `SERVICE`, `NOW` |
| Directories | `WORK_DIR`, `OUTPUT_DIR` |

### Secrets

Keep passwords and private keys out of the config file. For the OCS password, export `OCS_PASSWORD`; it overrides `PASSWORD` from the config. The build writes the resolved setup values into the generated `postinstall` script, so supplying the password through the environment does **not** keep it out of the `.pkg`.

- `OCS_PASSWORD` overrides `PASSWORD`
- `OCS_P12_PASSWORD` (name configurable via `P12_PASSWORD_ENV`) unlocks `.p12` files
- `SIGNING_KEYCHAIN_PASSWORD_ENV` names the variable holding a keychain's unlock password

### Installer logo

An optional PNG is shown as the installer window's background image. It is off unless `LOGO_FILE` is set.

| Key | Meaning |
|---|---|
| `LOGO_FILE` | PNG for the light appearance. Empty = no logo. |
| `LOGO_DARK_FILE` | Optional PNG for Dark Mode. If empty, Dark Mode shows no logo. |
| `LOGO_ALIGNMENT` | `center`, `left`, `right`, `top`, `bottom`, `topleft`, `topright`, `bottomleft` (default), `bottomright` |
| `LOGO_SCALING` | `none` (default, native size), `proportional` (scale down to fit), `tofit` (stretch) |

The build copies the PNGs into a resources folder passed to `productbuild --resources` and references them from `<background>` / `<background-darkAqua>` in `distribution.xml`. Preflight checks that the files exist, have a `.png` extension and are really PNG images, and that the alignment and scaling values are valid. Relative paths (also for `ENTITLEMENTS_FILE`) are resolved against the directory containing `ocs-build.sh`, so the script works from any working directory.

Preparing the image: export only the logo, tightly cropped, with a transparent background, and add about 40–60 px of transparent padding on the sides that touch the window edge so it does not hug the border. No full-window canvas or positioning in Photoshop is needed, because the installer places the whole PNG according to `LOGO_ALIGNMENT`. For a sharp Retina result, make the PNG twice the intended size and set its resolution to 144 ppi. Do a `--no-notarize` build to check how it looks before the final build.

### Setup values

| Key | Meaning |
|---|---|
| `SILENT=true` | The package's `postinstall` passes the baked-in values to `install.sh`. `URL`, `USERNAME`, `PASSWORD` are required. |
| `SILENT=false` | The package only installs files; nothing is configured. |
| `SERVICE=true` | Register the LaunchDaemon. The service reports once when it starts, then on the schedule set by the OCS server. Without it, silent mode installs and configures the agent but **never schedules it**. |
| `NOW=true` | Only used when `SERVICE=false`: run one inventory at install time (nothing is scheduled afterwards). With `SERVICE=true` it is ignored, with a build warning, because the service already reports on start. |

| `SERVICE` | `NOW` | Result |
|---|---|---|
| `false` | `false` | Installed and configured; no daemon, no report |
| `false` | `true` | One-shot inventory at install; nothing scheduled afterwards |
| `true` | any | Daemon registered; reports once on start, then on the server's schedule (`-n` is never passed) |

Validation rules enforced at build time:

- `URL` must start with `http://` or `https://`
- `URL`, `USERNAME`, `PASSWORD` must not contain `"`, `\`, or line breaks (`install.sh` writes them into `config.json` unescaped)
- With `NOW=true` and `SERVICE=false` they must also contain no whitespace or any of `* ? [ $ \``, because `install.sh` passes them unquoted to a command (`-n` is not used with the service, so this does not apply when `SERVICE=true`)

### Entitlements

`dart_entitlements.plist` grants the hardened-runtime exceptions a Dart AOT executable typically needs:

- `com.apple.security.cs.allow-jit`
- `com.apple.security.cs.allow-unsigned-executable-memory`
- `com.apple.security.network.client`

Leave `ENTITLEMENTS_FILE` empty to sign without entitlements, and add them only if testing shows the hardened runtime blocks something.

## Re-installs

If an agent is already present, `install.sh` itself runs `uninstall.sh -S -D` first. This is a **hard delete** of the config directory, data directory, log file, binary, and LaunchDaemon. It then installs again with the baked-in values. The package's `postinstall` never uninstalls anything. `uninstall.sh` needs `realpath` (built into recent macOS; check your minimum supported version).

## Security notes

- With `SILENT=true`, the OCS URL, username, and password are stored **in clear text inside the `.pkg`**, embedded in its `postinstall` script. The values are passed as arguments to `install.sh` when the package is installed.
- The credentials are **not left on the target Mac** by the package: `install.sh` ships unmodified, and the `postinstall` script that carries the values is discarded after installation. They end up only in `config.json` (root-only, `0600`) and, with `NOW=true`, in the root-only setup logs.
- While `install.sh` runs, the password is on its command line (`-p <password>`), so any local user can see it in `ps` for those few seconds.
- The package is **uploaded to Apple** for notarization.
- Anyone who can obtain the `.pkg` (for example from a Jamf distribution point) can read the credentials from the `postinstall` script.
- **Use a dedicated, low-privilege OCS account.**
- Changing any value means rebuilding, re-signing, and re-notarizing.
- On the target Mac, `config.json` and the setup logs are restricted to root (`0600`), and `/var/log/ocsinventory-agent-pkg-install.log` records the install.
- The temporary keychain (when using `.p12` files) is deleted and your keychain search list is restored on exit.

## Older `install.sh` versions

Some older revisions of `install.sh` referenced `$plist_source_file`, a local variable of `register_service()`, from `check_installed_agent()`. Under `set -u`, that aborted silent mode whenever an existing agent was found, so re-installs failed while fresh installs worked. The build still runs `check_install_sh_layout` on the staged `install.sh` (detection only, no edits) and prints a warning if the pinned `OCS_REF` contains the old pattern. If you see it, build from a newer ref or a commit that includes the fix.

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
| `Logo file not found` / `Logo must be a PNG file` / `not a real PNG image` | Check `LOGO_FILE` / `LOGO_DARK_FILE` paths; they must be genuine PNG files ending in `.png` |
| Logo looks too big, small or soft | Resize the PNG (use 2x size at 144 ppi for Retina) and rebuild; keep `LOGO_SCALING="none"` |

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
