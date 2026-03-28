# Design: Python Rewrite with Subdirectory-Aware Snapshots

Date: 2026-03-28

## Motivation

The current bash implementation indexes a fixed set of directories
(FIND_SNAPSHOT_PATHS) as a single monolithic snapshot. This works for
broad searches but falls short when users want to:

* Search only within a specific subdirectory without building a full index
* Have separate snapshots for different subtrees at different freshness
* Reuse a parent snapshot (e.g., /foo/bar) when searching a child
  (/foo/bar/zoo/woo)
* Import the tool as a Python library for programmatic use

The rewrite to Python enables subdirectory-aware snapshot resolution,
a proper importable API, and cleaner architecture.

## File Structure

```
find_snapshot.py       # single file, chmod+x shebang, importable
find-snapshot          # symlink → find_snapshot.py (backward compat)
```

`find_snapshot.py` uses `#!/usr/bin/env python3` shebang (no extra
packages needed — stdlib + subprocess xz only). If dependencies are
ever added, switch to uv run shebang with PEP 723 inline metadata.

Entry point: `if __name__ == '__main__': main()`

## Cache Structure

```
/tmp/find-snapshot-{UID}/
├── index.jsonl                          # path→hash registry
├── 260328-a1b2c3d4-files.find.xz       # /home/user (today)
├── 260328-a1b2c3d4-dirs.find.xz
├── 260327-a1b2c3d4-files.find.xz       # /home/user (yesterday)
├── 260327-a1b2c3d4-dirs.find.xz
├── 260328-e5f6a7b8-files.find.xz       # /foo/bar/zoo/woo (today)
└── 260328-e5f6a7b8-dirs.find.xz
```

### Filename convention

`{YYMMDD}-{path_hash}-{type}.find.xz`

Where:

* `YYMMDD` — snapshot date
* `path_hash` — first 8 hex chars of `sha256(os.path.realpath(path))`
* `type` — `files` or `dirs`

### index.jsonl format

One JSON object per line. Each line represents a unique (path, date) pair:

```jsonl
{"id":"a1b2c3d4","path":"/home/user","date":"260328","created":"2026-03-28T09:15:00"}
{"id":"e5f6a7b8","path":"/foo/bar/zoo/woo","date":"260328","created":"2026-03-28T14:30:00"}
```

Fields:

* `id` — `sha256(os.path.realpath(path)).hexdigest()[:8]`
* `path` — canonical absolute path (via `os.path.realpath`)
* `date` — snapshot date as YYMMDD string
* `created` — ISO 8601 timestamp of when the snapshot was built

The `id` is deterministic from the path, so the same path always
produces the same id. Multiple dates for the same path produce
multiple index entries with the same id but different dates.

### Index integrity

* Writes are atomic: write to tempfile, then `os.rename()` over the
  index file
* On read: validate that referenced snapshot files exist on disk;
  silently prune stale entries
* If index is lost/corrupted: `clean` + rebuild. It's a cache.

## Path Resolution Algorithm

When the user requests a search within path P:

1. Read index.jsonl
2. Find all entries where `entry.path` is a prefix of P (i.e.,
   P starts with `entry.path + "/"` or P == `entry.path`)
3. Sort matches by:
   * **Freshness first** — newest `date` wins
   * **Specificity second** — longest `path` breaks ties
4. Select the top match
5. If match.path == P: use snapshot directly
6. If match.path is a parent of P: decompress and filter lines by
   prefix (only output lines starting with P)
7. If no match: build a new snapshot for P

### Design decision: freshness over specificity

A newer broader snapshot (today's /foo/bar, filtered) is preferred
over an older exact match (yesterday's /foo/bar/zoo/woo). Rationale:
the user wants the most current view of the filesystem. The filtering
cost is acceptable.

## CLI Interface

### Smart path detection

The first non-option argument is treated as a path if it is NOT a
known command name. Known commands: `cat`, `grep`, `sk`, `fzf`,
`build`, `status`, `clean`.

Relative paths are resolved via `os.path.realpath()`:

* `.` → current directory
* `..` → parent directory
* `../../data` → resolved relative path

The explicit `--path` / `-p` flag is also supported as an escape
hatch (e.g., if a directory is literally named `cat`).

### Full CLI

```bash
# Search commands (auto-build if no matching snapshot)
find-snapshot . grep "pattern"              # current dir
find-snapshot /foo/bar sk -f "needle"       # explicit path
find-snapshot --path /foo/bar grep "x"      # explicit flag
find-snapshot grep "pattern"                # default: FIND_SNAPSHOT_PATHS

# Build commands
find-snapshot build                         # build for default paths
find-snapshot /foo/bar build                # build for specific subtree
find-snapshot /foo/bar build --force        # force rebuild

# Inspection
find-snapshot status                        # config + all snapshots

# Cleanup
find-snapshot clean                         # wipe all cache + index
find-snapshot clean --dry-run               # preview what would be deleted
find-snapshot clean --older 3               # delete snapshots > 3 days old
find-snapshot clean --path /foo             # delete snapshots for this path

# Options (global, before path/command)
--dirs, -d                                  # search dirs instead of files
--help, -h                                  # show help
--path PATH, -p PATH                        # explicit path (escape hatch)
```

### Default behavior (backward compatible)

When no `--path` and no path-as-first-arg:

* Uses `FIND_SNAPSHOT_PATHS` environment variable (colon-separated)
* Falls back to `$HOME` if unset
* Same behavior as current bash version

## Commands Detail

### `build`

1. Run auto-cleanup
2. If `--force`: delete existing snapshot files for the target path + date
3. Run `find {path} -type f | xz -T0 > {snapshot}` (parallel for
   files + dirs)
4. Update index.jsonl with new entry

### `status`

Read-only. No cleanup. No auto-build. Output:

```
=== Configuration ===
  Paths:     /home/user:/srv/data
  Cache dir: /tmp/find-snapshot-1000
  Max age:   2 days

=== Today's cache (260328) ===
  /home/user            260328-a1b2c3d4-files.find.xz   12.4M  09:15
  /home/user            260328-a1b2c3d4-dirs.find.xz     1.8M  09:15
  /foo/bar/zoo/woo      260328-e5f6a7b8-files.find.xz    0.3M  14:30

=== Older cache files ===
  /home/user            260327-a1b2c3d4-files.find.xz   12.3M  2026-03-27
  ...
```

### `clean`

* No args: delete ALL cache files + index (fresh start)
* `--dry-run`: print what would be deleted, exit
* `--older N`: delete snapshots older than N days
* `--path P`: delete only snapshots whose indexed path matches P

### `cat`, `grep`, `sk`, `fzf`

Same as current bash version, but with path resolution:

1. Resolve target path (smart detection or `--path` or default)
2. Find best matching snapshot (resolution algorithm)
3. If no match: auto-build snapshot for target path
4. Run the command (with prefix filtering if using parent snapshot)

## Compression

Subprocess `xz` for all compression and decompression:

* Build: `find ... | xz -T0 > snapshot.find.xz`
* Read: `xzcat snapshot.find.xz` (piped to grep/sk/fzf/stdout)
* Filter: `xzcat snapshot.find.xz | grep '^/prefix/'`

`xz` is a required system dependency (same as current bash version).

## Auto-Cleanup

Runs automatically on `build`, `cat`, `grep`, `sk`, `fzf` (not on
`status` or `clean`):

1. Read index.jsonl
2. For each entry: if snapshot date is older than MAX_AGE, delete
   files and remove entry
3. Scan cache dir for orphaned files (not in index) and delete them
4. Write cleaned index atomically

This handles cleanup of snapshots from ALL paths, including small
subdirectory snapshots that might otherwise accumulate.

## Python API

```python
from find_snapshot import SnapshotIndex, SnapshotSearch

# Programmatic usage
idx = SnapshotIndex()                    # uses env var config
idx = SnapshotIndex(cache_dir="/tmp/x")  # explicit config

idx.build("/home/user")
idx.build("/foo/bar", force=True)

info = idx.status()  # list of SnapshotInfo objects

search = SnapshotSearch(idx)
for line in search.cat(path="/home/user"):
    print(line)
for line in search.grep(r"\.pdf$", path="/home/user/docs"):
    print(line)

removed = idx.clean(dry_run=True)  # preview
removed = idx.clean(older=3)       # cleanup old
```

Classes use native Python docstrings. Public API is clean and
composable for use in scripts, notebooks, and other tools.

## Environment Variables (backward compatible)

* `FIND_SNAPSHOT_PATHS` — colon-separated default paths (default: $HOME)
* `FIND_SNAPSHOT_CACHE_DIR` — cache directory (default: /tmp/find-snapshot-{UID})
* `FIND_SNAPSHOT_MAX_AGE` — days before auto-cleanup (default: 2)

## Migration from Bash Version

The Python version is a drop-in replacement:

* Same environment variables
* Same command names and flags
* Same cache file format (.find.xz)
* `find-snapshot` symlink preserves the CLI name
* New features (path resolution, clean, Python API) are additive
