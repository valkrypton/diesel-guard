# AGENTS.md - diesel-guard

Detects unsafe Postgres migration patterns before they cause production incidents. Parses SQL using `pg_query` (Postgres's actual parser via libpg_query) and identifies operations that acquire dangerous locks or trigger table rewrites. Supports both built-in Rust checks and user-defined Rhai script checks.

**Core Tech:** Rust, `pg_query`, Rhai scripting, Diesel/SQLx migrations, Postgres 9.6+

## Architecture

- `src/checks/pg_helpers.rs` — Public AST navigation functions; `extract_node()` unwraps `RawStmt` → `NodeEnum`
- `src/checks/mod.rs` — `Check` trait (`Send + Sync`); `check()` takes `(node, config: &Config)`; `Registry` with `add_check()`, `check_stmts_with_context(stmts, sql, ignore_ranges, config)`
- `src/parser/mod.rs` — `parse()` wraps `pg_query::parse()`, `parse_with_metadata()` adds safety-assured blocks
- `src/safety_checker.rs` — Entry point; loads custom Rhai checks from `custom_checks_dir` config
- `src/scripting.rs` — `CustomCheck` (implements `Check`), sandboxed Rhai engine, `load_custom_checks()`
- `src/ast_dump.rs` — `dump_ast()` for `dump-ast` CLI subcommand (JSON AST output)
- `src/config.rs` — Config with `custom_checks_dir: Option<String>`, `postgres_version: Option<u32>`; `disable_checks` warns (not errors) on unknown names

## CLI Commands

- **`check <path> [--format text|json]`** — Check migration file or directory for unsafe operations. Pass `-` as path to read SQL from stdin. Loads `diesel-guard.toml` from CWD (warns and uses defaults if missing). Exits with code 1 if violations found, 0 if clean.
- **`init [--force]`** — Creates `diesel-guard.toml` from bundled template. Errors if file exists unless `--force` is passed.
- **`dump-ast --sql <SQL> | --file <PATH>`** — Parse SQL and print pg_query AST as JSON. Exactly one of `--sql` or `--file` required. Useful for writing custom Rhai checks.

## Configuration (`diesel-guard.toml`)

- **`framework`** (required): `"diesel"` or `"sqlx"`. Case-sensitive. Default (when no config file): `"diesel"`.
- **`start_after`** (optional): Timestamp to skip older migrations. Accepts `YYYYMMDDHHMMSS`, `YYYY_MM_DD_HHMMSS`, or `YYYY-MM-DD-HHMMSS`. Separators are normalized before comparison, so any format works against any migration naming convention.
- **`check_down`** (optional, default `false`): Include down/rollback migration files in checks.
- **`disable_checks`** (optional): List of check names to skip. Unknown names produce a warning (not an error), so custom check names can be listed safely. Mutually exclusive with `enable_checks`.
- **`enable_checks`** (optional): Whitelist — only these checks run. Mutually exclusive with `disable_checks`; setting both is a `ConfigError`.
- **`postgres_version`** (optional): Target Postgres major version as integer (e.g., `16`). Used by version-aware checks (e.g., `AddColumnCheck` skips the rewrite warning on PG 11+ when default is a constant).
- **`custom_checks_dir`** (optional): Path to directory containing `.rhai` script files for custom checks.

## Safety-Assured Blocks

Wrap SQL statements in `-- safety-assured:start` / `-- safety-assured:end` comment directives to suppress all checks (built-in + custom) for enclosed statements. Case-insensitive. No nesting — a second `start` before `end` is an error. Unclosed blocks and unmatched `end` directives are also errors.

```sql
-- safety-assured:start
ALTER TABLE users DROP COLUMN legacy_field;
-- safety-assured:end
```

## Framework Adapters

- **Diesel**: `migrations/<TIMESTAMP>_<name>/{up.sql, down.sql}`. Timestamp formats: `YYYY_MM_DD_HHMMSS`, `YYYY-MM-DD-HHMMSS`, or `YYYYMMDDHHMMSS`.
- **SQLx**: 2 formats supported:
  1. **Suffix-based** (reversible): `<VERSION>_<DESC>.up.sql` / `<VERSION>_<DESC>.down.sql`
  2. **Single file** (up-only): `<VERSION>_<DESC>.sql`

  SQLx versions are any positive integer (e.g., `1`, `001`, `42`, `20240101000000`). Short numeric versions use numeric comparison for `start_after` filtering; 14-digit timestamps use string comparison.

## How to Add a Built-in Check

1. **Create** `src/checks/your_check.rs` — implement `Check` trait: `fn check(&self, node: &NodeEnum, _config: &Config) -> Vec<Violation>` (use `_config` if unused, `config` if version-aware). Add `#[cfg(test)]` unit tests using the macros from `test_utils.rs` (see below).
2. **Register** in `src/checks/mod.rs` — add `mod`, `pub use`, and `register_check` call (all alphabetically). Check names are derived from struct names automatically.
3. **Create fixtures** — `tests/fixtures/your_operation_{safe,unsafe}/up.sql`. First line MUST be `-- Safe: ...` or `-- Unsafe: ...`.
4. **Update integration tests** in `tests/fixtures_test.rs` — add to `safe_fixtures` vec, add detection test, update `test_check_entire_fixtures_directory` counts.
5. **Update docs** — create `docs/src/checks/<check>.md` with bad/good examples and add entry to `docs/src/SUMMARY.md`.
6. **Verify** — `cargo test && cargo fmt --check && cargo clippy --all-targets --all-features -- -D warnings`

## Test Macros (`src/checks/test_utils.rs`)

All check unit tests should use these macros rather than calling `check()` manually:

- `assert_detects_violation!(Check, sql, "OPERATION NAME")` — exactly 1 violation with matching operation
- `assert_detects_violation_with_config!(Check, sql, "OPERATION NAME", config)` — same with explicit config
- `assert_detects_violation_containing!(Check, sql, "OPERATION", "substring1", ...)` — 1 violation, problem contains substrings
- `assert_detects_n_violations!(Check, sql, n, "OPERATION NAME")` — exactly N violations with same operation
- `assert_detects_n_violations_any_containing!(Check, sql, n, "substring", ...)` — N violations, any contains substrings
- `assert_allows!(Check, sql)` — no violations
- `assert_allows_with_config!(Check, sql, config)` — no violations with explicit config

## pg_helpers Quick Reference

Key functions beyond `extract_node()`:

- `alter_table_cmds(node)` → `Option<(table_name: String, cmds: Vec<&AlterTableCmd>)>` — entry point for any `ALTER TABLE` check
- `cmd_def_as_column_def(cmd)` → `Option<&ColumnDef>` — for `ADD COLUMN` commands
- `cmd_def_as_constraint(cmd)` → `Option<&Constraint>` — for `ADD CONSTRAINT` commands
- `column_has_constraint(col, ConstrType::ConstrDefault as i32)` — test for a specific constraint type on a column
- `for_each_column_def(node)` → `Vec<(table_name, &ColumnDef)>` — for `CREATE TABLE` column inspection
- `drop_object_names(objects)` → `Vec<String>` — extract names from `DROP` statement object list
- Type predicates: `is_serial_pattern()`, `is_json_type()`, `is_char_type()`, `is_short_integer()`, `is_timestamp_without_tz()`

## How Custom Rhai Checks Work

- Users place `.rhai` files in a directory and set `custom_checks_dir` in `diesel-guard.toml`
- Each script receives a `node` variable (pg_query AST node serialized via `rhai::serde::to_dynamic()`) and a `config` variable (the current `diesel-guard.toml` settings)
- Scripts access fields like `node.IndexStmt.concurrent`, `node.CreateStmt.relation.relname`; access config like `config.postgres_version` (integer or `()` when unset)
- Return protocol: `()` = no violation, `#{ operation, problem, safe_alternative }` = one, array of maps = multiple
- Check name = filename stem (e.g., `require_concurrent.rhai` → `require_concurrent`); disableable via `disable_checks`
- Safety-assured blocks automatically skip custom checks (same `check_stmts_with_context` path)
- Engine limits: `max_operations(100_000)`, `max_string_size(10_000)`, `max_array_size(1_000)`, `max_map_size(1_000)`
- Runtime errors and invalid return values are logged as warnings to stderr, never panic
- `dump-ast` CLI subcommand helps users inspect AST structure: `diesel-guard dump-ast --sql "..."`. Output strips outer `RawStmt`/`Node` wrappers — the JSON starts directly at the concrete node type (e.g. `{"IndexStmt": {...}}`), matching what a Rhai script receives as `node`
- Reference scripts in `examples/`: `no_unlogged_tables.rhai`, `require_concurrent_index.rhai`, `require_if_exists_on_drop.rhai`, `no_truncate_in_production.rhai`, `limit_columns_per_index.rhai`, `require_index_name_prefix.rhai`
- **`pg` constants module** — Scripts can use pg_query protobuf enum values via the `pg::` prefix:
  - `pg::OBJECT_*`: INDEX, TABLE, COLUMN, DATABASE, SCHEMA, SEQUENCE, VIEW, FUNCTION, EXTENSION, TRIGGER, TYPE
  - `pg::AT_*`: ADD_COLUMN, COLUMN_DEFAULT, DROP_NOT_NULL, SET_NOT_NULL, DROP_COLUMN, ALTER_COLUMN_TYPE, ADD_CONSTRAINT, DROP_CONSTRAINT, VALIDATE_CONSTRAINT
  - `pg::CONSTR_*`: NOTNULL, DEFAULT, IDENTITY, GENERATED, CHECK, PRIMARY, UNIQUE, EXCLUSION, FOREIGN
  - `pg::DROP_*`: RESTRICT, CASCADE

## Naming Conventions

- **Check structs**: `YourOperationCheck`
- **Tests**: `test_detects_*` (violation found), `test_allows_*` (safe variant), `test_ignores_*` (unrelated operation)
- **Fixtures**: `{operation}_{safe|unsafe}` or `{operation}_{variant}_{safe|unsafe}`

## Non-Obvious Gotchas

- **RenameStmt separation**: `ALTER TABLE t RENAME COLUMN/TO` is `RenameStmt` in pg_query, NOT `AlterTableStmt`. Check `rename_type` field to distinguish column vs table renames.
- **FK columns vs constraint keys**: `constraint_columns_str()` reads from `Constraint.keys` — works for UNIQUE/CHECK/PK. FK columns are in `fk_attrs`, not `keys`.
- **Protobuf default values**: Fields with value 0 may be omitted. Match on node type presence rather than `subtype == 0`.
- **Fixture counts**: When adding fixtures, update both file count and violation count in `test_check_entire_fixtures_directory`. Some fixtures produce multiple violations due to check overlaps — read the assertion message for the breakdown.
- **Macros position**: Keep macros before `mod test_helpers` in `test_utils.rs` to avoid `clippy::items_after_test_module`.
- **Rhai `sync` feature**: Required because `Check` trait is `Send + Sync`. Without it, `CustomCheck` won't compile.
- **`Violation.operation` is `String`**: Not `&'static str`. Changed to support runtime-built strings from Rhai. `"literal".into()` works automatically for built-in checks.
- **`disable_checks` validation is relaxed**: Unknown names produce a warning, not an error, so users can disable custom check names without the validator rejecting them.
- **`enable_checks`/`disable_checks` are mutually exclusive**: Setting both raises `ConfigError::ConflictingCheckConfig`. The Registry calls `config.is_check_enabled(name)` — don't bypass this.
- **Unknown check name validation runs after custom checks load**: `SafetyChecker::with_config()` compares `disable_checks`/`enable_checks` names against both `Registry::builtin_check_names()` AND the stems of loaded `.rhai` files. This means a custom check name is only validated after scripts are compiled — typos in custom check names are caught at startup.
- **`extract_node()` in `pg_helpers`**: Use this instead of manually unwrapping `raw_stmt.stmt.as_ref().and_then(|n| n.node.as_ref())`. It's the single source of truth for `RawStmt → NodeEnum` extraction.
- **`alter_table_cmds()` vs direct node matching**: For any `ALTER TABLE` check, always use `alter_table_cmds()` — it handles both `AlterTableStmt` node matching and table name extraction in one call. Do not match `NodeEnum::AlterTableStmt` directly.
- **Version-aware checks**: Compare `config.postgres_version >= Some(N)` — `None` means unknown version, so the safe path is to flag the violation when version is unset.
- **`Registry::new()` ignores config**: Creates all checks enabled. Only `Registry::with_config(config)` respects `enable_checks`/`disable_checks`. Don't call `Registry::new()` directly if config filtering matters.
- **Diesel adapter skips `start_after` for single migration directories**: When the user passes a specific migration directory (detected by the presence of `up.sql` directly inside it), the `start_after` filter is bypassed entirely — the assumption is explicit intent. This is handled by `is_single_migration_dir()`.
- **Safety-assured directive is strict**: The regex requires the directive to be the only content on the line — no trailing text, not even a comment (`-- safety-assured:start -- reason` will NOT match). A non-matching line is simply not a directive, so a typo in `start` leaves the block unclosed.
- **`load_custom_checks()` is non-fatal**: Script compilation errors are collected and returned as warnings; they don't abort initialization. Files without `.rhai` extension are silently skipped. Scripts are loaded in alphabetical order for deterministic behavior.
