# find-snapshot

> Fast cached find searches over xz-compressed file/directory snapshots.

Avoid running expensive `find` commands repeatedly on large filesystems. `find-snapshot` creates daily xz-compressed snapshots of file and directory listings, then lets you search them instantly with `grep`, `cat`, or fuzzy finder (`sk`).

## Features

* Daily auto-caching with parallel xz compression (`xz -T0`)
* ~95% compression ratio on typical file listings
* Automatic cleanup of old snapshots
* Separate file and directory indexes
* Configurable search paths, cache location, and retention via environment variables
* Pipe-friendly output (handles SIGPIPE gracefully)

## Quick start

```bash
# Clone and make executable
git clone https://github.com/shibuido/find-snapshot.git
chmod +x find-snapshot/find-snapshot

# Symlink into PATH
ln -s "$(pwd)/find-snapshot/find-snapshot" ~/.local/bin/find-snapshot

# Set directories to index
export FIND_SNAPSHOT_PATHS="/home/user:/srv/data"

# Search
find-snapshot grep -i "\.pdf$"              # find all PDFs
find-snapshot cat | head -20                # list first 20 cached files
find-snapshot sk -f "report" | head -10     # fuzzy match files (skim)
find-snapshot fzf -f "report" | head -10    # fuzzy match files (fzf)
find-snapshot -d sk -f "projects"           # fuzzy match directories
```

## Configuration

All settings are optional environment variables:

| Variable | Default | Description |
|---|---|---|
| `FIND_SNAPSHOT_PATHS` | `$HOME` | Colon-separated directories to index |
| `FIND_SNAPSHOT_CACHE_DIR` | `/tmp/find-snapshot-$UID` | Where to store compressed snapshots (per-user) |
| `FIND_SNAPSHOT_MAX_AGE` | `2` | Days before old snapshots are cleaned up |

## Dependencies

* `bash` (4.0+)
* `xz` (xzcat, xzgrep)
* `find` (coreutils)
* `sk` (skim fuzzy finder) - optional, only needed for the `sk` command
* `fzf` (fuzzy finder) - optional, only needed for the `fzf` command

## License

MIT
