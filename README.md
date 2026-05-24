# recall-doctor

> PRD-agentic-memory §9.4 surfaces operational gap: /self-review hand-rolls index-vs-file divergence checks because there's no `recall doctor`.

## Why

PRD-agentic-memory §9.4 surfaces operational gap: /self-review hand-rolls index-vs-file divergence checks because there's no `recall doctor`. The full v0.2 rebuild is multi-session work; this slice ships `recall-doctor` as a standalone companion binary that reads the v0.1 recall data dir, reports {file_count, indexed_count, orphans, missing, embedder_ids, schema_version}, and exits non-zero on divergence. When v0.2 properly rebuilds recall, `doctor` absorbs into the main binary; today it's a useful side-tool that closes the /self-review gap immediately.

## Build

```sh
cargo build --release
```

Produces `target/release/recall-doctor`. Symlink into `~/.local/bin/` if you want it on `$PATH`.

## Usage

```sh
recall-doctor --help
```

## Audience

the author and /self-review Phase A reading divergence reports between the on-disk Markdown memory store (~/.claude/recall/memories/) and the SQLite index (~/.claude/recall/index/recall.sqlite). Output: JSON consumed by /self-review (and shell-pipeline), text for human terminal use.

## Acceptance criteria

This project was scaffolded from a PRD via the `autobuilder` pipeline. The MUST-level acceptance criteria are:

- **AC1**: CLI binary `recall-doctor` accepts `[--root <dir>]` (default `~/.claude/recall`). Reads the directory, prints a divergence report to stdout, exits 0 when in sync. Empty/nonexistent root → exit 0 with empty report.
- **AC2**: Walks `<root>/memories/` recursively, counting *.md files; ignores non-Markdown and dot-prefixed directories. Reports `file_count: N`.
- **AC3**: Queries `<root>/index/recall.sqlite` `memories_meta` table for row count; reports `indexed_count: N`. Missing DB → `indexed_count: null` and warning. Shells out to `sqlite3` binary; absence of `sqlite3` → reports `indexed_count: null` + ...
- **AC4**: Computes orphans: file IDs present on disk (parsed from frontmatter `id:` field) but not in `memories_meta`. Reports `orphans: [<id>...]` sorted lexicographically.
- **AC5**: Computes missing: `memories_meta` rows whose `path` column points to a non-existent file. Reports `missing: [<id>...]` sorted lexicographically.
- **AC6**: Reports `embedder_ids: [<id>...]` — distinct `embedding_id` values from `memories_meta` (excluding NULL). Empty list when no rows or no embeddings.
- **AC7**: Output format selectable via `--format text|json` (default text). JSON shape: `{file_count, indexed_count, orphans, missing, embedder_ids, schema_version, warnings}`. Deterministic key ordering; same input → byte-identical output.
- **AC8**: Exit code 0 when in sync (orphans+missing empty AND indexed_count == file_count). Exit code 1 when divergent. Exit code 2 on invocation error (bad --root that exists but isn't a dir, malformed args). Stable across runs.
- **AC9**: `--fix` invokes `recall reindex` (shelling out via PATH) when orphans OR missing is non-empty. Default (no --fix) is read-only — never writes. When `recall` isn't on PATH, `--fix` emits a warning and exits with code 1.

Each AC has a matching integration test under `tests/acceptance_ac<n>.rs`.

## Provenance

Built via the [`autobuilder`](https://github.com/j0yen/autobuilder) pipeline (PRD intake -> intent-card -> scaffold -> iterate-and-prove). Originally consolidated as a subdir of the [`wintermute`](https://github.com/j0yen/wintermute) monorepo; this standalone repo is a fresh-init snapshot for easier consumption and distribution.

## License

Licensed under either of:

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE))
- MIT license ([LICENSE-MIT](LICENSE-MIT))

at your option.
