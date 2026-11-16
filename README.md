# Schemity Migration Check

Reports what the SQL migrations in a pull request do, before they run:

- what each statement does to existing data (drops it, can fail on it,
  rewrites the table, holds back other statements) and which other tables it
  reaches through foreign keys;
- design findings the change introduces (a foreign key with no index, a table
  with no primary key, and about 45 more).

It needs no database and no credentials. The schema a migration runs against
is rebuilt by reading the older migration files in the repo, or a schema dump
you already commit. Nothing is executed, and nothing is sent anywhere: the
files are read on your runner and a report is printed.

## Use

```yaml
name: Migrations
on: pull_request

permissions:
  contents: read
  pull-requests: read

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: schemity/ci@v1
        with:
          dialect: postgres
          migrations: db/migrations
```

Findings appear as notices on the changed lines and as a report in the job
summary. The job passes whatever the report says, unless you set `fail-on`.

## Inputs

| Input | Default | |
|---|---|---|
| `dialect` | required | `postgres`, `mysql`, `mariadb`, `sqlserver`, `sqlite`, `snowflake`, `cockroachdb` |
| `migrations` | required | The folder that holds the SQL migration files |
| `command` | `check` | `analyze` (what the statements do), `lint` (design findings), `check` (both) |
| `schema` | | A schema dump to start from, see below |
| `before` | the migrations folder | Older migrations to replay, one path per line |
| `files` | the pull request's files | The migration files to check, one per line |
| `fail-on` | | What fails the job, see below |
| `version` | `latest` | The CLI release, such as `v0.1.0` |

### Failing the job

`fail-on` takes a list separated by commas:

| Value | Fails when the report holds |
|---|---|
| `any` | anything at all |
| `impact` | any finding about what a statement does |
| `lint` | any design finding the change introduces |
| `unread` | a statement that could not be read |
| `loses-data`, `can-fail`, `storage-engine`, `rewrites-table`, `holds-back`, `dependents`, `spreads`, `changes-data` | a finding of that kind |

```yaml
        with:
          dialect: postgres
          migrations: db/migrations
          fail-on: loses-data,can-fail
```

### A history that cannot be replayed

On an old database the first migrations often no longer describe what is
there. If the repo holds a schema dump (`pg_dump --schema-only`, Rails'
`structure.sql`), start from it, and name the migrations newer than the dump:

```yaml
        with:
          dialect: postgres
          migrations: db/migrations
          schema: db/structure.sql
          before: |
            db/migrations/2026
```

With `schema` set and no `before`, only the dump is read.

## Which migrations work

Plain SQL files, read in name order: Prisma, Drizzle, Flyway, Liquibase's SQL
format, golang-migrate, goose, dbmate, sqlx, Supabase. For Django and Laravel,
print the SQL first (`manage.py sqlmigrate`, `artisan migrate --pretend`) and
pass the result as `files`.

A statement that cannot be read is listed under "Not analysed", never passed
over in silence. Row counts and views are not known here: there is no
database. The Schemity desktop app reports those against a live connection.

## The command line tool

The Action downloads the `schemity` binary from this repo's Releases and
checks it against `checksums.txt`. The same binary runs anywhere:

```
schemity check --dialect postgres --before db/migrations db/migrations/0042_orders.sql
schemity --help
```

The binary is free to use. Its source is not in this repo. The licences of
the open source software it contains are in `THIRD_PARTY_LICENSES.txt` inside
each download.
