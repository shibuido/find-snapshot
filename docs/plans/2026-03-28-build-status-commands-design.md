# Design: `build` and `status` commands

Date: 2026-03-28

## Motivation

Currently `find-snapshot` always auto-builds the cache on first use of
cat/grep/sk/fzf. Two gaps:

1. No way to pre-build the cache without running a query (useful for
   cron jobs, startup scripts, or after changing `FIND_SNAPSHOT_PATHS`)
2. No way to inspect what cache files exist, their sizes, or the
   effective configuration

## `build` command

**Usage:**

```
find-snapshot build           # build today's cache if missing
find-snapshot build --force   # delete + rebuild today's cache
```

**Behavior:**

1. Runs `cleanup_old` (same as existing commands)
2. If `--force`: deletes today's `*-files.find.xz` and `*-dirs.find.xz`
3. Calls existing `ensure_cache` to build both files+dirs in parallel
4. Prints status messages to stderr via `info()`

**Design decisions:**

* Always builds both files and dirs — simpler UX, mirrors `ensure_cache`
* `--force` is a command-specific flag, not a global option
* Reuses existing `ensure_cache` and `cleanup_old` functions unchanged

## `status` command

**Usage:**

```
find-snapshot status
```

**Behavior:**

1. No cleanup (read-only command)
2. No auto-build (does not call `ensure_cache`)
3. Prints three sections to stdout:

```
=== Configuration ===
  Paths:     /home/user:/srv/data
  Cache dir: /tmp/find-snapshot-1000
  Max age:   2 days

=== Today's cache (260328) ===
  260328-files.find.xz   12.4M  2026-03-28 09:15
  260328-dirs.find.xz     1.8M  2026-03-28 09:15

=== Older cache files ===
  260327-files.find.xz   12.3M  2026-03-27 08:30
  260327-dirs.find.xz     1.7M  2026-03-27 08:30
```

**Edge cases:**

* No cache files at all: shows config + "No cache files found. Run
  'find-snapshot build' to create one."
* Today's cache missing, older exist: "Today's cache" section says
  "(not yet built)"

**Design decisions:**

* Human-friendly formatted output (not machine-parseable)
* No entry counts — keeps status fast (no decompression needed)
* Shows ALL cache files including those past max age (status is
  informational, not a mutation)

## Changes to existing code

**`usage()` function:**

* Add `build` and `status` to Commands section with descriptions
* Add examples for both commands

**Argument parsing / main flow:**

* `build` and `status` handled as new `case` branches
* `build`: runs `cleanup_old`, handles `--force`, calls `ensure_cache`
* `status`: skips cleanup and ensure_cache, reads filesystem directly

**No changes to:**

* Existing `ensure_cache` logic
* Environment variable interface
* Cache file naming convention
* Behavior of cat/grep/sk/fzf commands
