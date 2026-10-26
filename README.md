# metadata.py

Collect and present metadata for **video and image files** in a folder, optionally
including all subfolders. Output goes to the screen, a text file, a CSV file or an
Excel sheet.

Current version: **1.3.0** (2026-10-26). Check with `metadata.py --version`.

## What it reports

| Kind  | Fields |
|-------|--------|
| Video | container, resolution, duration, video codec, bitrate (kb/s), frame rate, audio codec, size, modified date |
| Image | format, resolution, colour mode, size, created and modified date, EXIF date taken and camera, GPS latitude/longitude/altitude (when present), EXIF check |

### EXIF check and GPS

Images always show GPS position (decimal degrees, south and west negative) when the file
has it; a missing, out-of-range or `0,0` placeholder position is ignored. Every image also
gets an **EXIF check** of its *date taken*, which is `valid` unless it is:

- missing, or blank/zero (`0000:00:00 00:00:00`)
- not understandable as a date
- before 1990 (typical of a reset camera clock)
- in the future
- more than one day after the file's creation or modification date (the earlier of the
  two is used; copying a file changes its creation date, so modification is the safety net)

A midnight time such as `2020:05:01 00:00:00` on a real date is accepted.

**Doubtful dates.** A date that passes those tests is changed from `valid` to
`doubtful: ...` when its folder holds *more than 20* such images whose dates all lie
within *60 seconds* of each other (a stuck or reset camera clock, or dates stamped by a
batch tool). Folders are judged one by one, and `-e` leaves doubtful images out. Tune with
`--doubtful-count N` (default 20, `0` switches the check off) and `--doubtful-seconds S`
(default 60). Only the date taken is compared.

Use `-e` / `--exif-only` to list **only** images that pass. It implies `--type image`
(videos are not probed, which also makes the scan faster), is incompatible with
`--type video`, and leaves out unreadable images. With `--rename`, only the listed
images are renamed.

Every listing ends with a grand summary (file count, video/image split, total size)
and, when grouped by folder, a summary per folder.

## Prerequisites

| Needed for | Requirement |
|------------|-------------|
| Everything | Python 3.8 or newer |
| Video files | `ffprobe` (part of [FFmpeg](https://ffmpeg.org/download.html)) on `PATH` |
| Image files | [Pillow](https://pypi.org/project/pillow/) |
| Excel output | [openpyxl](https://pypi.org/project/openpyxl/) |

Only what your files and chosen output need is required. Run

```
python metadata.py --check
```

to see what is installed and the exact install command for anything missing, for example:

```
winget install Gyan.FFmpeg        # Windows (then open a new terminal)
brew install ffmpeg               # macOS
sudo apt install ffmpeg           # Debian/Ubuntu
python -m pip install pillow openpyxl
```

The script also checks before it starts and prints the same instructions if something
is missing.

## Usage

```
metadata.py [folder] [options]
```

| Option | Long form | Meaning |
|--------|-----------|---------|
| `-r` | `--recursive` | include subfolders |
| `-o FORMAT` | `--output` | `screen` (default), `txt`, `csv`, `xlsx`; comma separated or repeated |
| `-s ORDER` | `--sort` | `name` or `folder` (default: `folder` with `-r`, otherwise `name`) |
| `-t KIND` | `--type` | `video`, `image` or `all` (default) |
| | `--doubtful-count N`, `--doubtful-seconds S` | thresholds for the "doubtful" EXIF check (defaults 20 and 60) |
| `-e` | `--exif-only` | list only images with valid EXIF (see above); implies `--type image` |
| `-k MODE` | `--kinds` | `together` (default): videos and images in one output; `separate`: one output per kind |
| `-x` | `--no-blank-lines` | when ordered by folder, leave out the blank line between folders |
| `-b NAME` | `--basename` | base name of output files (default `metadata`) |
| `-d CHAR` | `--delimiter` | CSV delimiter; use `;` for European Excel |
| `-w` | `--overwrite` | overwrite existing output instead of adding `_01`, `_02`, ... |
| `-n` | `--rename` | append resolution (and duration) to file names |
| `-p` | `--dry-run` | with `--rename`: show what would be renamed |
| `-c` | `--check` | check prerequisites and exit |
| `-v` | `--version` | show version |
| `-h`, `-?` | `--help` | show help |

Switches are **case-insensitive** (`-R` = `-r`, `--Recursive` = `--recursive`), and every
option has a unix-style `--long` form (also `--sort=name`). The folder may be given
before or after the options.

### Examples

```
python metadata.py                              # current folder, to the screen
python metadata.py -r -o xlsx                   # recursive, Excel, ordered by folder then name
python metadata.py D:\Media -R -S name -o txt   # recursive, ordered by file name only
python metadata.py -t image -o csv -d ";"       # images only, CSV for European Excel
python metadata.py -r -o screen,txt,xlsx        # several outputs at once
python metadata.py -r -k separate -o xlsx       # metadata_video.xlsx and metadata_image.xlsx
python metadata.py -r -o csv -x                 # no blank rows between folders
python metadata.py -n -p                        # preview renaming
```

### Ordering

- `name`: one list ordered by file name; with `-r` the folder is shown in front of each name.
- `folder` (with `-r`): ordered by folder, then file name, with a summary per folder.
  In Excel this is a single sheet with a *Folder* column (filter-ready) plus a
  *Summary* sheet.

### Videos and images together or separate

By default (`--kinds together`) videos and images share one listing, distinguished by
the *Type* column. With `--kinds separate` you get two outputs: on screen two sections
(videos, then images), and with files `metadata_video.*` and `metadata_image.*`, each
with its own summaries. A kind with no files produces no output.

### Blank lines between folders

When ordered by folder (`-r`), CSV and Excel output get an empty row between folders so
the groups are easy to see. Pass `-x` for one unbroken table instead, which is easier to
sort and filter in Excel. In the screen and text reports `-x` removes the gap after each
folder summary.

### Output files

Files are written to the **current directory** as `metadata.txt`, `metadata.csv` and
`metadata.xlsx`. An existing file is never overwritten unless you pass `--overwrite`;
otherwise `metadata_01.xlsx`, `metadata_02.xlsx`, ... is used. If the target is locked
(for example open in Excel) a new name is used automatically.

## Problem files

A problem file never stops the run. It is listed with an *Error* (and repeated in a
list on the terminal at the end), and the rest is processed normally.

| Situation | Result |
|-----------|--------|
| Locked by another program / no read permission | `Access denied or file is locked by another program` |
| Empty file | `Empty file (0 bytes)` |
| Corrupt, truncated or not really media | `Error` with the reason |
| Reading takes more than 60 s | `Timed out ...` |
| Folder that cannot be read | warning, scan continues |
| Mis-labeled file (e.g. a JPEG named `file1.mp4`) | read by its **content**, reported as an image, with a *Note* |
| Extension differs from content (e.g. PNG named `.jpg`) | read normally with a *Note* |

## Renaming

`--rename` turns `holiday.mp4` into `holiday (1920x1080 - 0-02-03).mp4` and `photo.jpg`
into `photo (4000x3000).jpg`. It never overwrites an existing file, skips files that
already carry the suffix, and reports failures per file. Use `--dry-run` first.

## Exit codes

| Code | Meaning |
|------|---------|
| 0 | done (files with errors are reported, not fatal) |
| 1 | missing prerequisite, folder not found, nothing found, or an output file could not be written |
| 2 | bad command line |
| 130 | interrupted with Ctrl+C |

## Differences from the earlier scripts

`metadata.py` replaces the earlier `meta.py`, `metas.py`, `metas2.py` and `metax.py`
(no longer part of this repository). Mapping of old switches: `-R` recursive (same),
`-N` is now `-n`/`--rename`, `-O` overwrite is now `-w`, `-S` is now `-s name|folder`,
`-CU`/`-CE` are `-o csv` (with `-d ";"` for the European style), `-XLSX` is `-o xlsx`.
The per-file output option `-F` and file-name patterns were dropped; use `--type` instead.

## Changelog

- **1.3.0** (2026-10-26): EXIF dates are marked "doubtful" when a folder has more than 20
  images all dated within 60 seconds (`--doubtful-count`, `--doubtful-seconds`).
- **1.2.0** (2026-10-26): GPS position, file creation date and an EXIF date check for
  images; `--exif-only` to list only images with valid EXIF.
- **1.1.1** (2026-10-26): fixed Excel output failing ("cannot be used in worksheets") when
  EXIF text such as the camera name contained NUL/control characters; such text is now
  cleaned in all outputs, and "Make Model" no longer repeats the make.
- **1.1.0** (2026-10-26): `--kinds together|separate` for one combined or two per-kind
  outputs; blank line/row between folders when ordered by folder (default), `-x` to omit.
- **1.0.0** (2026-10-26): first unified release. Video and image support; screen, txt, csv
  and xlsx output; name/folder ordering; content-based detection of mis-labeled files;
  graceful handling of locked, unreadable and damaged files; case-insensitive switches;
  `--check`, `--help` and `--version`.
