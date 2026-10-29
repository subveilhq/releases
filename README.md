# Subveil releases

Prebuilt binaries for the [Subveil](https://subveil.dev) CLI.

Subveil makes smaller, anonymized copies of PostgreSQL and MySQL databases. It runs inside your network or CI, so raw data never leaves your infrastructure.

- **Documentation:** https://subveil.dev/docs
- **What changed in each version:** [versions/](versions/)

This repository only hosts release binaries and notes. The source is private.

## Install

### Homebrew (macOS and Linux)

```sh
brew install subveilhq/tap/subveil
```

Upgrade later with `brew update && brew upgrade subveil`.

### Manual download

1. Download the archive for your OS and architecture, and `checksums.txt`, from the [Releases](../../releases) page.
2. Verify it:

   ```sh
   shasum -a 256 -c checksums.txt --ignore-missing
   ```

3. Extract it and put `subveil` on your `PATH`, then check it:

   ```sh
   tar -xzf subveil_<version>_darwin_arm64.tar.gz subveil
   sudo mv subveil /usr/local/bin/
   subveil version
   ```

On Windows, extract the `.zip` and add `subveil.exe` to your `PATH`. On macOS, run `xattr -d com.apple.quarantine subveil` once if the download is blocked.

### CI (Linux)

```sh
curl -fsSL https://subveil.dev/install.sh | sh -s -- --version <version>
```

Add `--pg-client <major>` on Debian or Ubuntu to also install the matching PostgreSQL client tools, which `--create-schema` needs for PostgreSQL.

## Supported databases

PostgreSQL 14+, MySQL 8.0+ and MariaDB 10.6+ (with `engine: mysql`). The versions each release was tested on are listed in its notes under [versions/](versions/).

## Help

Questions and bug reports: https://subveil.dev/feedback
