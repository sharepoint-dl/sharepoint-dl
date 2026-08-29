# Developer Guide — CLI Sync

## Design in one minute

The script reads the public HTML page for a shared folder, extracts the temporary drive endpoint and access token Microsoft exposes for that share, then walks the folder through the corresponding API. It uses only the Python standard library.

```text
Public folder link → shared-folder metadata → recursive file listing
                                            → compare against recorded version markers
                                            → resolve conflicts (CONFLICT_MODE)
                                            → download changed files → save state
```

## Key pieces

| Area                          | Responsibility                                                                             |
| ----------------------------- | ------------------------------------------------------------------------------------------ |
| `SharePointClient`            | Requests, cookies, retry/backoff, folder metadata, and downloads.                          |
| `sync()`                      | Filtering, change detection, conflict resolution, parallel downloads, deletion, and state. |
| `safe_target()`               | Ensures remote paths cannot write outside the selected destination.                        |
| `is_current()`/`has_divergence()` | Decide whether a local file matches the recorded cloud version, or conflicts with it.  |
| `.sharepoint-sync-state.json` | Stores each file's remote eTag and last-modified timestamp for incremental syncs.          |

## Local development

The project has no install step. Create a `.env` from `.env.example`, then safely inspect a real public share:

```bash
cp .env.example .env
python3 sharepoint_public_sync.py --dry-run
```

Use a dedicated test destination. A normal run with mirroring enabled can delete local files that are absent from the remote share.

## Implementation notes

- Downloads use `ThreadPoolExecutor`; `--threads` controls concurrency.
- HTTP `429`, `500`, `502`, `503`, and `504` responses are retried with exponential backoff.
- Files are written to a temporary sibling file and atomically moved into place after a successful download.
- Path containment is checked before every write to protect against directory traversal.
- Conflict resolution is driven by `CONFLICT_MODE` (`.env`): `latest` compares local mtime with the remote's `lastModifiedDateTime` (keeping local when the cloud date is unknown), `cloud` always overwrites, and `keep_both` renames the local file to `<name>-modified_locally.<ext>` (deduplicated with a counter) before the cloud copy is downloaded. Renamed copies are excluded from `MIRROR_DELETE` cleanup.
- `destination` is resolved once at the top of `sync()` so `safe_target()`'s symlink-resolved paths stay consistent with every `relative_to()` comparison.

## Extending the tool

- Update public-share parsing in `public_folder_metadata()` if Microsoft changes the shared-folder page.
- Adjust traversal in `list_files()` for new metadata or pagination behavior.
- Add filters around the `exclude_patterns` handling in `sync()`.
- Adjust `CONFLICT_MODE` behavior around `has_divergence()` in `sync()`.
- Keep new write paths behind `safe_target()` and preserve the temporary-file download behavior.

## License

This project is licensed under the [MIT License](LICENSE).
