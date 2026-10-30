# craft-cli-sync

Keeps the command-line tools of your ArtCraft apps in step with the desktop
apps installed by ArtCraft Launcher.

The launcher installs and updates the desktop apps (DeckCraft, GridCraft, …)
but not their CLIs. `craft-cli-sync` reads which apps the launcher has
installed, checks each app's version, and installs the matching
`<app>-cli` from the app's GitHub release into `~/.local/bin`.

## Usage

```sh
./craft-cli-sync            # install or update every out-of-sync CLI
./craft-cli-sync --check    # report status only, no network unless out of sync
./craft-cli-sync --dry-run  # show what would be downloaded and installed
```

Example:

```
== DeckCraft
Desktop: 0.12.0
CLI:     deckcraft-cli 0.12.0
Already synchronized.

== GridCraft
Desktop: 0.13.0
CLI:     not installed
Looking up release v0.13.0...
Asset:        gridcraft-cli-0.13.0-macos-universal.zip
Architecture: arm64
Downloading CLI...
Downloading checksums...
SHA-256 verified.

Successfully synchronized GridCraft!
Desktop: 0.13.0
CLI:     gridcraft-cli 0.13.0
Path:    /Users/you/.local/bin/gridcraft-cli
```

Make sure `~/.local/bin` is on your `PATH`.

### Exit codes

| Code | Meaning                                                           |
| ---- | ----------------------------------------------------------------- |
| 0    | All CLIs are in sync (or were synchronized / skipped)             |
| 1    | At least one app failed, or `--check` found an out-of-sync CLI    |
| 2    | Invalid arguments                                                 |

## How it works

For each app listed in
`~/Library/Application Support/ArtCraft Launcher/installed.json`:

1. Reads the desktop version (`CFBundleShortVersionString`) from the app's
   `Info.plist`.
2. Compares it with `~/.local/bin/<id>-cli --version`. If they match, the app
   is done.
3. Looks up the GitHub release `storytold/<id>` tagged `v<version>` and picks
   the `<id>-cli-…-macos-…zip` asset, preferring a universal build over an
   architecture-specific one.
4. Downloads the asset and the release's checksum manifest
   (`SHA256SUMS.txt`, `checksums.txt` or `SHA256SUMS`) and verifies the
   SHA-256. Without a valid checksum it refuses to install.
5. Unpacks the archive, runs the new binary's `--version` to confirm it matches
   the desktop version, then atomically replaces `~/.local/bin/<id>-cli`.

Each app is processed independently: a failure in one app is reported and the
script carries on with the next. Apps are skipped when their app bundle is
missing or their release has no macOS CLI.

## Requirements

- macOS (arm64 or x86_64) with zsh
- `curl`, `jq`, `unzip`, `shasum`, `mktemp`, `find`, `awk`
- ArtCraft Launcher, with at least one app installed

## Limitations

- The repository is derived from the launcher's app id
  (`deckcraft` → `storytold/deckcraft`) and releases must be tagged
  `v<version>`. An app that breaks either convention will fail its lookup.
- GitHub API requests are unauthenticated and subject to GitHub's rate limit
  (60 requests per hour per IP). Only out-of-sync apps make requests.
