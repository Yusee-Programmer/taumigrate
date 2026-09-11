# taumigrate — project proposal

An Alembic-equivalent migrations toolkit for [tauorm](../tauorm) — a
revision DAG (branching + merge migrations, `Migration.down_revisions:
Vec[str]`), a DB-side version-tracking table (multi-head aware), full
upgrade/downgrade, and full **autogenerate**: reflecting a live
database's actual schema and diffing it against tauorm's
`MetaData`/`Table`/`Column` to produce ready-to-run migration SQL,
including composite primary keys, column rename detection, and
Postgres primary-key-change constraint DDL.

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
    migration.tr           # Migration{revision, down_revisions: Vec[str], message, up, down} + build_chain()/heads() (DAG, Kahn's algorithm)
    runner.tr                 # version-tracking table (multi-head aware) + upgrade/downgrade (free generic fns over [C: DbConnection[RS], RS: DbResultSet])
    introspect.tr               # IntrospectedTable/IntrospectedColumn + introspect_schema() (real SQLite, --check-clean-only Postgres)
    diff.tr                       # SchemaOp enum (incl. RenameColumn) + diff_schema(MetaData, Vec[IntrospectedTable]) -- purely structural, no SQL
    autogen.tr                      # GeneratedSql{up,down} + generate_migration_sql() -- diff -> dialect-correct SQL, composite-PK/rename/PG-constraint aware
  tests/
    test_migration_chain.tr    # pure-logic: build_chain()/heads() ordering + every error case, incl. branching/merge (no DB)
    test_introspect_sqlite.tr    # real SQLite: table/column reflection accuracy
    test_runner_sqlite.tr          # real SQLite: upgrade/downgrade chain + version tracking, hand-written migrations
    test_autogen_sqlite.tr           # real SQLite: the full cycle -- introspect/diff/generate/apply/re-introspect/confirm-empty, every SchemaOp kind
    test_composite_pk_sqlite.tr        # real SQLite: composite PK create + widen-via-rebuild
    test_autogen_postgres_pk.tr          # pure function test (no live server): PK-change constraint DDL generation
    test_rename_detection_sqlite.tr        # real SQLite: rename heuristic, alone and combined with an unrelated AlterColumn
    test_branching_sqlite.tr                 # real SQLite: two branches upgraded, merged, and split back apart via downgrade_head
```

### The revision graph

Modeled on Alembic's own runtime design, not Rails/Django's
timestamp-ordered-files model: each `Migration` carries its own
`revision` id and `down_revisions: Vec[str]` — the parent(s) it was
written against. A plain `Migration.init(revision, down_revision, ...)`
(singular, unchanged signature) is the common single-parent case; a
`Migration.init_merge(revision, down_revisions: Vec[str], ...)` declares
a merge migration built on TWO OR MORE parents. `build_chain()` runs
Kahn's algorithm over these links into a topologically valid
root-to-heads order (a real DAG, not a list), and `heads()` returns
every revision nobody else is built on top of. Both raise a descriptive
`DbError` (not a silent best-effort guess) for every real failure mode:
`MIGRATION_MULTI_ROOT` (disconnected root sets — still rejected; a
single connected graph can still branch freely from its one root),
`MIGRATION_NO_ROOT`, `MIGRATION_ORPHAN` (a `down_revisions` entry naming
nothing in the set — checked for every parent of a merge migration too),
`MIGRATION_CYCLE`, `MIGRATION_DUP_REVISION`.

**Supported:** branching (two migrations built on the same parent) and
merging (one migration built on multiple parents, collapsing multiple
heads back into one) — a real multi-head DAG. **Not supported:**
multiple simultaneous DISCONNECTED root sets (two entirely separate
histories in one `Vec[Migration]`), or named branch labels — Alembic's
own `--branch-label` convenience has no equivalent here; a branch is
identified by its own revision id.

### Version tracking

The `taumigrate_version` table holds one row PER CURRENT HEAD (zero rows
means "base", nothing applied; exactly one row is the common
non-branching case). `current_revision()`/`set_revision()` remain the
singular, backward-compatible API for the single-head case (raising
`MIGRATION_MULTIPLE_HEADS` if more than one row exists);
`current_revisions()`/`set_heads()` are the multi-head-aware equivalents
`upgrade_to_heads()`/`downgrade_head()` use internally. This mirrors real
Alembic's own `alembic_version` table, which is likewise multi-row once
multiple heads are live.

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

**Column rename detection.** A heuristic, not name-based matching:
`diff_schema()` treats a table's dropped column and added column as a
RENAME (not an unrelated drop+add) exactly when there's precisely one of
each on that table AND they match shape (same `primary_key`, same
`nullable`, and `types_match()`) — the same conservative heuristic
Alembic's own opt-in `--rename` autogenerate behavior uses. Multiple
simultaneous ambiguous drop/add pairs on one table still fall back to
plain `AddColumn`/`DropColumn` (no guessing which pairs with which).
`autogen.tr`'s SQLite rebuild path is rename-aware: a rename combined
with an unrelated `AlterColumn` on the same table still produces one
correct rebuild, carrying the renamed column's data across under its
NEW name (verified in `test_rename_detection_sqlite.tr`).

**Composite primary keys.** `Table.primary_key_columns()` +
`create_sql()` (tauorm Core) render a trailing `PRIMARY KEY (a, b, ...)`
constraint whenever more than one column is marked `primary_key=true`
(single-column PKs are byte-identical to before — verified against
`test_phase2_sqlite.tr`'s exact-SQL-text assertions). `autogen.tr`
detects a PK-membership change (a column's `primary_key` flag flipping)
and routes it through the SQLite rebuild path (any PK shape) or a
Postgres constraint-level `DROP`/`ADD CONSTRAINT ... PRIMARY KEY (...)`
(once per table, not per column — see below).

**Postgres primary-key changes** are constraint-level DDL
(`ALTER TABLE ... DROP CONSTRAINT <table>_pkey`, then
`ADD CONSTRAINT <table>_pkey PRIMARY KEY (...)`), not a per-column
`ALTER COLUMN` — Postgres has no other way to change which columns are
the primary key. Emitted once per table (after all per-column
`AlterColumn`s), not once per PK-flagged column.

**Remaining honest limitations** (each is a genuine, load-bearing
tradeoff, not an oversight):

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
- **Postgres introspection/diff/autogen is written and `--check`-clean
  but UNVERIFIED against a live server** — no Postgres server is
  available in this environment, matching every other Postgres code path
  in tauorm itself (see its own PROPOSAL.md's Status section). SQLite is
  fully real, run, and verified end-to-end.

## Status

**Done and verified for real, including the full "robust and complete"
follow-on pass** (branching/merge, composite PKs, rename detection,
Postgres PK-constraint DDL). All eight test files run against real,
in-memory SQLite (`test_migration_chain.tr` and
`test_autogen_postgres_pk.tr` are pure logic, no live DB needed) — 140
assertions total, all passing:

- `test_migration_chain.tr` (28 assertions): normal linear ordering,
  branching (two migrations on the same parent, now valid), merging
  (one migration on multiple parents collapsing two heads into one), plus
  every documented graph-validation error case.
- `test_introspect_sqlite.tr` (16 assertions): table/column reflection
  against a schema built by hand via raw SQL (deliberately not through
  tauorm's own `Table`/`Column`, so this doesn't just check "what we
  wrote comes back out") — names, nullability, primary-key detection,
  and `introspected_to_table()` round-tripping into valid DDL.
- `test_runner_sqlite.tr` (17 assertions): a 3-migration hand-written
  chain (create table / add column / create another table) exercising
  `upgrade_one`/`upgrade_to_head`/`downgrade_one`/`downgrade_to`/
  `pending_migrations`/`current_revision`, including real row data
  surviving an `ADD COLUMN` migration — confirmed unchanged/backward-
  compatible after the DAG rewrite.
- `test_autogen_sqlite.tr` (23 assertions): the full autogenerate cycle
  — `CreateTable` → `AddColumn` → `AlterColumn` (the SQLite rebuild
  dance, both directions) → `DropColumn` → `DropTable`, re-introspecting
  and confirming the diff is empty after every single step, with real
  row data surviving each one.
- `test_composite_pk_sqlite.tr` (12 assertions): `CreateTable` with a
  composite PK, then widening a single-column PK into a composite one
  via the SQLite rebuild path, data surviving both directions.
- `test_autogen_postgres_pk.tr` (8 assertions): pure `diff_schema`/
  `generate_migration_sql` function test (no live server needed) —
  single-column→composite PK constraint swap, dropping a PK entirely,
  and confirming an unrelated type change still emits a plain
  `ALTER COLUMN`, unaffected by the PK-handling pass.
- `test_rename_detection_sqlite.tr` (15 assertions): a pure rename (one
  statement, no rebuild, data preserved) and a rename combined with an
  unrelated `AlterColumn` on the same table (rebuild-aware, both the
  renamed AND the altered column's data survive up and down).
- `test_branching_sqlite.tr` (21 assertions): `upgrade_to_heads` applying
  two independent branches off a shared root, `current_revisions`
  reporting both heads, merging them back into one via a merge
  migration, `downgrade_head` splitting the merge apart again, and
  `downgrade_head` on one specific branch leaving the other completely
  untouched.

Also verified: `taupkg build --features sqlite` succeeds end-to-end
through the real package manager (not just a manually-set
`TAURARO_PATH`), matching tauorm's own established verification bar.

### tauraroc bugs found building this

Building taumigrate exercised several generic-function shapes NOTHING in
tauorm itself ever had a concrete reason to use — most centrally, "a
generic function calling ANOTHER generic function, passing its OWN type
parameters as the callee's explicit type arguments"
(`introspect_schema[C,RS]` calling `introspect_sqlite[C,RS](engine)`) —
and the later "make it robust and complete" pass (branching/merge +
composite PK + rename detection) surfaced a 6th — **6 real `tauraroc`
bugs total**, all found via minimal, taumigrate-unrelated repros before
editing anything here, all fixed at the root in
`~/tauraro/src/{sema.tr,codegen/c.tr}`, verified via the same
gen-to-gen self-hosting fixpoint + full `tests/lang`+`tests/regression`
suite convention as every other fix in that project (no regressions
beyond the 3 pre-existing, unrelated failures — `fmt` idempotency on two
example files, one `cdylib` test), then the compiler was re-blessed
after each batch:

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
6. **The most serious bug of the "robust and complete" pass: a
   monomorphized generic function/method body's local-variable bookkeeping
   (`decl_vars` and five sibling maps, plus `cur_self_is_ptr`) was RESET
   but never SAVED/RESTORED across re-entrant monomorphization.**
   `ensure_mono_func_n`, `ensure_mono` (class-generic methods), and
   `ensure_mono_method` (generic methods on non-generic classes) each
   reset this per-function codegen state at the start of generating a
   monomorphized body — correct in isolation, but these functions can run
   RE-ENTRANTLY, mid-generation of an ENCLOSING function's own body: a
   call inside a loop to ANOTHER generic function, using the enclosing
   function's own still-abstract type params as the callee's explicit
   type args (exactly the shape bug #1's `resolve_nested_mono_callee`
   handles — the fix that made this reachable at all), triggers
   monomorphization for that callee right there, mid-loop. Resetting
   `decl_vars` for the INNER function's own body permanently wiped the
   OUTER function's already-declared locals, so a later reassignment in
   the outer body (`count = count + 1`, after a nested generic call earlier
   in the same loop) miscompiled as "'count' undeclared" — surfaced
   verbatim by `runner.tr`'s `upgrade_to_heads[C,RS]` (a `count` local,
   incremented in a loop that also calls `_apply_one[C,RS](...)`,
   itself calling further `[C,RS]`-parameterized helpers). Reproduced
   with a minimal, taumigrate-unrelated repro (two generic functions,
   `outer[C,RS]` looping and calling `inner[C,RS]`, incrementing its own
   `count` local in between) before touching anything here. Fixed by
   saving the caller's prior `decl_vars`/`str_local_names`/
   `coll_local_sfx`/`coll_local_idict`/`coll_local_strval`/
   `coll_local_vtcoll`/`cur_self_is_ptr` before the reset in all three
   call sites, and restoring them after the monomorphized body finishes
   generating.

### tauorm changes

Two small, additive changes to tauorm itself were needed:

- **`col_types.tr` gained a `Raw(text: str) -> ColumnType`** passthrough
  kind (`ddl(dialect)` returns `text` verbatim, ignoring `dialect`) — see
  "Remaining honest limitations" above for why introspected columns need
  this rather than a guessed semantic kind.
- **Composite primary key DDL**: `column.tr` gained
  `ddl_no_pk(dialect) -> str` (same as `ddl()` but never renders an
  inline `PRIMARY KEY`); `table.tr` gained
  `primary_key_columns() -> Vec[str]`, and `create_sql()` now renders a
  trailing `PRIMARY KEY (a, b, ...)` constraint whenever more than one
  column is PK-flagged (single-column PKs are byte-identical to before).

Both are purely additive; every existing call site and `ColumnType`
kind is unchanged. `Raw` is re-exported from `tauorm.tr`'s hub alongside
the other `col_types` constructors.

## What's deferred, and why

- **A CLI** (`taumigrate revision --autogenerate -m "message"`,
  `taumigrate upgrade head`, ...) — everything above is a library API;
  wiring a command-line tool around it is straightforward but wasn't
  asked for and has no test harness yet. A natural, small follow-on.
- **Named branch labels** (Alembic's `--branch-label` convenience) —
  branches here are identified by their own revision id; no concrete
  need for a separate labeling layer yet.

Composite primary keys, branching/merge migrations, column rename
detection, and Postgres PK-change constraint DDL — all previously listed
here as deferred — are now implemented; see "Remaining honest
limitations" above and the Status section for what each one does and
does not cover.
