# Database backends: an interface first, then PostgreSQL

**Goal:** SQLite stays the default. PostgreSQL becomes an opt-in second backend,
picked with one env var. Every database call becomes async, and both backends
share one set of SQL.

## Where things stand

- `src/database.js` builds and upgrades the schema on every start: `CREATE TABLE
  IF NOT EXISTS`, `addColumn()`, and one-time steps guarded by marker keys in
  `server_settings`. About 15 other modules add their own tables or steps from there.
- About 1,800 `db.prepare()` calls and 74 transactions across 86 files, all synchronous
  (better-sqlite3). Modules get `db` from `getDb()` (16 files) or as a factory
  argument (`setupSocketHandlers(io, db)`, `createPermissions(db)`).
- SQLite-only pieces: FTS5 search (`searchIndex.js`), 10 triggers, backup via
  `VACUUM INTO`, restore by swapping `haven.db`, the size cap reading the file
  size, and `secure_delete`.
- Timestamps are text like `2026-10-22 19:39:00`. Server and browser code append
  `'Z'` to parse them in about a dozen places.
- About 200 test files, 64 of which use better-sqlite3 directly. CI doesn't run the
  test suite today.
- Precedent to copy: `src/storage/` picks `LocalStorage` or `S3Storage` in
  `fromEnv()`, and the S3 library is an add-on installed only when turned on.

## Decisions

1. **Async, same shape.** Callers keep `db.prepare(sql).get/all/run`, now
   awaited. Converting is mostly adding `await`, and SQL strings don't change.
   Postgres runs on a `pg` pool, with no blocking bridge.
2. **One set of SQL, no ORM.** Write portable SQL, plus a few `db.sql` helpers for the
   bits that differ. An ORM or query builder would mean rewriting all ~1,800
   queries.
3. **Postgres stores what SQLite stores, for v1.** Timestamps stay text in the
   same format, flags stay integers, and case-insensitive names become `citext`.
   The client gets identical rows, and moving data over is a straight copy.
   Real timestamp types can come later.
4. **One schema.** `database.js` keeps building it, and the Postgres driver
   translates the handful of SQLite type names. `database.js` changed ~100 times
   in the last three months, so two hand-kept schemas would drift. Triggers and
   search get a Postgres version inside the module that owns them today.
5. **SQLite stays the default.** Desktop and one-click installs don't change.
6. **`pg` is an add-on, like S3.** It's installed into the data folder on the first
   start with Postgres on, so a normal install never downloads it.
7. **PostgreSQL 18 or newer.**

## Layout

Shaped like `src/storage/`:

```
src/database.js                    schema, initDatabase(), getDb() as now; opens its driver from src/db
src/db/index.js                    fromEnv(): HAVEN_DATABASE_URL set → Postgres, else haven.db
src/db/sqlite.js                   SqliteDatabase: better-sqlite3, pragmas, the statement cache
src/db/postgres.js                 PostgresDatabase: pool, placeholder + DDL translation, transactions
src/db/pgAddon.js                  installs and loads pg, like src/storage/s3Addon.js
addons/postgres/                   package.json + package-lock.json pinning pg
scripts/migrate-db-to-postgres.js  like scripts/migrate-uploads-to-s3.js
test/dbHarness.js                  like test/socketServerHarness.js: a fresh DB for the backend under test
.github/workflows/test.yml         npm test on SQLite, and on Postgres from Phase 4
```

Changed in place: `searchIndex.js`, `protectedMessages.js`, `badges.js` (the Postgres
versions sit beside the SQLite ones), `routes/backup.js`, `backupExtract.js`,
`routes/maintenance.js`, `server.js` (async startup), `.env.example`,
`docker-compose.yml`, `GUIDE.md`.

## The interface

| Member | Does |
|---|---|
| `kind` | `'sqlite'` or `'postgres'`, like `storage.kind` |
| `prepare(sql)` | statement with `get`, `all`, `run` (→ `{ changes }`), `iterate`; `?` and `@name` work on both |
| `exec(sql)` | several statements, mostly schema |
| `transaction(fn)` | runs `fn` in a transaction, and helpers called inside join it. Nesting makes a savepoint |
| `hasTable`, `hasColumn`, `addColumn` | what `database.js` asks `PRAGMA` and `sqlite_master` today |
| `sql.now()`, `sql.like`, `sql.greatest()`, `sql.contains()`, `sql.jsonEach()` | the few things whose SQL text differs |
| `sizeBytes()`, `close()` | for the size cap and shutdown |

`src/db/index.js`, in the house style:

```js
'use strict';

// Where Haven's tables live. initDatabase() opens one of these and every query
// goes through it, so the backend is chosen in one place, the way src/storage
// chooses local disk or a bucket.

const { DB_PATH } = require('../paths');

// HAVEN_DATABASE_URL=postgres://… keeps the tables in PostgreSQL; unset is
// haven.db in the data folder, as always. Read once.
function fromEnv(env = process.env) {
  const url = String(env.HAVEN_DATABASE_URL || '').trim();
  if (!url) {
    const { SqliteDatabase } = require('./sqlite');
    return new SqliteDatabase(DB_PATH);
  }
  if (!/^postgres(ql)?:\/\//i.test(url)) throw new Error('HAVEN_DATABASE_URL must be a postgres:// URL');
  const { PostgresDatabase } = require('./postgres');
  return new PostgresDatabase({ url, max: parseInt(env.HAVEN_DATABASE_POOL, 10) || 10 });
}

module.exports = { fromEnv };
```

What a converted call site looks like (registration in `src/auth.js`):

```js
result = await db.transaction(async () => {
  if (inviteRow) {
    const took = await db.prepare('UPDATE invite_codes SET spent = spent + 1 WHERE id = ? AND (max_uses <= 0 OR spent < max_uses)').run(inviteRow.id);
    if (!took.changes) throw Object.assign(new Error('invite used up'), { inviteUsedUp: true });
  }
  // RETURNING works on both backends; it replaces result.lastInsertRowid
  return db.prepare(
    'INSERT INTO users (username, password_hash, is_admin, avatar) VALUES (?, ?, ?, ?) RETURNING id'
  ).get(username, hash, isAdmin, avatarPath);
});
if (isAdmin) await grantAdminRole(db, result.id);
```

## Phases

| # | Phase | Size | Postgres usable? |
|---|---|---|---|
| 0 | Safety net | S | no |
| 1 | Interface, SQLite only | S | no |
| 2 | Async, module by module | XL | no |
| 3 | Portable SQL (alongside 2) | M | no |
| 4 | Postgres driver | L | in CI |
| 5 | Data, backups, docs, release | M | for users |

Every phase ships on its own. Phases 0–3 change nothing for users.

### Phase 0: Safety net
- [ ] `.github/workflows/test.yml` runs `npm test` on every push and PR.
- [ ] `test/dbHarness.js` opens whatever database the server under test uses. Move
      the 64 tests that use better-sqlite3 directly onto it, a batch at a time.

### Phase 1: The interface, SQLite only
- [ ] Add `src/db/index.js` and `src/db/sqlite.js`, and move the pragmas and statement
      cache out of `database.js`.
- [ ] `initDatabase()` builds the schema through the driver, and `hasColumn` and
      `addColumn` become driver methods. `getDb()` keeps its name.
- [ ] While the conversion runs, the SQLite driver answers synchronously, so both
      awaited code and not-yet-converted code keep working.

### Phase 2: Go async, module by module
Convert leaf helpers first (`protectedMessages.js`, `forumUnread.js`, …), then the
socket handler factories, then routes, and finally `server.js` startup (`server.js:3134`)
and its timers. Use small PRs on `main`, not a long-lived branch, because
`database.js` moves weekly.

Rules for converted code:
- `await` every database call, and make the function and its callers `async`.
- Don't `await` files or network inside `db.transaction`. In tests, the SQLite
  driver flags a transaction that yields to the event loop.
- Make check-then-write a single statement (`ON CONFLICT`, `UPDATE … WHERE`, as in
  the example above) or a transaction. Async handlers can interleave between the
  check and the write, which can't happen today.
- Socket events that must stay in order (send, edit, delete) go through a small
  per-socket queue.
- Wrap async Express routes, because Express 4 doesn't catch rejected promises.
- `userHasPermission` (~170 calls) and `getUserEffectiveLevel` (~70) run several
  queries each and are the biggest cascade. Convert them as they are, and add a
  permission cache later only if Postgres latency shows up.
- Once everything awaits, the SQLite driver returns real Promises too, so a missed
  `await` fails on both backends. Its single connection then needs a small queue,
  so another request's query never lands inside an open transaction.

### Phase 3: Portable SQL (alongside Phase 2)

| SQLite today | Becomes | Count |
|---|---|---|
| `INSERT OR IGNORE` | `INSERT … ON CONFLICT DO NOTHING` | 74 |
| `INSERT OR REPLACE` | `INSERT … ON CONFLICT (key) DO UPDATE`. Check each one: REPLACE deletes the row first, which fires `ON DELETE CASCADE` | 55 |
| `.lastInsertRowid` | `INSERT … RETURNING id` with `.get()` | 86 |
| `datetime('now', …)`, `julianday()`, `CURRENT_TIMESTAMP` in writes | a time computed in JS and passed in, or `db.sql.now()` | ~40 |
| `LIKE` (case-insensitive in SQLite only) | `db.sql.like`, which is `ILIKE` on Postgres | 45 |
| `COLLATE NOCASE` in queries | `citext` columns, `LOWER()` elsewhere | ~15 |
| `MAX(a, b)` | `db.sql.greatest(a, b)` | 8 |
| `instr(a, b)` | `db.sql.contains(a, b)` | 7 |
| `json_each`, `json_extract`, `json_valid` | `db.sql.jsonEach()`, or JS | 4 spots |
| `rowid` | the table's own id | 2 |
| `sqlite_master`, `PRAGMA table_info` | `db.hasTable()`, `db.hasColumn()` | ~8 |

Also check:
- 27 `GROUP BY` queries, because Postgres rejects selected columns that aren't grouped.
- NULL sort order where it matters: SQLite puts NULLs first, Postgres last.
- Comparisons in `SELECT` lists, which return `true` rather than `1` on Postgres.

### Phase 4: Postgres driver
- [ ] `src/db/pgAddon.js` and `addons/postgres/` pin and load `pg`.
- [ ] `src/db/postgres.js`:
  - rewrites `?` and `@name` to `$1…`, skipping quoted text, cached per SQL string
  - `run()` returns `{ changes: rowCount }`, and `iterate()` reads through a cursor
    in batches
  - parses `int8` and `numeric` as numbers; otherwise `COUNT(*)` comes back as the
    string `'0'`
  - uses one pool client per transaction, found through `AsyncLocalStorage`, so
    helpers join it without passing it around
  - in Postgres an error aborts the whole transaction, so the few places that
    catch an error mid-transaction and carry on get a savepoint
  - gives a plain-English message when it can't connect, like
    `explainOpenFailure()` does for file permissions
- [ ] Translate DDL in `CREATE TABLE` and `ADD COLUMN`:

| SQLite | Postgres |
|---|---|
| `INTEGER PRIMARY KEY AUTOINCREMENT` | `BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY` |
| `INTEGER` | `BIGINT` |
| `DATETIME`, `DEFAULT CURRENT_TIMESTAMP` | `TEXT`, defaulting to UTC now in SQLite's format |
| `TEXT … COLLATE NOCASE` | `CITEXT` |
| `REFERENCES …` | the same plus `DEFERRABLE`, so the migration can load tables in any order |

- [ ] Write the triggers in PL/pgSQL next to the SQLite ones: the protected-message
      guard, released uploads, and badge counters.
- [ ] Search in `searchIndex.js`, chosen by the same `HAVEN_SEARCH_TOKENIZER`. The
      search query in `messages.js` comes from there, so the handler doesn't care
      which backend runs. Ranking will differ slightly.

| Tokenizer | SQLite | Postgres |
|---|---|---|
| `unicode61` (default) | FTS5 | `tsvector` ('simple') + GIN |
| `porter` | FTS5 | `tsvector` ('english') + GIN |
| `trigram` | FTS5 | `pg_trgm` GIN + `ILIKE` |

- [ ] Add a second CI job with a `postgres:18` service that runs the suite with
      `HAVEN_DATABASE_URL` set.

### Phase 5: Data, backups, docs, release
- [ ] `scripts/migrate-db-to-postgres.js [--data <dir>]`, run with Haven stopped:
  - creates the schema through `initDatabase()`, copies each table in batches
    inside one transaction, resets identity sequences, rebuilds search, and
    compares row counts
  - starts with a pre-flight check that lists rows Postgres would reject (NUL
    characters, text in integer columns, orphaned references)
  - never touches `haven.db`, so going back is just unsetting the variable
- [ ] Backup and restore: on Postgres the backup zip carries each table as JSON
      lines, using the migration script's copier, and restore loads them in one
      transaction. `VACUUM INTO` and the WAL checkpoint move into `SqliteDatabase`.
      `GUIDE.md` also points to `pg_dump`.
- [ ] Maintenance: the size cap uses `db.sizeBytes()`. Postgres doesn't hand
      space back after `DELETE`, so measure live table size there.
- [ ] Config and docs:
  - `.env.example` gets an "External database (PostgreSQL)" section:
    `HAVEN_DATABASE_URL`, `HAVEN_DATABASE_POOL`, and TLS via `?sslmode=require`
  - `docker-compose.yml` gets a commented-out `postgres` service
  - `GUIDE.md` gets a section, and `CHANGELOG.md` an entry
  - the known differences get written down: search ranking, and `secure_delete`
    (#5699), which has no Postgres equivalent (deleted rows stay on disk until
    vacuum reuses the space)
- [ ] Label Postgres "experimental" for a release or two.

## Code style to keep

- CommonJS, `'use strict';`, 2-space indent, single quotes. Each new file opens with a
  plain-English comment on what it holds and why.
- Drivers are classes with a `kind`, chosen once by `fromEnv(env = process.env)`,
  as in `src/storage/`.
- Log lines are prefixed by area (`[db]`), like `[storage]` and `[search]`.
- Scripts start with `#!/usr/bin/env node`, put a usage line in the header, take
  `--data <dir>`, and are safe to run again.
- Tests use `node:test` and `node:assert/strict`, with the run command in the header
  and their own temp `HAVEN_DATA_DIR`.
- No new frameworks: no ORM, no TypeScript, and no Express 5 in the same change.

## Open questions

1. Is a script enough for moving to Postgres in v1, or should the admin panel offer
   it too? I'd ship the script first.
2. Is it OK to require the `citext` extension? Every major managed Postgres has
   it. Without it, usernames need `LOWER()` indexes and query changes.
3. For built-in backup on Postgres: a JSON export inside the zip (my
   recommendation, since it shares code with the migration), or `pg_dump` only?
