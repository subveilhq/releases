# Subveil releases

Prebuilt binaries for the [Subveil](https://subveil.dev) CLI.

Subveil creates safe, smaller copies of a **PostgreSQL or MySQL** database. It subsets data while keeping foreign keys valid, and it anonymizes PII deterministically. It runs inside your network or CI, so raw data never leaves your infrastructure.

This repository only hosts release binaries and release notes. Documentation is at **https://subveil.dev/docs**.

| Engine | Support |
|---|---|
| PostgreSQL 14+ | Supported |
| MySQL 8.0+ | Supported (tested on MySQL 8.0) |
| MariaDB 10.6+ | Expected to work with `engine: mysql`; not yet part of our test suite |

## What's new in v0.2.0

**MySQL support.** Subveil can now subset and anonymize MySQL, with the same workflow as PostgreSQL.

- Set `engine: mysql` in `subveil.yaml`. MySQL accepts `mysql://` URLs or Go driver DSNs.
- Reads from a consistent snapshot (`START TRANSACTION WITH CONSISTENT SNAPSHOT`); the source is never written to.
- `--create-schema` builds the target tables with `SHOW CREATE TABLE`, so no external `mysqldump` is needed.
- Loads with batched multi-row `INSERT`s (up to 250 rows or 8 MiB per batch).
- Resets `AUTO_INCREMENT` counters after loading, skips generated (`VIRTUAL`/`STORED`) columns, and handles circular and self-referential foreign keys by disabling `FOREIGN_KEY_CHECKS` for the loading session only.
- Masks values inside `JSON` columns with JSONPath rules (the `jsonb` transformer).
- MySQL schema DDL commits implicitly, so schema creation is not rolled back if the row load fails. Use a dedicated, empty target database.

**Other changes**

- The database engine is now an explicit setting. See *Upgrading* below.
- The `ip` transformer is chosen automatically for PostgreSQL `inet` and `cidr` columns.
- Failed cloud-mode pulls now record an audit receipt, so failures show up in the audit log.

### Upgrading from 0.1.x

Add `engine: postgres` (or `engine: mysql`) at the top of your existing `subveil.yaml`:

```yaml
version: 1
engine: postgres
```

`init`, `plan` and `pull` ask you to choose when the key is missing, but only on a terminal. In CI, without the key, they stop with exit code 2. The engine is never guessed from the connection string.

## Install

### Homebrew (macOS and Linux)

```sh
brew install subveilhq/tap/subveil
```

Upgrade later with `brew update && brew upgrade subveil`.

### Manual download

1. Open the [Releases](../../releases) page and download the archive for your OS and architecture (macOS, Linux and Windows; amd64 and arm64), together with `checksums.txt`.
2. Verify the checksum:

   ```sh
   shasum -a 256 -c checksums.txt --ignore-missing
   ```

3. Extract it and put `subveil` on your `PATH`:

   ```sh
   # macOS / Linux example
   tar -xzf subveil_<version>_darwin_arm64.tar.gz subveil
   sudo mv subveil /usr/local/bin/
   subveil version
   ```

On Windows, extract the `.zip` and add `subveil.exe` to your `PATH`.

macOS may quarantine an unsigned download. Run this once to allow it:

```sh
xattr -d com.apple.quarantine subveil
```

### CI install script (Linux)

```sh
curl -fsSL https://subveil.dev/install.sh | sh -s -- --version 0.2.0 --pg-client 17
```

`--version` is required. `--pg-client MAJOR` also installs the matching PostgreSQL client tools (Debian/Ubuntu only), which `--create-schema` needs for PostgreSQL.

## Requirements

- A source and a target database, both PostgreSQL 14+ or both MySQL 8.0+. The target should be separate from production.
- PostgreSQL only: `pg_dump` from the PostgreSQL client tools, and only if you use `--create-schema` or `--recreate-schema`. Its major version must be at least your server's. With Homebrew's keg-only `libpq`, run `brew link --force libpq`.
- MySQL needs no extra tools.

## Quick start

Both databases are read from environment variables, never from the config file:

```sh
# PostgreSQL
export SUBVEIL_SOURCE_DSN="postgres://user:pass@source-host/db"
export SUBVEIL_TARGET_DSN="postgres://user:pass@target-host/db"

# or MySQL
export SUBVEIL_SOURCE_DSN="mysql://user:pass@source-host:3306/db"
export SUBVEIL_TARGET_DSN="mysql://user:pass@target-host:3306/db"

export SUBVEIL_SEED="a-long-random-secret"   # keys the masking; same seed gives the same fakes

subveil init                                  # introspect the source, write subveil.yaml
subveil plan                                  # preview load order and masking, writes nothing
subveil pull --create-schema                  # copy a masked, referentially intact slice
```

Use a **read-only** account for the source. Keep the seed secret; if you change it, every masked value changes.

## Commands

| Command | What it does | Use it when |
|---|---|---|
| `subveil init` | Introspects the source, finds foreign keys, flags likely PII by column name and by sampled values, and writes `subveil.yaml`. | Starting a new project, or regenerating a config. |
| `subveil plan` | Builds the foreign-key graph, shows the load order, circular references and masked columns. Writes nothing. | Reviewing a config before copying data. |
| `subveil pull` | Reads a consistent snapshot, selects the subset, masks values in flight and loads the target. | Building or refreshing a staging, CI or local database. |
| `subveil login` / `logout` | Connects the CLI to your Subveil Cloud account (or removes the saved token). | Using team policies, drift alerts and audit receipts. |
| `subveil branch report` | Reports the state of a pull request's masked database to the dashboard. | Per-PR preview databases in CI. |
| `subveil ci report` | Maps the final GitHub Actions result to a branch state and reports it. | The last step of a PR-database workflow. |
| `subveil version` | Prints the installed version. | Checking you are on the release you expect. |

### `subveil init`

```sh
subveil init                                  # uses $SUBVEIL_SOURCE_DSN, writes ./subveil.yaml
subveil init --schemas public,billing         # limit to some PostgreSQL schemas or MySQL databases
subveil init --sample-rows 200                # look at more values per column for hidden PII
subveil init --sample-rows 0                  # name-based detection only
subveil init --project prj_xxx --report-drift # also send schema metadata to the cloud (no data)
subveil init --force                          # overwrite an existing file
```

`--force` replaces the whole file, including your hand-written subset settings. Keep the config in version control and review the diff.

| Flag | Meaning |
|---|---|
| `--dsn URL` | Source database (default: `$SUBVEIL_SOURCE_DSN`) |
| `--schemas a,b` | PostgreSQL schemas or MySQL databases (default: non-system schemas / current database) |
| `--out FILE` | Output file (default `subveil.yaml`) |
| `--force` | Overwrite an existing file |
| `--sample-rows N` | Values sampled per text/JSON column to find PII the name does not show (default 50, 0 = off) |
| `--project ID` | Cloud project id to write into the config |
| `--report-drift` | Send schema metadata to the cloud and print unmasked-PII findings |
| `--commit-sha SHA` | Git commit to attach to the schema report |

### `subveil plan`

```sh
subveil plan                                  # reads ./subveil.yaml
subveil plan --config staging.yaml
```

Use it before every first pull and after editing rules. It tells you the load order, which foreign keys form cycles, and what will be masked.

### `subveil pull`

```sh
subveil pull --create-schema                  # first copy into an empty target
subveil pull --dry-run                        # compute the subset and report sizes; writes nothing
subveil pull --truncate                       # refresh data in a target that already has the schema
subveil pull --recreate-schema                # source schema changed (a migration): rebuild the target
subveil pull --recreate-schema --yes          # same, for CI (skips the type-the-name prompt)
subveil pull --jobs 8                         # more parallel source readers (PostgreSQL)
subveil pull --require-audit                  # cloud mode: stop unless policy and audit receipt both succeed
```

| Flag | Meaning |
|---|---|
| `--config FILE` | Config file (default `subveil.yaml`) |
| `--source URL`, `--target URL` | Override the DSNs from the environment |
| `--schemas a,b` | PostgreSQL schemas, or the MySQL database to copy |
| `--dry-run` | Compute the subset and report sizes; write nothing |
| `--truncate` | Empty the target tables before loading |
| `--create-schema` | Create the target schema (PostgreSQL uses `pg_dump`; MySQL uses native DDL). The target must have no tables yet. |
| `--recreate-schema` | Drop and rebuild the target schema from the source, then load. Implies `--create-schema`. Asks you to type the target database name, or pass `--yes`. |
| `--yes` | Confirm `--recreate-schema` without the prompt (required in CI) |
| `--jobs N`, `-j N` | Parallel source readers (default 4; 1 streams sequentially). On PostgreSQL, above 1 needs a primary server. |
| `--parallel-writes` | Load independent tables at the same time. **Not atomic**: a failure empties the target. Needs `--jobs` above 1 and one of `--create-schema`, `--recreate-schema` or `--truncate`. |
| `--source-timeout 30m` | Longest any single source query may run (0 removes the limit) |
| `--cloud` / `--no-cloud` | Require, or disable, cloud policy sync, drift report and audit receipt |
| `--project ID` | Cloud project id (overrides `project_id` in the config) |
| `--require-audit` | Fail unless the policy fetch and the audit receipt both succeed |
| `--commit-sha SHA` | Git commit to attach to the schema report |

By default the whole load runs in one transaction on the target. The target must match the source schema; otherwise `pull` stops and lists the missing tables.

### `subveil login` and `subveil logout`

```sh
export SUBVEIL_API_URL=https://subveil.dev     # see the note below
subveil login                                  # opens the browser to approve this machine
subveil login --no-browser                     # print the URL instead (SSH, containers)
subveil login --api-key "$KEY"                 # store an API key directly (CI)
subveil logout                                 # delete the saved credentials
```

Cloud mode starts automatically when `project_id` is set in `subveil.yaml` and credentials exist. In CI, set `SUBVEIL_API_KEY` (a project CI token) instead of logging in.

> **Note:** in 0.2.0 the CLI's built-in default API address does not resolve. Set `SUBVEIL_API_URL=https://subveil.dev` (or pass `--api-url https://subveil.dev`) until a later release changes the default.

### `subveil branch report` and `subveil ci report`

For per-pull-request databases in GitHub Actions. On Actions, the repository, pull request, branch, commit and run link are detected automatically.

```sh
subveil branch report --status ready           # tell the dashboard the database is ready
subveil branch report --status failed --message "tests failed" --warn-only
subveil ci report --status "${{ job.status }}" # final state: finished, failed or closed
```

`--warn-only` prints a warning instead of failing the job if the report cannot be sent. Statuses for `branch report` are `provisioning`, `ready`, `finished`, `failed` and `closed`. The Branches page in the dashboard generates a complete workflow file for you.

## Configuration

`subveil.yaml` is the only source of truth. Secrets stay in environment variables.

```yaml
version: 1
engine: postgres            # or: mysql

source:
  dsn_env: SUBVEIL_SOURCE_DSN
target:
  dsn_env: SUBVEIL_TARGET_DSN
masking:
  seed_env: SUBVEIL_SEED

subset:
  roots:
    - table: public.accounts
      where: "status = 'active'"        # exactly one of where / scale per root
    - table: public.patients
      scale: 0.05                       # a 5% sample
  default_scale: 10%                    # other tables not reached from a root
  full_tables: [public.countries]       # small reference tables copied whole
  exclude_tables: [public.audit_log]    # schema only, no rows

tables:
  public.customers:
    columns:
      email:     { transformer: email }
      full_name: { transformer: name }
      last_ip:   { transformer: ip }
      profile:
        transformer: jsonb              # mask keys inside JSON/JSONB
        paths:
          - { path: "$.contact.email", transformer: email }
```

Parents of selected rows are pulled in to keep foreign keys valid, and children are pulled in through their foreign keys. Circular and self-referential keys are resolved automatically.

## Transformers

| Transformer | Use it for |
|---|---|
| `email` | Email addresses (optionally with a fixed `domain`) |
| `name` | Person names |
| `phone` | Phone numbers |
| `address` | Street addresses |
| `uuid` | UUIDs and opaque ids |
| `token` | Secrets, API keys, tokens |
| `ip` | IPv4/IPv6 addresses, including PostgreSQL `inet`/`cidr` |
| `number` | Numeric values, keeping scale |
| `date_shift` | Dates, shifted consistently (birth dates, event dates) |
| `redact` | A fixed replacement value (for example a password hash) |
| `null` | Remove the value (quote it in YAML: `"null"`) |
| `custom_sql` | A SQL expression run on the source during the copy |
| `jsonb` | Mask chosen paths inside JSON/JSONB columns with other transformers |

Masking is deterministic: the same input and seed give the same output in every table and run, so joins still line up. Masked values in `UNIQUE` columns never collide.

## Environment variables

| Variable | Purpose |
|---|---|
| `SUBVEIL_SOURCE_DSN` | Source database (names configurable in the YAML) |
| `SUBVEIL_TARGET_DSN` | Target database |
| `SUBVEIL_SEED` | Secret key for deterministic masking |
| `SUBVEIL_API_KEY` | Cloud token for CI (overrides the saved credentials) |
| `SUBVEIL_API_URL` | Cloud address (see the login note above) |

## Common workflows

**First copy:** `init`, review `subveil.yaml`, `plan`, then `pull --create-schema`.

**Refresh staging nightly:** `subveil pull --truncate` on a schedule. Same seed, same fakes.

**After a migration:** `subveil pull --recreate-schema` (add `--yes` in CI).

**Check before you copy:** `subveil pull --dry-run` shows the subset size without writing.

**Per-PR databases and audit receipts:** log in or set `SUBVEIL_API_KEY`, set `project_id`, and use `--require-audit`. See https://subveil.dev/docs.

## Support and feedback

- Questions and bug reports: https://subveil.dev/feedback
- Documentation: https://subveil.dev/docs (PostgreSQL: https://subveil.dev/postgresql, MySQL: https://subveil.dev/mysql)
- Release notes: see each entry on the [Releases](../../releases) page.
