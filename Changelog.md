# Changelog

All notable changes to ValoonSync are documented in this file.

## [1.1.13] - 2026-06-30

### Fixed
- Update dialog: the download progress bar no longer shows a clipped, misaligned percentage. The slim 8px bar was still drawing Qt's built-in "%" text crammed into those few pixels; the numeric progress is already shown in the status line below the bar (e.g. "Downloading… 12.3 / 48.0 MB"), so the bar is now a clean indicator only.
- **Username no longer disappears from file names.** Some media and reports arrive without an attributable author (e.g. forwarded or auto-ingested WhatsApp media, where `creatorId` is empty). The `{creator}` slot in the naming pattern then resolved to nothing, collapsing the separators and dropping the username segment entirely (e.g. `26001121_2026-06-24_IMG.jpeg` instead of `26001121_<user>_2026-06-24_IMG.jpeg`). Authorless files now use a language-specific marker (Unknown / Desconocido / Unbekannt / Inconnu) so the position is always preserved, which also prevents two authorless files sharing the same date and name from colliding.
- Files already synced under an outdated name are now renamed in place on the next full sync instead of keeping the stale name, so the fix above also reaches previously downloaded media and reports (a numeric deduplication suffix is still preserved, not mistaken for a name change).

### Changed
- **Mandatory updates (always the latest version).** When a newer version is published, the client now shows a non-dismissable update wall on startup explaining that the update is required and that the app will no longer work until it is applied. The previous "Later"/snooze option is gone — the only choices are **Update now** or **Close app**. A failed download/install re-enables the button so the user is never trapped, and the check still fails open: with no connectivity no wall appears (the client cannot sync against Valoon anyway).
- Release source moved to the organization account: the update check and the release workflow now target `valoon-mexico/Valoon-Sync-Releases` instead of the personal account. Clients installed before this version keep updating through GitHub's automatic redirect from the old path.

## [1.1.12] - 2026-06-25

### Added
- **Parallel downloads.** Within each project, media and report files now download in a small pool (3 at a time by default) instead of one after another, significantly cutting sync time on large projects. Tunable via `download_concurrency` in `config.json` — set it to `1` to restore fully sequential downloads. Naming/deduplication and database writes stay single-threaded, so only the byte transfer runs concurrently.
- **Server-side failure visibility.** When the backend can't deliver a file — a request timeout, a `5xx`, or a report PDF the server never generates — it is now reported as **pending** ("server, will retry") in the sync feed and the completion summary, kept separate from real errors. The file is left undownloaded and retried automatically on the next cycle, making it clear the cause is the server, not the app.
- A prominent **bold amber notice** appears when a report download stalls past 20 s ("The server is taking long to deliver a file — waiting…"), so a sync waiting on the backend no longer looks frozen. The existing 429 "Server busy" notice now shares the same emphasis.

### Changed
- **Fewer API calls per sync cycle.** Media is re-listed in full only when the server-side item count changes (validated against the local database); otherwise an incremental listing since the last watermark is used. The per-cycle report list also drops its heavy unused form payload (~98% of the response) before caching, cutting memory use.
- **Faster failure on unresponsive downloads.** Building on 1.1.11, a download that receives no bytes is now cut loose immediately without retrying, with a tighter 30 s window for report PDFs (media keeps the 90 s window for legitimately slow large transfers). One stuck file can no longer hold up the rest of the sync, and with the download pool a single stalled file ties up only one worker while the others keep going.

### Fixed
- Files that share the same number and title no longer churn a spurious `[MOVE]` on every sync — a dedup-suffix difference in the computed name was being misread as a relocation.
- Incremental cycles now keep already-synced media accounted for in the feed ("N already synced"), so a cycle that re-lists nothing no longer looks like media stopped downloading.

## [1.1.11] - 2026-06-16

### Fixed
- Report sync no longer freezes when the backend accepts the request for a report's PDF but never delivers it (observed with reports in `draft` state). The download worker was left blocked waiting for a response with no effective timeout — the session's read-timeout retries stacked with the manual download retry loop, turning one stuck PDF into a multi-minute freeze that stalled every report after it. Downloads now use an explicit `(connect, read)` timeout and stop retrying read-timeouts, so a non-responding download fails fast (~90 s read window for legitimate on-demand PDF generation), is counted as an error, and the sync continues with the remaining reports.

## [1.1.10] - 2026-06-12

### Added
- New **"moved"** status in the sync feed: files relocated after an organization or folder change are now reported as `[MOVE]` instead of `[SKIP]`, so it is clear they were placed in their new location rather than omitted. A "Moved: N" counter appears next to Downloaded/Skipped only when relocations happen, and completion messages include the moved count.
- Relocations performed upfront when switching from weekly to flat organization, and consolidations of week folders left by other language prefixes, are now reported per file in the feed as well (previously they showed as plain skips).
- A successful language folder rename (e.g. `Media` → `Multimedia`) posts an informational notice to the feed.
- The tray notification distinguishes moved files: "N new files, M moved" / "M files moved" instead of "all up to date" when a sync only relocated files.

## [1.1.9] - 2026-06-11

### Added
- New organization mode for media and reports: **shared folder for all projects**. All files are stored flat in a single user-chosen folder, relying on the naming pattern (project name / abbreviation variables) to tell projects apart. Already-synced files are moved there automatically on the next sync, and the app never deletes user-owned subfolders inside the shared folder.
- Organization labels reworked to make the scope explicit: "Per project — weekly subfolders", "Per project — all together", "Shared folder for all projects". Folder pickers for the shared folders live in Settings → Storage, with save-time validation and a warning when the naming pattern cannot distinguish projects.
- Rate-limit visibility: while the API backs off on a 429 the sync screen now shows "Server busy — retrying in X s (attempt N/5)" instead of appearing frozen. The server's `Retry-After` header is honoured (capped at 120 s).

### Fixed
- Auto-update: the app now relaunches automatically after a silent update install. The installer's `[Run]` entry only applied to interactive installs, so after an auto-update the app stayed closed.
- Downloads interrupted mid-stream are retried (2 attempts with backoff) and partial files are removed. Previously a dropped connection could leave a corrupt file on disk that was never cleaned up.
- Configuration saves are now atomic (temp file + replace): a crash mid-write can no longer truncate `config.json` and wipe the user's settings.
- Changing the language no longer corrupts database paths when a project or file name happens to contain a week-prefix fragment (e.g. `-SC-`): the rename is anchored to the `YYYY-PREFIX-WW` pattern.
- `safe_filename` hardened against Windows reserved device names (`CON`, `NUL`, `COM1`…), control characters, trailing dots/spaces, and over-long names (200-char cap).
- Quitting the app stops background threads with a bounded wait. Quitting mid-sync could crash on exit, and a silent auto-update could run the installer while files were still being written.
- The installer download can be cancelled cleanly (Esc, dialog close, or quitting from the tray); the partial file is removed and the app no longer risks a crash on teardown.
- "Open folder" opens the folder files actually land in: it honours the per-project folder override and the shared media folder. It used to open — and recreate — the old or global folder.
- Relocating files after an organization or folder change now uses a cross-volume-safe move (works when the destination is on another drive) with re-download fallback; a failed move no longer leaves partial files or aborts the sync.
- Unreachable destination folders (e.g. a disconnected network drive) are reported as readable per-project errors instead of killing the whole sync queue.

## [1.1.8] - 2026-06-09

### Fixed
- Blank white window when launching the app while it was already running minimized to tray: the second instance forced the existing window visible via raw Win32 `ShowWindow`, bypassing Qt so the content was never repainted. The second instance now signals a named event (`ValoonSyncActivate`) and the running instance shows its window through Qt from the GUI thread.
- Restoring from a taskbar-minimized state now uses `showNormal()` (`show()` alone kept the window minimized).
- Foreground activation: the second instance hands its foreground rights to the running instance (`AllowSetForegroundWindow`) so the restored window reliably comes to front.
- ctypes hardening in the single-instance logic: `use_last_error=True` + `ctypes.get_last_error()` for reliable error checks, explicit `HANDLE` restypes to avoid 64-bit handle truncation, explicit `WaitForMultipleObjects` signature with `WAIT_FAILED` handling (prevents a silent busy-loop on invalid handles), and fail-fast checks when the named events cannot be created.

## [1.1.7] - 2026-06-08

### Fixed
- Startup freeze: the user profile (`get_current_user`) was fetched synchronously on the GUI thread right after auto-login. Combined with the 429 retry backoff in the API client, a rate-limited response could block the Qt event loop, leaving an unresponsive white/black window that could only be closed via Task Manager. The call now runs in a background worker like every other API request.
- Keyring access moved off the GUI thread: reading stored credentials at startup (auto-login) now runs in a worker together with validation, and saving/clearing credentials on login/logout runs in a daemon thread. Prevents the Windows Credential Manager from stalling the event loop on startup or during login/logout.

## [1.1.6] - 2026-05-11

### Added
- Settings → General now displays the currently installed version next to the "Updates" section title
- 4 new random greeting phrases on the projects screen (9 total)

### Fixed
- Update dialog buttons ("Later", "Update now") now show the pointer cursor on hover
- Auto-release workflow trigger changed to `workflow_dispatch` to prevent accidental runs before RELEASES_PAT is configured

## [1.1.5] - 2026-05-10

### Added
- Auto-update system via GitHub Releases: app checks for new versions on startup and notifies the user
- Update dialog showing current vs. new version, collapsible release notes, and download progress bar
- Multi-language release notes: write `[ES]`, `[EN]`, `[DE]`, `[FR]` sections in GitHub Release body — each user sees notes in their configured language
- Manual "Check for updates" button in Settings → General
- `GET /repos/Cantuuuu/Valoon-Sync-Releases/releases/latest` used as the update source (public repo, no auth required for clients)
- GitHub Actions release workflow (`.github/workflows/release.yml`) ready for future automated builds

### Changed
- Update check runs 5 seconds after startup (non-blocking background thread) to avoid interfering with app load
- "Later" on the update dialog snoozes the notification for 3 days

### Fixed
- Update dialog scroll area background forced to white to prevent dark-theme bleed
- Partial installer download file cleaned up on network error

## [1.1.4] - 2026-05-09

### Added
- Custom 429 rate-limit handling with exponential backoff for all API requests

## [1.1.3] - 2026-05-08

### Added
- Granular date variables for file naming: year, month, and day for both created and updated dates
- README with project overview, setup, build, and architecture documentation
- CHANGELOG documenting all releases from v1.0.1 through v1.1.3
- GitHub Actions CI workflow (pytest on Python 3.11/3.12/3.13, Windows)

### Changed
- Translated all new date variable labels and tooltips to ES, DE, FR

### Removed
- Legacy `sync_engine.py` module (replaced by `CompositeSyncEngine` in production)

### Fixed
- `.gitignore` missing patterns for `.pytest_cache/`, `.mypy_cache/`, `.ruff_cache/`, `.coverage`
- Tag v1.1.1 missing annotation message

## [1.1.2] - 2026-04-29

### Changed
- Overhauled Settings screen with sidebar navigation and stacked pages
- Centralized settings styles into styles.py
- Added tooltips, reset button, save toast, and unsaved-changes indicator

### Added
- Chip-based pattern editor with drag-and-drop pills for file naming
- User name greeting on project screen and account info in settings
- SVG chevron indicator for dropdown comboboxes

### Fixed
- Chip duplication when dragging within pattern editor
- Sidebar navigation styling (QSS braces)

## [1.1.1] - 2026-04-27

### Added
- Syntax highlighting for naming pattern variables
- API-driven naming variables (creator, status resolution)
- Free-text naming pattern system replacing block-based tokens

### Fixed
- FileExistsError on media rename and download
- Deduplication during consolidation

## [1.1.0] - 2026-04-23

### Added
- PDF report sync with download
- Per-project folder selection
- Configurable file naming system with token chips
- Media and report organization toggle (weekly/flat)
- Per-project Media/Reports sync toggles
- Report organization toggle (weekly/flat)

### Fixed
- Weekly folder consolidation with foreign prefixes
- Duplicate folders when renaming fails
- Language-specific abbreviations in UI columns
- Empty week folders after organization mode change
- File rename when naming config changes

## [1.0.7] - 2026-04-15

### Added
- Default language resolved from installer registry and OS locale

## [1.0.6] - 2026-04-14

### Fixed
- Invisible checkboxes in compiled .exe

## [1.0.5] - 2026-04-13

### Added
- Terms and conditions in installer (DE/EN/ES/FR)
- Checkboxes with checkmark instead of solid fill

### Fixed
- Stop sync cancels the full project queue

## [1.0.4] - 2026-04-09

### Added
- Multi-entityType support
- Close app button

## [1.0.3] - 2026-04-07

### Added
- Rate limiting for sync (10 requests/10 min)
- Year prefix in weekly folders
- Weekly folder prefix by language (SC/CW/KW) with auto-rename
- Rounded logo

## [1.0.2] - 2026-04-06

### Added
- Full internationalization with Qt Linguist (ES, EN, DE, FR)
- Scroll on settings screen for small windows
- Selection checkboxes and "Select all" in project list

## [1.0.1] - 2026-04-02

### Added
- Manual sync interval configuration
