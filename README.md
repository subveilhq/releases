# Subveil releases

Prebuilt binaries for the [Subveil](https://subveil.dev) CLI.

Subveil creates safe, smaller copies of a PostgreSQL database. It subsets data while keeping foreign keys valid, and it anonymizes PII deterministically. It runs inside your network or CI, so raw data never leaves your infrastructure.

This repository only hosts release binaries and release notes. Documentation is at **https://subveil.dev/docs**.

## Install

### Homebrew (macOS and Linux)

```sh
brew install subveilhq/tap/subveil
```

Upgrade later with:

```sh
brew upgrade subveil
```

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

## Requirements

- PostgreSQL source and target databases.
- `pg_dump` from the PostgreSQL client tools, only if you use `--create-schema`. With Homebrew's keg-only `libpq`, run `brew link --force libpq`.

## Quick start

```sh
export SUBVEIL_SOURCE_DSN="postgres://user:pass@source-host/db"
export SUBVEIL_TARGET_DSN="postgres://user:pass@target-host/db"

subveil init        # generate a starter subveil.yaml
subveil plan        # preview the dependency plan
subveil pull --config subveil.yaml
```

See the [documentation](https://subveil.dev/docs) for configuration, transformers, subsetting and circular foreign keys.

## Support and feedback

- Questions and bug reports: https://subveil.dev/feedback
- Release notes: see each entry on the [Releases](../../releases) page.
