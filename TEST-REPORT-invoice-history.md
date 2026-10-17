# Test report — invoice notes, attachments & relations

## Automated (run locally, SQLite harness)
- `python qa/invoice_history_test.py` → **79 checks passed**
  (notes/attachments, access rules, link/merge/unlink, index shift on multiple merges,
  tampered-line refusal, cross-batch links, hidden-batch refusal, batch lock, delete guards, audit rows).
- `python qa/mobile_qr_contract_test.py` → 3 checks passed (existing, DB-free).
- Static check (pyflakes) on changed modules: no undefined names. Pre-existing unused imports left as-is.
- MySQL DDL for the three tables generated from the models and the upgrade SQL parsed with a MySQL grammar.

## Screenshots
`qa/screenshots/` — captured from the demo server (`demo_server.py`) with headless Chromium:
01 login, 02 invoice page top, 03 relations & notes section, 04 related link, 05 other-batch invoice, 06 mobile history.

## Not tested in this environment
- **MySQL 8.x / MariaDB runtime** (no server available): run `sql/upgrade_invoice_history.sql` or `upgrade_database.py` in staging first.
- **Android app**: could not be built or unit-tested here (no JDK / Android SDK / Gradle downloads). Static check: all endpoints the app calls (`/api/v1/invoices/<serial>`, `/next`, `/media`, `/me`, `/config`) exist and are unchanged.
- Full legacy Selenium/Playwright QA suites in `qa/` need the original MySQL preview database and sandbox paths; not run.

## Round 2: exclude-from-totals, mobile API, file types, Android

Run with `python qa/exclusion_and_mobile_api_test.py` (SQLite harness): **47 / 47 checks pass.**
Existing `qa/invoice_history_test.py`: **79 / 79 pass** after the change.
`qa/mobile_qr_contract_test.py`: 3 / 3 pass (the 3 QR checks, unchanged).

Covered: merge with exclude and with include (totals count once / twice); unlink restores totals; relation-only link ignores the choice; API link/note/attachment/delete with batch scoping and hidden batch; other clerk refused; executable, SVG and `.html` refused even with an image MIME; CSV and WEBP accepted; sample CSV download works.

Not run here (environment): MySQL-backed tests (`batch_test`, `detection_test`, `sql_upgrade_test`, `startup_test` and others) need a MySQL server; `workflow_test` needs a browser build. Run them on staging. The workflow and browser suites need a running server.

Android (`invoiceflow-android-source-by_agent`): new `NotesLinksApiTest.kt` (Robolectric, stub server) and the new `InvoiceHistoryScreen`. **Not compiled or run**: no Java / Android SDK in the sandbox. Static checks only (brace balance, all called functions exist). Build and run the unit tests before release.


## Round 3: per-user permissions, dead-code cleanup, book average, fresh screenshots

Run with `python qa/permissions_test.py`: **67 / 67 checks pass** (SQLite harness).
Covers: defaults (existing users keep access); admin page (access rules, audit row, presets);
web enforcement (403 JSON for AJAX, flash + redirect otherwise) for invoice create/edit/delete, upload,
scanner upload, notes, links, CSV export and import; hidden buttons in the invoice page; legacy
`can_export` sync; mobile API enforcement (`mobile.access`, notes, links, media, `/me` capabilities);
reset to defaults; another clerk unaffected; book average excludes merged-and-excluded sources.

Re-run after the round: `exclusion_and_mobile_api_test.py` 47/47, `invoice_history_test.py` 79/79,
`mobile_qr_contract_test.py` 3/3.

Bugs found and fixed in this round:
- `invoices.export` crashed (UnboundLocalError) for a user without export permission (inner import shadowed `current_batch_id`).
- Book page average divided by used pages, so an excluded merge source lowered it. Now divided by counted invoices.
- Dead code removed: `app/utils/validators.py` (not imported), unused forms, audit/import/helper functions, unused imports (99 names), duplicate imports, unused locals.

Not run here (environment): MySQL-backed suites (`batch_test`, `detection_test`, `sql_upgrade_test`, `startup_test`, `mysql_runtime_test`, ...) and browser suites (`workflow_test` etc.). The sandbox has no MySQL server and no Playwright browser download. Run them on staging. The workflow and browser suites need a running server.

Screenshots: `docs/screenshots/` (12 PNG, captured with headless Chromium from `qa/screenshots/demo_server.py` + `take_shots.mjs`). Demo accounts are synthetic.

Android (not compiled; no JDK/SDK in the sandbox): new `PermissionsModelTest.kt` (pure JVM), gating in `MainViewModel`, `ScanScreen`, `InvoiceHistoryScreen`, `Me.capabilities` parsing in `ApiService.ping`. Static brace-balance check: all files balanced. Removed unused `toggleManualCropEnabled`, `toggleAutoOpenHandles`, `CropSelection.toRect` and the template `ExampleUnitTest` / `ExampleRobolectricTest`.

## Round 4: note-attachment QR, link lookup, background sync

Server, SQLite harness:
- `qa/note_qr_and_lookup_test.py`: **32 / 32 pass**. Covers QR `target` payload (default invoice, unknown falls back), QR auth and owner access, link lookup (found, hidden, self, missing, empty, bad batch, login), AJAX note with text / file / blocked file type / empty, mobile API attachment-only note, `notes.create` off → 403, rendered page markers.
- `qa/mobile_qr_contract_test.py`: updated for the new `target` key (3/3).
- Re-run: `permissions_test.py` 67/67, `exclusion_and_mobile_api_test.py` 47/47, `invoice_history_test.py` 79/79.

Browser (Chromium headless against `qa/screenshots/demo_server.py`), `qa/screenshots/verify_ui.mjs`: **14 / 14 pass**. Covers link preview, no-match message, submit disabled when not found, link without page reload, history list updates in the background, typed note text survives a background refresh, note saved in the background and list updated, Note QR modal image, no uncaught page errors.

Bugs found and fixed in this round:
- Linked-invoices list did not update after linking (realtime.js skipped regions containing forms). The lists are now opt-in regions (`data-live-allow-forms`); typed or focused fields are still protected.
- Invoice QR endpoints were public. Now login and invoice access required.
- Note file picker used a narrow browser accept list. Now the server's allow-list decides.
- Link serial had no check until submit. Now live lookup.

Screenshots: `docs/screenshots/` (14 PNG; 13 = link preview, 14 = Note QR modal). Demo accounts are synthetic.

Not run here: Gradle build and Android unit tests (no JDK/SDK in the sandbox). Android Kotlin syntax was checked with tree-sitter-kotlin (24 files, 0 parse errors); type checking still needs Android Studio. Also not run: MySQL-backed and live-server suites (see Round 3).


## Round 5: background sync (auto refresh) fix

Symptom: pages needed a manual refresh after another user or device changed data.

Root cause (confirmed in a browser test, not assumed): `realtime.js` treated any non-modal form on a page as an editor. The invoice view has note, link and delete forms, so the whole page was limited to live regions. The invoice's own total therefore stayed stale until a manual reload. Dashboard, summary, invoice list and analytics already updated in the same test.

Fix: only the invoice editor (`form.invoice-form`) and upload file inputs (not the note attachment input) count as editors. Typed or focused fields are still protected by `retain()`, and any dirty field still falls back to region-only updates.

Browser evidence (`qa/live_sync_browser_check.py`, Chromium, SQLite demo server):
- Old `realtime.js`: 9 / 10. Invoice view total did not update (FAIL).
- New `realtime.js`: 10 / 10. Dashboard, summary, invoice list, invoice view and analytics update without reload. A note draft keeps its text and focus. An editor amount draft is kept and the editor is not reloaded.

Known limit: the grid page shows a serial and date presence matrix, so a total change is not visible there. Its refresh request runs, but it is not asserted by the browser check.

Harness changes (test only, `qa/sqlite_harness.py`): MySQL functions `concat`, `lpad`, `date_format`, `dayname` and `hour` are registered for SQLite so the reporting pages run in the demo. Production code is unchanged.

Server regression after this round: permissions 67/67, exclusion and mobile API 47/47, invoice history 79/79, note QR and lookup 32/32, mobile QR contract 3/3.

Not verified here: MySQL runtime (no MySQL server in the sandbox), and the Android build (no JDK/SDK).

## Round 6: admin search across all batches

Problem: an administrator who searched for an invoice had to switch to the batch first, because the batch scope applied to admin search too.

Change (admin only, data-entry users unchanged):
- Invoice list search (serial, book, date) covers every batch. Each result shows a Batch column with the batch name and ID, and a banner says how many batches matched. Browsing without a query still shows the current batch.
- Book number search matches the same number in every batch. The book summary is shown only when one book matches.
- `/invoices/search` (AJAX) covers every batch for admins and returns `batch_id` and `batch_name` per result.
- Mobile API `GET /api/v1/invoices?q=` and `?date=` covers every batch for admins, ignoring the batch selection. Results carry `batch_id` and `batch_name`.
- Link lookup is intentionally unchanged: links stay inside one batch.

Test: `qa/admin_all_batch_search_test.py` (18 checks, SQLite harness). Browser check on the demo: the admin search for `01` shows Batch 1 #1 and Batch 2 #2 in one list (screenshot `docs/screenshots/15-admin-all-batch-search.png`).

Regression after this round: permissions 67/67, exclusion and mobile API 47/47, invoice history 79/79, note QR and lookup 32/32, mobile QR contract 3/3.

Date and book summary cards under all-batch search add totals across all batches and say so in their subtitle. The "Full Details" link is hidden for a book that belongs to another batch, because the book page only opens the current batch.
