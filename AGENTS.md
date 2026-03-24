# find-snapshot

Single-file bash tool. The executable is `find-snapshot` at the repo root.

## Architecture

* Pure bash, no external dependencies beyond coreutils + xz-utils
* Configuration via `FIND_SNAPSHOT_*` environment variables (no config files)
* Cache files: `YYMMDD-{files,dirs}.find.xz` in `$FIND_SNAPSHOT_CACHE_DIR`
* Parallel cache building for files and dirs indexes

## Development

* Keep it a single self-contained bash script
* Test with: `FIND_SNAPSHOT_PATHS="/tmp" ./find-snapshot cat | head -5`
* SIGPIPE handling is intentional (`set +eo pipefail` before output commands)

## Best practices

* @../ai_docs/small_descriptive_commits.md
* @../ai_docs/meta_commit_motivation.md
