# Invoice-flow-for-udpate-V2-04-to-V2-04

InvoiceFlow: invoice entry web app (Flask + MySQL) and the Android scanner app.

## What is in this repository

| File / folder | What it is |
|---|---|
| `invoiceflow-server-source.zip` | Full server source (folder `invoiceflow-server/server`). |
| `invoiceflow-android-source-by agent.zip` | Android app source (folder `android/`). Build in Android Studio. |
| `invoiceflow-server-changes.patch` | Patch of the server source against the previous zip (note-attachment QR, live link lookup, background sync, login for QR images). |
| `invoiceflow-screenshots.zip` / `docs/screenshots/` | Current screenshots from the demo server (12 images). |
| `invoiceflow-android-source-functional source by google.zip` | Separate earlier Android project, kept unchanged. |

## Highlights of this update

- **Per-user permissions.** Admin → Users → the Permissions button on each user row. For each data-entry user the admin switches on or off: mobile access, invoice create/edit/delete, upload and delete documents, add and delete notes, link and unlink invoices, CSV export, CSV import. Buttons hide and the server refuses the action. Defaults keep today's access.
- **Notes, attachments and invoice links** with optional amount merge. For a merge, you choose whether the other invoice is excluded from totals.
- **Note attachment QR:** Notes & comments → *Note QR*. Scan it with the Android app to attach a photo or file to a new note on that invoice.
- **Live link lookup:** type a serial in *Link another invoice* and see the other invoice's date, serial, book/page and total before linking. Linking and notes save in the background; typed text is kept.
- **Android:** notes & links screen; actions follow the admin's switches.
- Dead code removed; unused imports cleaned up.

## Database upgrade (existing installations)

Read `invoiceflow-server-source.zip` → `invoiceflow-server/server/DEPLOY-CHECKLIST.md`. Four new tables are added: `invoice_notes`, `invoice_note_media`, `invoice_links`, `user_capabilities`. Either run `python upgrade_database.py` (automatic) or the SQL files `sql/upgrade_invoice_history.sql` and `sql/upgrade_user_permissions.sql` (manual, MySQL Workbench). Nothing existing is altered.

## Tests

Server: `python qa/permissions_test.py` (67), `python qa/exclusion_and_mobile_api_test.py` (47), `python qa/invoice_history_test.py` (79), `python qa/mobile_qr_contract_test.py` (3), `python qa/note_qr_and_lookup_test.py` (32). Browser: `qa/screenshots/verify_ui.mjs` (14, against the demo server). Android unit tests run with Gradle in Android Studio. See `TEST-REPORT-invoice-history.md` for what was and was not run.
