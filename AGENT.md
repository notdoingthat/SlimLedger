# Project rules
Offline single-user Windows ledger app. Electron + React + TypeScript + SQLite (better-sqlite3).

- Money is stored as INTEGER paisa. Convert only for display (formatMoney helper).
- Never store balances. Compute from ledger_entries.
- All multi-table writes use db.transaction().
- Reference records by id, never by name.
- Issued documents are cancelled (status/is_void), never deleted. Drafts may be deleted.
- Business logic goes in src/main/services/. UI components never run SQL.
- The renderer talks to the database only through IPC handlers defined in src/main/ipc/.
- Every service function that does calculation gets a Vitest test.
- No internet access, no CDN links, no remote fonts. Everything bundled locally.
- Documents are printed on A4. Follow the A4 rules in SPEC.md section "A4 printing".
- Make small changes. Do not refactor unrelated files. Tell me which files you changed.
- Dates stored as ISO text (YYYY-MM-DD).