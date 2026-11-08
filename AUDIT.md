# Security review of Couchlink (branch `svarus`)

Reviewed: upstream `cranebeaker/couchlink` at `2ce2748` (2026-11-05), source only. The upstream release zip was
downloaded for comparison but never run. This branch builds from source and turns off everything that reaches an
outside site by default.

## Summary

Couchlink is a reasonable piece of software with a few places where one person's key or repository can run code on
every install. Those places are switched off here. Everything else that needs elevated rights already asks through
the normal Windows prompt and only on a button press.

## Findings

| # | What | Where | Risk | In `svarus` |
| --- | --- | --- | --- | --- |
| 1 | Live recipes are fetched from `coopanything.app` with the catalog and trusted if signed by the upstream author's ed25519 key. A recipe can `fetch` a file into the game folder (hash given by the same recipe) and `launch` it, so the key holder can run code on every install. | `couchlinkd/src/session/remote.rs`, `couchlink-core/src/recipe.rs` (`Fetch`, `Launch`, `first`) | High | Off. No built-in URL or key; needs `COUCHLINK_RECIPES_URL` and `COUCHLINK_RECIPES_PUBKEY`. |
| 2 | Self-update downloads the newest GitHub release of upstream. The SHA-256 it checks comes from the same GitHub API answer, so it proves transport integrity, not who built it. Install needs a click. | `couchlinkd/src/update.rs`, `hub.rs` | Medium | Off. Needs `COUCHLINK_RELEASES_URL`. |
| 3 | The game catalog and vote counts are fetched from `coopanything.app` every 6 hours (GET only; sends IP and `couchlinkd/<version>` user agent). No telemetry upload was found. | `couchlinkd/src/catalog.rs` | Low (privacy) | Off. Needs `COUCHLINK_CATALOG_URL`. |
| 4 | "Virtual screens" setup (elevated, on request) downloads Virtual Display Driver and nefcon at pinned SHA-256, adds the driver's signing certificate to `LocalMachine\TrustedPublisher`, and gives Users modify rights on `C:\VirtualDisplayDriver`. | `couchlinkd/src/vscreen_setup.rs` | Medium (machine-wide trust) | Unchanged. Only use when needed. |
| 5 | Per-seat Steam accounts create a standard Windows user with a random password kept in Credential Manager, then start Steam as that user. Removing the seat deletes the user and password. | `couchlink-steam/src/accounts.rs`, `couchlinkd/src/accounts.rs` | Low | Unchanged. |
| 6 | The Goldberg Steam emulator (gbe_fork) is downloaded at a pinned release with a pinned SHA-256 and copied into game copies, not the original install. | `couchlink-steam/src/goldberg.rs` | Low | Unchanged. |
| 7 | The input hook is injected into each game copy with `VirtualAllocEx` + `CreateRemoteThread(LoadLibraryW)`. Expected for per-copy input; antivirus may flag it. No network code in the hook. | `hook/src/child_hook.cpp` | Low | Unchanged. |
| 8 | The local control service listens on `127.0.0.1` on a random port and requires a random token stored in `%LOCALAPPDATA%\Couchlink\daemon.json`. A web page cannot drive it without the token. | `couchlinkd/src/main.rs`, `ws.rs` | OK | Unchanged. |
| 9 | Recipe limits: file paths must stay inside the game folder, registry edits are HKCU only, every fetch needs a SHA-256. | `couchlink-core/src/recipe.rs` | OK | Unchanged. |

## What `svarus` changes

- `catalog.rs`: default catalog and stats URLs are empty; `Urls::enabled()`.
- `hub.rs`: no catalog or live-recipe fetch unless the catalog is enabled.
- `session/remote.rs`: no built-in recipe URL or public key; `refresh` returns early when off.
- `update.rs`: `check()` returns "no update" unless `COUCHLINK_RELEASES_URL` is set.

## Not covered

- The Flutter app was only scanned for network calls (it talks to the local service and opens links in the browser).
- Third-party downloads (Virtual Display Driver, nefcon, gbe_fork) were checked for pinning, not reviewed themselves.
- Upstream binaries were not compared byte for byte with a local build.
