# taumigrate — project proposal

An Alembic-equivalent migrations toolkit for [tauorm](../tauorm) — a
single-linear-chain revision model (Migration.revision/down_revision), a
DB-side version-tracking table, upgrade/downgrade, and full
**autogenerate**: reflecting a live database's actual schema and diffing
it against tauorm's `MetaData`/`Table`/`Column` to produce ready-to-run
migration SQL.

This is tauorm's own Phase 6 ("Migrations", explicitly deferred in
[tauorm's PROPOSAL.md](../tauorm/PROPOSAL.md) — "an Alembic equivalent...
MetaData/Table/Column is the same introspectable schema representation a
migration tool would diff against, so nothing here forecloses it") —
built as its own sibling `taupkg` package (not folded into tauorm itself)
since migrations are a distinct, optional concern most tauorm consumers
never need to depend on, exactly mirroring how the two DB drivers
(taupostgres/tausqlite3) are their own packages rather than living inside
tauorm.

## Why a separate package, not a tauorm module

- **Optional by nature.** Plenty of tauorm consumers never run a
  migration (a read-only reporting tool, a script against an
  already-provisioned DB, a test suite that just calls `create_all`).
  Forcing every tauorm consumer to pull in migration machinery would be
  the same mistake tauorm's own dialect-optionality already avoids for
  the two DB drivers.
- **`from taumigrate import ...`, a clean, separate import surface** —
  consumers who DO want migrations add one more `taupkg` dependency,
  exactly like adding the `sqlite`/`postgres` feature today.
- Depends on tauorm (path dependency) for the schema representation
  (`Table`/`Column`/`MetaData`/`col_types`) and the dialect-agnostic
  `DbConnection`/`DbResultSet`/`Engine` boundary every runner/introspection
  function is generic over — never a concrete driver directly, same
  reasoning as tauorm's own hub.

## Architecture

```
taumigrate/
  taupkg.toml          # deps: tauorm (required), postgres/sqlite3 (optional, for THIS package's own tests)
  src/
    taumigrate.tr        # hub -- `from taumigrate import Migration, upgrade_to_head, ...`
    migration.tr           # Migration{revision, down_revision, message, up, down} + build_chain()
    runner.tr                 # version-tracking table + upgrade/downgrade (free generic fns over [C: DbConnection[RS], RS: DbResultSet])
    introspect.tr               # IntrospectedTable/IntrospectedColumn + introspect_schema() (real SQLite, --check-clean-only Postgres)
    diff.tr                       # SchemaOp enum + diff_schema(MetaData, Vec[IntrospectedTable]) -- purely structural, no SQL
    autogen.tr                      # GeneratedSql{up,down} + generate_migration_sql() -- diff -> dialect-correct SQL
  tests/
    test_migration_chain.tr    # pure-logic: build_chain() ordering + every error case (no DB)
    test_introspect_sqlite.tr    # real SQLite: table/column reflection accuracy
    test_runner_sqlite.tr          # real SQLite: upgrade/downgrade chain + version tracking, hand-written migrations
    test_autogen_sqlite.tr           # real SQLite: the full cycle -- introspect/diff/generate/apply/re-introspect/confirm-empty, every SchemaOp kind
```

### The revision chain

Modeled on Alembic's own runtime design, not Rails/Django's
timestamp-ordered-files model: each `Migration` carries its own
`revision` id and the `down_revision` (parent) it was written against.
`build_chain()` walks these links into the single canonical root-to-head
order, and raises a descriptive `DbError` (not a silent best-effort
guess) for every real failure mode: `MIGRATION_MULTI_ROOT`,
`MIGRATION_NO_ROOT`, `MIGRATION_MULTI_HEAD` (two migrations built on the
same parent — a branch), `MIGRATION_ORPHAN` (a `down_revision` naming
nothing in the set), `MIGRATION_CYCLE`, `MIGRATION_DUP_REVISION`.

**Scope: a single linear chain only** — Alembic's own branch/merge
support (multiple heads merged back into one, a real DAG rather than a
list) is deliberately not built. Nothing in this project has a concrete
need for it yet, and it's a meaningfully bigger design than a linear
chain; picking it up if/when a real multi-branch workflow shows up
matches how tauorm itself has deferred every other speculative feature
throughout its own phases.

### Version tracking

ONE tracking table, `taumigrate_version`, holding at most a SINGLE row —
the current head revision — not a full applied-migrations history log,
because the chain itself (the `down_revision` links) already IS the
history; the DB only needs to remember "where in the chain am I right
now". No row at all means "base" (nothing applied). This mirrors real
Alembic's own `alembic_version` table exactly.

### Autogenerate

`introspect_schema()` reflects a live database's actual tables/columns.
`diff_schema()` compares that against tauorm's `MetaData` and produces a
flat `Vec[SchemaOp]` (`CreateTable`/`DropTable`/`AddColumn`/`DropColumn`/
`AlterColumn`) — purely structural, no SQL text, so it doubles as a
dry-run "what's pending?" report on its own. `generate_migration_sql()`
turns that into dialect-correct `up`/`down` SQL:

- **Postgres** renders every op directly — `CREATE`/`DROP TABLE`,
  `ADD`/`DROP COLUMN`, `ALTER COLUMN ... TYPE` / `SET`/`DROP NOT NULL`.
  Postgres supports all of these natively.
- **SQLite** is different for exactly one case: it has no `ALTER TABLE
  ... ALTER COLUMN` at all, ever (a permanent design choice, not a
  version gap — this project vendors SQLite 3.53.4, which DOES support
  `ADD`/`DROP COLUMN` directly, used for those ops). A column
  type/nullable/primary-key CHANGE instead gets the standard
  "12-step" workaround: create a new table under a temp name with the
  target shape, `INSERT ... SELECT` the surviving data across the
  columns common to both shapes, drop the old table, rename the new one
  into place — generated as a real, multi-statement `up`/`down` pair
  (verified reversible in both directions, including real row data
  surviving the round trip, in `test_autogen_sqlite.tr`).

**Known, honest limitations** (each is a genuine, load-bearing tradeoff,
not an oversight):

- **Column renames are not detected** — a renamed column reads as an
  unrelated drop + add, since name-based matching can't distinguish
  them. This is the identical limitation real Alembic autogenerate has;
  reviewing a generated migration before running it is expected, exactly
  as it is with Alembic.
- **SQLite type comparison is best-effort, not exact**, for a real
  reason: SQLite's type-AFFINITY system collapses several distinct
  Tauraro-level `ColumnType` kinds onto the identical declared DDL text
  (`integer`/`bigint`/`boolean` all declare `"INTEGER"`;
  `text`/`varchar`/`datetime` all declare `"TEXT"` — see
  `col_types.tr`). `diff.tr` compares RENDERED DDL text
  (`ColumnType.ddl(dialect)`) against the introspected raw text
  specifically to avoid false-positive "changed" diffs from this
  collapse — reversing a bare `"INTEGER"` back to one specific semantic
  kind isn't possible without guessing, so `introspect.tr` doesn't try;
  introspected columns carry a `ColumnType.Raw(text)` passthrough (a
  small, additive change to tauorm's own `col_types.tr`) instead of a
  guessed kind.
- **A primary-key column is normalized to non-nullable on introspection**
  regardless of what SQLite itself reports for its `NOT NULL` flag —
  needed because of a real, well-known SQLite quirk: a bare `INTEGER
  PRIMARY KEY` column (no explicit `NOT NULL` written) reports
  `notnull=0` via `PRAGMA table_info`, even though it's the table's
  primary key. Without this normalization, every table tauorm itself
  creates (`Column.pk()` always forces `nullable = false`) would show a
  spurious "column changed" diff on every single autogenerate run,
  forever.
- **A primary-key CHANGE via Postgres `ALTER COLUMN` is not generated** —
  that needs constraint-level `ADD`/`DROP CONSTRAINT` DDL, not a
  per-column `ALTER`, and is rare enough in an autogenerated migration
  (usually a deliberate, hand-written change) to leave as an explicit,
  documented gap rather than guess at.
- **Postgres introspection/diff/autogen is written and `--check`-clean
  but UNVERIFIED against a live server** — no Postgres server is
  available in this environment, matching every other Postgres code path
  in tauorm itself (see its own PROPOSAL.md's Status section). SQLite is
  fully real, run, and verified end-to-end.

## Status

**Done and verified for real.** All four test files run against real,
in-memory SQLite (`test_migration_chain.tr` is pure logic, no DB
needed) — 72 assertions total, all passing:

- `test_migration_chain.tr` (16 assertions): normal linear ordering, plus
  every documented chain-validation error case.
- `test_introspect_sqlite.tr` (16 assertions): table/column reflection
  against a schema built by hand via raw SQL (deliberately not through
  tauorm's own `Table`/`Column`, so this doesn't just check "what we
  wrote comes back out") — names, nullability, primary-key detection,
  and `introspected_to_table()` round-tripping into valid DDL.
- `test_runner_sqlite.tr` (17 assertions): a 3-migration hand-written
  chain (create table / add column / create another table) exercising
  `upgrade_one`/`upgrade_to_head`/`downgrade_one`/`downgrade_to`/
  `pending_migrations`/`current_revision`, including real row data
  surviving an `ADD COLUMN` migration.
- `test_autogen_sqlite.tr` (23 assertions): the full autogenerate cycle
  — `CreateTable` → `AddColumn` → `AlterColumn` (the SQLite rebuild
  dance, both directions) → `DropColumn` → `DropTable`, re-introspecting
  and confirming the diff is empty after every single step, with real
  row data surviving each one.

Also verified: `taupkg build --features sqlite` succeeds end-to-end
through the real package manager (not just a manually-set
`TAURARO_PATH`), matching tauorm's own established verification bar.

### tauraroc bugs found building this

Building taumigrate exercised several generic-function shapes NOTHING in
tauorm itself ever had a concrete reason to use — most centrally, "a
generic function calling ANOTHER generic function, passing its OWN type
parameters as the callee's explicit type arguments"
(`introspect_schema[C,RS]` calling `introspect_sqlite[C,RS](engine)`) —
and surfaced **4 more real `tauraroc` bugs**, all found via minimal,
taumigrate-unrelated repros before editing anything here, all fixed at
the root in `~/tauraro/src/{sema.tr,codegen/c.tr}`, verified via the same
gen-to-gen self-hosting fixpoint + full `tests/lang`+`tests/regression`
suite convention as every other fix in that project (no regressions
beyond the 3 pre-existing, unrelated failures — `fmt` idempotency on two
example files, one `cdylib` test), then the compiler was re-blessed:

1. **A generic function calling another generic function, passing its
   own type parameters as the explicit type args, never re-mangled the
   callee through the active monomorphization substitution.** An
   explicit generic call `f[T1,T2](...)` mangles its callee to
   `f__MONO_T1_T2` once, at sema time, as literal text from whatever
   names were written at the call site. When that call site sits inside
   ANOTHER generic function's own body, `T1`/`T2` are the OUTER
   function's own still-abstract type parameters — sema can't know what
   they'll resolve to, since that's only decided per-instantiation,
   during codegen's own monomorphization of the outer function. Without
   a fix, the baked name was emitted completely unchanged in every
   instantiation ("implicit declaration of function 'inner__MONO_C_RS'"
   — a real copy for the actual concrete types was never generated
   either, since nothing had asked for it). Fixed in `codegen/c.tr`'s
   `gen_call` + a new `resolve_nested_mono_callee` helper: splits the
   mangled name, resolves each type-arg segment through
   `resolve_generic_tyname`, and — if anything actually resolved to
   something new — triggers `ensure_mono_func`/`_n` for the now-concrete
   target and uses the corrected name.
2. **Explicit generic-call type inference never applied `throws`-Result
   wrapping** to the inferred call type (both the single- and
   multi-type-arg cases in `sema.tr`), unlike the ordinary/implicit call
   path. A `throws`-declared function's C-level return type is always
   `Result`, but the explicit-call HIR type stayed the bare, unwrapped
   success type — so `codegen`'s own `return <this call>` inside another
   throws function tried to box the whole `Result` struct into a fresh
   `Result_Ok`'s `void*` payload slot ("cannot convert to a pointer
   type"). Fixed by adding the identical `Result[V,Err]`-wrapping
   `register_decl` already does for the ordinary case, to both explicit
   generic-call branches.
3. **`return <call>` inside a `throws` function had no case for the
   returned call's own type ALREADY being the correctly-shaped
   aggregate** (`Result`/`Option`/`Tuple`) — every branch assumed a bare
   scalar/string/pointer value needing a FIRST wrap into `Result_Ok`,
   so returning another throws function's result directly (now correctly
   typed as `Result` by fix #2) got wrapped a SECOND, redundant, and not
   even well-typed time. Fixed in `codegen/c.tr`'s `SReturn` handling: a
   `Result`/`Option`/`Tuple`-typed return expression is now emitted
   verbatim, no re-wrapping.
4. **Reassigning an already-declared variable via `x = f()?` (an
   `SAssign`, as opposed to `mut x = f()?`, an `SLet`) had NO real
   `?`-propagation support at all** — it fell through to `gen_expr`'s
   own generic `ETryExpr` fallback, whose own comment admits "proper
   propagation handled at statement level" (untrue for this exact
   shape): it neither checked `Result.tag` (silently treating an `Err`'s
   payload as if it were `Ok`, since `.data.Ok.val`/`.data.Err.err` alias
   the same union slot) nor unboxed a `str` `Ok` value correctly
   (emitting a bare `TrStr*` pointer where the target's own type is a
   `TrStr` VALUE struct). Fixed by adding a proper, dedicated `SAssign`
   case (mirroring `SLet`'s own existing one) for an `EIdent` target.
5. *(Pre-existing, module-boundary-only bug, same session)* **A private
   (non-`pub`) GENERIC free function's forward declaration, in a
   multi-MODULE build specifically, rendered its own unsubstituted
   parameter types literally** (`Engine[C,RS]` as if `C`/`RS` were real
   class names → `"unknown type name 'Engine_C_RS'"`) — every OTHER
   prototype-emission loop in `codegen/c.tr` already excludes generics
   for exactly this reason (only monomorphized copies get real
   declarations); this one loop, specific to `generate_module_c`'s
   private-function forward-declaration pass, didn't. Nothing had
   exercised a private generic free function in a multi-module build
   until `runner.tr`'s `_run_statements` helper. Fixed by adding the
   matching `generics.len == 0` guard.

### tauorm changes

One small, additive change to tauorm itself was needed:
**`col_types.tr` gained a `Raw(text: str) -> ColumnType`** passthrough
kind (`ddl(dialect)` returns `text` verbatim, ignoring `dialect`) — see
"Known, honest limitations" above for why introspected columns need this
rather than a guessed semantic kind. Purely additive; every existing
`ColumnType` kind and call site is unchanged. Re-exported from
`tauorm.tr`'s hub alongside the other `col_types` constructors.

## What's deferred, and why

- **Multi-head branching / merge migrations** — see "The revision chain"
  above. No concrete need yet.
- **`ALTER TABLE ... RENAME COLUMN` detection** — would need heuristic
  matching (similar name + similar type at a similar position) Alembic
  itself only offers as an opt-in, explicitly-fuzzy `--rename` heuristic
  in some autogenerate plugins, not core behavior. Reviewing a generated
  migration remains the expected workflow either way.
- **Composite / multi-column primary keys** — matches tauorm Core's own
  existing single-column PK convention throughout (`Column.pk()`); not
  attempted here either.
- **A CLI** (`taumigrate revision --autogenerate -m "message"`,
  `taumigrate upgrade head`, ...) — everything above is a library API;
  wiring a command-line tool around it is straightforward but wasn't
  asked for and has no test harness yet. A natural, small follow-on.
