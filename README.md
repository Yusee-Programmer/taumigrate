# taumigrate

An Alembic-equivalent migrations toolkit for [tauorm](../tauorm) — a
revision chain, a DB-side version-tracking table, upgrade/downgrade, and
full **autogenerate** (reflect a live database's schema, diff it against
tauorm's `MetaData`, get back ready-to-run migration SQL).

**Status: done and verified for real.** See [PROPOSAL.md](PROPOSAL.md)
for the full design, the SQLite rebuild-dance for column changes (SQLite
has no `ALTER COLUMN`), known/honest limitations, and the `tauraroc`
compiler bugs this surfaced (all fixed upstream). 72 assertions across
4 test files, all passing against real, in-memory SQLite.

```python
from taumigrate import Migration, upgrade_to_head, downgrade_one
from sqlite_dialect import SqliteConnection, SqliteResultSet
from engine import Engine

def create_users_up(dialect: str) -> Vec[str]:
    mut s: Vec[str] = Vec[str].init(1)
    s.push("CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT NOT NULL)")
    return s

def create_users_down(dialect: str) -> Vec[str]:
    mut s: Vec[str] = Vec[str].init(1)
    s.push("DROP TABLE users")
    return s

def main():
    mut engine: Engine[SqliteConnection, SqliteResultSet] = ...

    mut migrations: Vec[Migration] = Vec[Migration].init(1)
    migrations.push(Migration.init("1", "", "create users", create_users_up, create_users_down))

    mut applied: Result[int, DbError] = upgrade_to_head[SqliteConnection, SqliteResultSet](engine, "sqlite", migrations)
```

Autogenerate — diff a live DB against tauorm's `MetaData` and get real
SQL back:

```python
from taumigrate import introspect_schema, generate_migration_sql

mut live = introspect_schema[SqliteConnection, SqliteResultSet](engine, "sqlite")?
mut gen  = generate_migration_sql(md, live, "sqlite")   # gen.up / gen.down: Vec[str]
```

## Choosing a backend

Same optional-dependency pattern as tauorm itself — this package's own
tests need a real driver to run against:

```
taupkg build --features sqlite     # needs ../tausqlite3
taupkg build --features postgres   # needs ../taupostgres (written, --check-clean, unverified live)
```

## License

Apache-2.0.
