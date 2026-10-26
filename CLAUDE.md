# CLAUDE.md

## Project

`metadata.py` is a single-file Python CLI that reports metadata for video and image
files (screen / txt / csv / xlsx). User docs are in `README.md`.

`metadata.py` replaces the earlier `meta.py`, `metas.py`, `metas2.py` and `metax.py`. They are
not in the repository (they may exist untracked in a local checkout; ignore them). Do not
reintroduce their known bugs (wrong CSV column counts, no-op `-N` flags, import-time exit on
missing openpyxl).

## Environment

- Windows is the primary platform (PowerShell/Git Bash); keep it working on macOS/Linux.
- Python 3.8+ compatible: no `match`, no `X | Y` type unions, no `str.removeprefix`.
- Runtime dependencies are optional and imported lazily: `ffprobe` (executable, video),
  `Pillow` (images), `openpyxl` (xlsx). The standard library does everything else.
  Never add a top-level import of Pillow/openpyxl.

## Commands

```
python metadata.py --check          # verify prerequisites
python metadata.py --help
python metadata.py -r               # smoke test on the current folder
```

There is no test suite yet. To test, build a scratch folder (use the session scratchpad,
not the repo) containing: a real mp4 (`ffmpeg -f lavfi -i testsrc=size=320x240:rate=25:duration=2 ...`),
a jpg, a jpeg renamed `.mp4`, a video renamed `.jpg`, an empty `.mp4`, a junk `.png`,
a truncated jpg, a file held open with `FileShare.None`, and a file with a deny-read ACL
(reset it with `icacls <file> /reset` afterwards). Run with and without `-r`, with each
`-o` format, and with `--rename --dry-run`.

## Architecture (all in `metadata.py`)

- `find_files` discovers candidates by extension; `analyze` reads one file and **never
  raises** - every problem ends up in `rec["error"]` (fatal for that file) or `rec["note"]`
  (informational, e.g. mis-labeled).
- File content decides the reader, not the extension: `sniff_is_image` checks magic bytes,
  and `analyze` falls back to the other reader if the first fails. Keep this order logic
  intact when adding formats.
- `probe_video` (ffprobe JSON) and `probe_image` (Pillow) return plain dicts and raise on
  failure; `friendly_error` converts exceptions to user-readable text.
- `run` builds a list of output sets: one for `--kinds together`, or one per kind (with a
  `_video`/`_image` file-name suffix) for `separate`. Writers take a record subset, so
  summaries are per output. `blank_between` (folder grouping + not `-x`) inserts empty
  csv/xlsx rows when the folder changes.
- EXIF: `probe_image` reads date/camera/GPS (`read_gps`); `check_exif` returns "valid" or the
  reason, using the earlier of creation/modification time. `--exif-only` filters on
  `exif_status == "valid"` right after analysis (before rename/sort) and forces `--type image`.
  `mark_doubtful` is a second, per-folder pass (runs before the `-e` filter) that downgrades
  "valid" to "doubtful: ..." for bursts of near-identical dates; status strings starting with
  "doubtful" are never "valid". The validity rules are documented in the README; keep both in sync.
- Records are dicts; `COLUMNS` maps headers to record keys for csv/xlsx; `record_text` /
  `render_text` produce the screen and txt layout.
- `require()` / `prerequisite_status()` / `run_check()` implement the prerequisite checks and
  install hints. Add any new dependency there.
- `normalize_argv` lower-cases switches so they are case-insensitive. Because of this,
  **no two options may differ only by case** (that is why overwrite is `-w`, not `-O`).
  Option values are not lower-cased unless the argument's `type` does it.

## Conventions

- Version lives in `__version__` / `__date__`; bump both, and add a README changelog entry,
  for every user-visible change (semver: patch fixes, minor features, major CLI breaks).
- Messages for the user go to stderr (`warn`); only the report itself goes to stdout.
- Output files are never overwritten silently (`unique_path`) unless `--overwrite`.
- Exit codes: 0 ok, 1 error, 2 usage, 130 interrupted.
- When adding or changing an option, update `build_parser`, the help epilog, and the
  README option table together.
