# recall-doctor

`fsck` for a [recall](https://github.com/j0yen/recall) memory store: report where the Markdown files and the SQLite index disagree, and exit non-zero when they do.

A recall store has two halves that are supposed to mirror each other — the `.md` memory files on disk and the rows in the SQLite index. They drift. A file gets deleted by hand and its index row lingers; a file is added and never indexed; an embedder change leaves a mix of vector ids. None of this errors at write time, so the divergence sits there silently until a query returns something stale or nothing at all. `recall-doctor` reads both halves, compares them, and tells you exactly where they parted ways.

It is read-only by default. It walks the store, prints a report, and sets its exit code so a script or a review step can gate on it.

## Install

### One-liner

```sh
curl -fsSL https://raw.githubusercontent.com/j0yen/recall-doctor/main/install.sh | bash
```

### Manual

```sh
git clone --depth 1 https://github.com/j0yen/recall-doctor.git
cd recall-doctor
./install.sh
```

Builds and installs the `recall-doctor` binary with `cargo install --path . --locked` (lands in `~/.cargo/bin/`). Requires `cargo` / `rustc 1.85+` and `git`. The index check shells out to `sqlite3`; without it, `indexed_count` reports `null` with a warning.

## Usage

```sh
# Check the default store (~/.claude/recall):
recall-doctor

# Point at another store, get JSON:
recall-doctor --root /path/to/store --format json

# Reindex if anything diverged (shells out to `recall reindex`):
recall-doctor --fix
```

The report has these fields:

| Field | Meaning |
|---|---|
| `file_count` | `.md` files under `<root>/memories/` |
| `indexed_count` | rows in the index's `memories_meta` table (`null` if the DB or `sqlite3` is missing) |
| `orphans` | ids on disk but absent from the index |
| `missing` | index rows whose file no longer exists |
| `embedder_ids` | distinct embedding ids present — a mix means a half-finished reindex |
| `schema_version` | the index schema version |
| `warnings` | anything that couldn't be checked |

JSON output has deterministic key ordering: the same store produces byte-identical output, so it's safe to diff.

Exit codes: `0` in sync (no orphans, no missing, `indexed_count == file_count`); `1` divergent; `2` invocation error (a `--root` that exists but isn't a directory, bad arguments). `--fix` runs `recall reindex` only when there's something to fix, and is the one mode that writes.

## Where it fits

Part of the recall family:

- **[recall](https://github.com/j0yen/recall)** — the agentic-memory store itself. It ships a built-in `recall doctor`; this standalone binary predates it, runs against any store, and is convenient as a `/self-review` gate or a CI step.
- **[recall-io](https://github.com/j0yen/recall-io)** — backup and migration for the same store (NDJSON export/import).

## Status

Each acceptance criterion has a matching integration test under `tests/acceptance_ac<n>.rs`. Built via the [`autobuilder`](https://github.com/j0yen/autobuilder) pipeline (PRD intake → intent-card → scaffold → iterate-and-prove). Originally a subdirectory of the [`wintermute`](https://github.com/j0yen/wintermute) monorepo; this repo is the standalone distribution.

## License

Dual-licensed under MIT OR Apache-2.0; pick whichever fits. See [LICENSE-MIT](LICENSE-MIT) and [LICENSE-APACHE](LICENSE-APACHE).
