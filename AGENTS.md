# Repository Guidelines

# IMPORTANT:
# Always read memory-bank/architecture.md before writing any code. Include entire database schema.
# Always read memory-bank/design-document.md before writing any code.
# Always read memory-bank/prd.md before writing any code.
# Always emphasize modularity (multiple files) and discourage a monolith (one giant file).
# After adding a major feature or completing a milestone, update memory-bank/architecture.md.

Guide for contributors to the Japanese yen ledger app. Planning docs are the current source of truth; keep them in sync with code or migration changes.

## Project Structure & Module Organization
- Root docs: `prd.md`, `design-document.md`, `design-discussion.md`, `tech-stack.md`. Keep them updated when behavior or schema changes.
- When code lands, use `app/` for Swift packages (iOS/macOS client), `scripts/` for automation, and `supabase/migrations/` for SQL. Keep names aligned with the docs: entries, entry_tags, budgets, fixed_expense_templates, fixed_expense_runs.

## Build, Test, and Development Commands
- Docs-only today; update Markdown before proposing features.
- When the client exists (SwiftPM + Xcode 15+), use `xcodebuild -scheme Ledger -configuration Debug build` and `swift test`.
- For Supabase, manage schema with `supabase db diff` and `supabase db push`; commit the generated SQL migrations.

## Coding Style & Naming Conventions
- Swift 5.9+, SwiftUI, async/await; 4-space indent, ~120-char lines. Use Swift Package Manager only; avoid extra UI dependencies.
- Map models to schema: `Entry`, `EntryTag` (role primary/secondary), `Budget` (locked), `BudgetRollover`, `FixedExpenseTemplate/Run`; currency fixed to JPY with integer yen amounts.
- Follow Swift API Design Guidelines; prefer value types, `os_log`, and observable state via Observation/Combine.

## Testing Guidelines
- Frameworks: XCTest / Swift Testing. Cover sync queue ordering, budget lock on first entry, fixed expense generation, and conflict resolution flow.
- Name tests `test_{Scenario}_Should{Outcome}`; keep fixtures deterministic with in-memory SQLite/GRDB and stubbed Supabase calls (no live network). Focus on amount math, date handling, and tag role rules.meiyi

## Commit & Pull Request Guidelines
- Commits: short, imperative, and scoped (e.g., `add design doc and tech stack`, `refine PRD`). Avoid mixing migrations, client code, and docs in one commit when possible.
- PRs: include summary, linked issue, migration note (if any), scenarios tested, and screenshots for UI. Call out sync/serialization changes explicitly.
- Update the relevant docs in the same PR when altering workflows, schema, or terminology.

## Security & Configuration Tips
- Keep Supabase keys and device IDs out of the repo; use env files or Xcode config, and store tokens in Keychain. Do not commit CSV exports or logs containing user data.
- Default currency is JPY; avoid floating-point for amounts. Preserve soft-delete (`deleted_at`), `rev`, and `device_id` fields in client models and payloads.
