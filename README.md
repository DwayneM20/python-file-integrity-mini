# File Integrity Monitor

A small defensive cybersecurity project that monitors a directory for unexpected file changes. The tool creates a trusted baseline of file hashes, then compares future scans against that baseline to detect modified, deleted, or newly added files.

---

## Project Goals

The goal is to build a Python-based File Integrity Monitor that can:

- Create a baseline of files in a target directory
- Calculate SHA-256 hashes for each file
- Save the baseline to a local JSON file
- Re-scan the directory later
- Detect and report:
  - New files
  - Modified files
  - Deleted files
- Output a clear report to the terminal
- Optionally export results to a JSON or CSV report

This project should be defensive only. It should not modify, delete, encrypt, or hide files.

---

## Suggested Tech Stack

- Python 3.10+
- Standard library first:
  - `hashlib`
  - `json`
  - `os`
  - `pathlib`
  - `argparse`
  - `datetime`
  - `csv`
- Optional packages:
  - `watchdog` for real-time monitoring
  - `rich` for nicer terminal output
  - `pytest` for tests

---

## Repository Structure

```text
file-integrity-monitor/
├── README.md
├── requirements.txt
├── fim.py
├── config.example.json
├── baselines/
│   └── .gitkeep
├── reports/
│   └── .gitkeep
├── tests/
│   └── test_fim.py
└── sample_files/
    ├── example.txt
    └── config.txt
```

---

## Setup Instructions

### 1. Clone the Repository

```bash
git clone https://github.com/DwayneM20/python-file-integrity-mini
cd file-integrity-monitor
```

### 2. Create a Virtual Environment

Windows:

```bash
python -m venv venv
venv\Scripts\activate
```

macOS/Linux:

```bash
python3 -m venv venv
source venv/bin/activate
```

### 3. Install Dependencies

For the base version, external packages are not required.

If using optional packages:

```bash
pip install -r requirements.txt
```

Example `requirements.txt`:

```text
watchdog
rich
pytest
```

---

## Core Commands

The final project should support commands similar to these:

### Create a Baseline

```bash
python fim.py baseline --path ./sample_files --output ./baselines/baseline.json
```

Expected behavior:

- Walk through the target directory
- Hash each file using SHA-256
- Store file path, hash, size, and last modified time
- Save results to `baseline.json`

### Scan for Changes

```bash
python fim.py scan --path ./sample_files --baseline ./baselines/baseline.json
```

Expected behavior:

- Recalculate hashes for current files
- Compare current state against the baseline
- Report new, modified, and deleted files

### Export a Report

```bash
python fim.py scan --path ./sample_files --baseline ./baselines/baseline.json --report ./reports/scan_report.json
```

Expected behavior:

- Save scan results to a report file
- Include timestamp, scanned path, and detected changes

---

## Baseline File Format

The baseline should be saved as JSON.

Example:

```json
{
  "created_at": "2026-06-22T14:30:00",
  "root_path": "./sample_files",
  "files": {
    "example.txt": {
      "sha256": "abc123...",
      "size_bytes": 128,
      "modified_time": "2026-06-22T14:20:00"
    }
  }
}
```

---

## Scan Report Format

Example scan report:

```json
{
  "scan_time": "2026-06-22T15:00:00",
  "root_path": "./sample_files",
  "summary": {
    "new_files": 1,
    "modified_files": 2,
    "deleted_files": 1
  },
  "changes": {
    "new": ["new_file.txt"],
    "modified": ["example.txt"],
    "deleted": ["old_config.txt"]
  }
}
```

---

## Milestone 1: Basic File Hashing

### Objective

Build the first version of the script that can calculate a SHA-256 hash for one file.

### Requirements

- Accept a file path
- Read the file safely in chunks
- Return the SHA-256 hash
- Handle missing files gracefully

### Steps

1. Import `hashlib` and `pathlib`.
2. Create a fresh SHA-256 hash object.
3. Open the file in **binary** mode (`"rb"`), not text mode.
4. Read the file in a loop, one fixed-size chunk at a time (e.g. 8192 bytes).
5. Feed each chunk into the hash object as you read it.
6. After the loop ends, return the hexadecimal digest.
7. Wrap the file access so a missing or unreadable file fails cleanly instead of crashing.

### Hints

- Why binary mode? Text mode can rewrite line endings (`\r\n` vs `\n`), which changes the hash across platforms. Binary keeps it stable.
- Why chunks? A multi-GB file should not be loaded into RAM all at once. Chunking keeps memory flat no matter the file size.
- `hexdigest()` returns a string like `"abc123..."`; `digest()` returns raw bytes. The baseline JSON wants the string.
- Calling `.update()` several times is the same as hashing the concatenation of all the pieces — that's exactly why chunking is safe.

### Related Snippets

```python
import hashlib

# Several updates == hashing the whole input at once
h = hashlib.sha256()
h.update(b"hello ")
h.update(b"world")
print(h.hexdigest())
```

```python
# Read a binary file in fixed-size chunks until it's empty
CHUNK = 8192
with open(path, "rb") as f:
    for chunk in iter(lambda: f.read(CHUNK), b""):
        ...  # what should happen with each chunk?
```

```python
# Fail gracefully on files you can't read
try:
    ...
except (FileNotFoundError, PermissionError) as exc:
    ...  # skip it? log it? re-raise? your call
```

### Deliverable

A function similar to:

```python
def hash_file(file_path: Path) -> str:
    ...
```

### Acceptance Criteria

- The function returns the same hash for the same file
- The function returns a different hash when the file content changes
- Large files are read in chunks instead of loading the whole file into memory

---

## Milestone 2: Directory Scanning

### Objective

Scan an entire directory and collect metadata for every file.

### Requirements

- Recursively scan a directory
- Ignore folders such as `.git`, `venv`, `__pycache__`, and `node_modules`
- Store relative file paths instead of absolute paths
- Collect:
  - SHA-256 hash
  - File size
  - Last modified timestamp

### Steps

1. Pick a traversal tool: `pathlib.Path.rglob("*")` or `os.walk()`.
2. For each entry, keep only the ones that are actually files (skip directories).
3. Skip anything inside an ignored folder.
4. Turn each absolute path into a path *relative to* the scan root.
5. For each file, gather its hash (reuse `hash_file` from Milestone 1), size, and last-modified time.
6. Build a dictionary keyed by the relative path string, with the metadata as the value.

### Hints

- Store paths as POSIX strings (`p.as_posix()`) so a baseline made on Windows still matches on Linux/macOS (`\` vs `/`).
- `p.stat()` gives you both size (`.st_size`) and modified time (`.st_mtime`) in one call.
- `.st_mtime` is a float (seconds since epoch). Convert it to something human-readable for the JSON.
- `os.walk` lets you prune ignored folders *before* descending into them, which is faster than filtering afterward.

### Related Snippets

```python
from pathlib import Path

root = Path("./sample_files")
for p in root.rglob("*"):
    if p.is_file():
        rel = p.relative_to(root).as_posix()
        size = p.stat().st_size
        ...
```

```python
import os

IGNORE = {".git", "venv", "__pycache__", "node_modules"}
for dirpath, dirnames, filenames in os.walk(root):
    dirnames[:] = [d for d in dirnames if d not in IGNORE]  # prune in place
    ...
```

```python
from datetime import datetime

# Turn an epoch mtime into a readable ISO timestamp
iso = datetime.fromtimestamp(p.stat().st_mtime).isoformat()
```

### Deliverable

A function similar to:

```python
def scan_directory(root_path: Path) -> dict:
    ...
```

### Acceptance Criteria

- All files in the target directory are detected
- Ignored folders are skipped
- Output is structured and easy to save as JSON

---

## Milestone 3: Baseline Creation

### Objective

Create a trusted baseline file.

### Requirements

- Add a command-line option for baseline creation
- Save scan results to a JSON file
- Include timestamp and root path
- Create the output folder if it does not exist

### Steps

1. Set up `argparse` with subcommands so the tool can grow (`baseline` now, `scan` later).
2. For the `baseline` command, accept `--path` and `--output` arguments.
3. Call `scan_directory(...)` from Milestone 2 to gather the file data.
4. Wrap that data in the baseline shape: `created_at`, `root_path`, and `files` (see **Baseline File Format** above).
5. Make sure the output folder exists before writing.
6. Write the dictionary to disk as nicely formatted JSON.

### Hints

- `add_subparsers(dest="command")` lets you check `args.command == "baseline"` to decide what to run.
- Everything you write must be JSON-serializable — a `Path` or a raw `datetime` object will raise; convert to `str` first.
- `Path.mkdir(parents=True, exist_ok=True)` creates the folder and won't error if it already exists.
- `datetime.now().isoformat()` is a clean value for `created_at`.

### Related Snippets

```python
import argparse

parser = argparse.ArgumentParser()
sub = parser.add_subparsers(dest="command")

b = sub.add_parser("baseline")
b.add_argument("--path", required=True)
b.add_argument("--output", required=True)

args = parser.parse_args()
if args.command == "baseline":
    ...
```

```python
import json
from pathlib import Path

out = Path(args.output)
out.parent.mkdir(parents=True, exist_ok=True)
with out.open("w", encoding="utf-8") as f:
    json.dump(data, f, indent=2)
```

### Example Command

```bash
python fim.py baseline --path ./sample_files --output ./baselines/baseline.json
```

### Acceptance Criteria

- Baseline file is created successfully
- Baseline is valid JSON
- Baseline includes all expected files and hashes

---

## Milestone 4: Change Detection

### Objective

Compare a current directory scan against a saved baseline.

### Requirements

Detect:

- New files: files present now but not in the baseline
- Deleted files: files present in the baseline but missing now
- Modified files: files where the SHA-256 hash changed

### Steps

1. Load the saved baseline JSON from disk.
2. Run a fresh `scan_directory(...)` on the current state of the folder.
3. Compare the two sets of file paths (the dictionary keys).
4. **New** = paths in the current scan but not in the baseline.
5. **Deleted** = paths in the baseline but not in the current scan.
6. **Modified** = paths in *both*, but whose `sha256` value differs.

### Hints

- `dict.keys()` behaves like a set, so you can use `-` (difference) and `&` (intersection) directly on them.
- Compare the **hash** to decide "modified" — not size or mtime. A file can be touched without its contents changing; the hash is the source of truth.
- Decide up front what should happen if the baseline file is missing or corrupt — catch `json.JSONDecodeError` and `FileNotFoundError`.

### Related Snippets

```python
baseline_files = baseline["files"]
current_files = current_scan["files"]

new     = current_files.keys() - baseline_files.keys()
deleted = baseline_files.keys() - current_files.keys()
common  = current_files.keys() & baseline_files.keys()

modified = [
    path for path in common
    if current_files[path]["sha256"] != baseline_files[path]["sha256"]
]
```

```python
import json

try:
    with open(baseline_path, encoding="utf-8") as f:
        baseline = json.load(f)
except (FileNotFoundError, json.JSONDecodeError) as exc:
    ...  # handle a missing or broken baseline gracefully
```

### Example Command

```bash
python fim.py scan --path ./sample_files --baseline ./baselines/baseline.json
```

### Acceptance Criteria

- Modified files are correctly detected
- Deleted files are correctly detected
- New files are correctly detected
- Unchanged files are not reported as changed

---

## Milestone 5: Reporting

### Objective

Make scan results easy to read and share.

### Requirements

- Print a clean summary to the terminal
- Show counts of new, modified, and deleted files
- List affected file paths
- Add optional JSON report export

### Steps

1. Build a `summary` dictionary with the three counts.
2. Print the summary, then each section (new / modified / deleted) with its file list.
3. If a `--report` path was provided, write the full result to JSON (see **Scan Report Format** above).
4. Sort the file lists before printing/saving so the output is stable and predictable.

### Hints

- Wrap the comparison results in a function that *returns data*, and keep printing in a separate function. That way you can test the logic without capturing console output.
- `sorted(...)` your lists so two runs on the same state produce identical output — much easier to test and diff.
- Consider returning a non-zero exit code when changes are found. Automation and CI pipelines watch the exit code to know "something changed."

### Related Snippets

```python
summary = {
    "new_files": len(new),
    "modified_files": len(modified),
    "deleted_files": len(deleted),
}

print("New Files:")
for path in sorted(new):
    print(f"- {path}")
```

```python
import sys

# Signal "changes detected" to the shell / CI
if any([new, modified, deleted]):
    sys.exit(1)
```

### Example Terminal Output

```text
File Integrity Scan Complete

Scanned Path: ./sample_files

Summary:
- New files: 1
- Modified files: 2
- Deleted files: 1

New Files:
- notes.txt

Modified Files:
- config.txt
- example.txt

Deleted Files:
- old_file.txt
```

### Acceptance Criteria

- Terminal output is readable
- JSON report export works
- Reports include timestamp and summary counts

---

## Milestone 6: Configuration File

### Objective

Allow users to configure scan behavior.

### Requirements

Create a `config.example.json` file.

Example:

```json
{
  "scan_path": "./sample_files",
  "baseline_path": "./baselines/baseline.json",
  "report_path": "./reports/scan_report.json",
  "ignore_dirs": [".git", "venv", "__pycache__", "node_modules"],
  "ignore_extensions": [".tmp", ".log"]
}
```

### Steps

1. Add an optional `--config` argument that points at a JSON config file.
2. If given, load the config; otherwise fall back to defaults.
3. Decide a clear precedence order for settings.
4. Apply `ignore_dirs` during traversal and `ignore_extensions` when deciding whether to hash a file.

### Hints

- Pick one precedence rule and stick to it: explicit CLI flag > config file value > built-in default. Document it.
- The tool must still work with **no** config file — config is a convenience, not a requirement.
- `Path(name).suffix` gives the extension (including the dot, e.g. `".log"`), which lines up with `ignore_extensions`.
- Turning the ignore lists into `set`s makes the membership checks fast and clean.

### Related Snippets

```python
import json

config = {}
if args.config:
    with open(args.config, encoding="utf-8") as f:
        config = json.load(f)

# CLI flag wins, then config, then a default
scan_path  = args.path or config.get("scan_path")
ignore_ext = set(config.get("ignore_extensions", []))
```

```python
from pathlib import Path

if Path(filename).suffix in ignore_ext:
    continue  # skip this file
```

### Acceptance Criteria

- User can run the tool with a config file
- Ignored directories are respected
- Ignored file extensions are respected

---

## Milestone 7: Testing

### Objective

Add automated tests for the main logic.

### Suggested Tests

- Hashing the same file twice returns the same hash
- Changing file content changes the hash
- New files are detected
- Deleted files are detected
- Modified files are detected
- Ignored directories are skipped
- Invalid baseline files are handled gracefully

### Steps

1. Create `tests/test_fim.py` and import the functions you want to test.
2. Use pytest's `tmp_path` fixture so each test gets a clean, throwaway folder.
3. Create files inside `tmp_path`, run your function, and assert on the result.
4. Follow Arrange → Act → Assert: set up the files, call the function, check the outcome.

### Hints

- `tmp_path` is a `Path` that pytest creates fresh per test, so your tests never touch real folders or hardcode machine-specific paths.
- `Path.write_text(...)` is the quickest way to create and later change a test file's contents.
- Test the data, not the print output — assert on what your scan/compare functions *return*.
- To test "modified," write a file, hash it, change it, hash again, and assert the two hashes differ.

### Related Snippets

```python
def test_same_content_same_hash(tmp_path):
    f = tmp_path / "a.txt"
    f.write_text("data")
    assert hash_file(f) == hash_file(f)
```

```python
def test_hash_changes_when_content_changes(tmp_path):
    f = tmp_path / "a.txt"
    f.write_text("hello")
    first = hash_file(f)
    f.write_text("hello!")
    assert hash_file(f) != first
```

### Example Command

```bash
pytest
```

### Acceptance Criteria

- Tests run successfully
- Core functionality is covered
- Test files do not depend on the developer's local machine paths

---

## Stretch Goals

These are optional if you finish the core project early.

### 1. Real-Time Monitoring

Use `watchdog` to monitor changes as they happen.

Example behavior:

```text
[ALERT] File modified: config.txt
[ALERT] New file created: suspicious.py
```

### 2. CSV Report Export

Allow reports to be exported as CSV.

```bash
python fim.py scan --path ./sample_files --baseline ./baselines/baseline.json --csv ./reports/scan_report.csv
```

### 3. Email Alerts

Send an email when changes are detected.

Useful for learning:

- SMTP
- Environment variables
- Secret handling

Important: credentials should never be hardcoded.

### 4. Severity Levels

Assign severity levels based on file type or path.

Examples:

- Changes to `.py`, `.exe`, `.dll`, `.sh`, `.bat`: High
- Changes to `.txt`, `.md`: Low
- Changes under `config/`: Medium or High

### 5. Simple Web Dashboard

Use Flask or FastAPI to display scan results in a browser.

Possible pages:

- Latest scan summary
- List of changed files
- Baseline creation page
- Scan history

---

## Security Considerations

You should follow these rules:

- Do not store secrets in the repository
- Do not scan directories without permission
- Do not modify files during scanning
- Do not follow symlinks unless explicitly configured
- Validate user-provided paths
- Store baselines and reports outside sensitive directories when possible
- Add `.gitignore` rules for generated baselines and reports if they may contain sensitive file names

Suggested `.gitignore`:

```text
venv/
__pycache__/
*.pyc
baselines/*.json
reports/*.json
reports/*.csv
.env
```

---

## Definition of Done

The project is complete when:

- A user can create a baseline for a folder
- A user can scan the same folder later
- The tool correctly reports new, modified, and deleted files
- Reports are readable in the terminal
- Optional JSON report export works
- The README explains setup and usage
- Basic tests are included and passing

---

## Suggested Timeline

### Week 1

- Learn the project goals
- Implement file hashing
- Implement directory scanning
- Create baseline JSON file

### Week 2

- Implement scan comparison
- Detect new, modified, and deleted files
- Add terminal reporting
- Add JSON report export

### Week 3

- Add config file support
- Add tests
- Clean up README
- Demo the project

### Optional Week 4

- Add real-time monitoring
- Add CSV export
- Add email alerts
- Add a small dashboard

---

## Final Demo Expectations

You should be able to demonstrate:

1. Creating a baseline
2. Modifying a file
3. Adding a new file
4. Deleting a file
5. Running a scan
6. Showing the detected changes
7. Exporting a report
8. Explaining how SHA-256 hashing is used to detect file changes

---

## Learning Outcomes

By completing this project, you should understand:

- How file hashing works
- Why baselines are useful in cybersecurity
- How to detect unauthorized file changes
- How to structure a small Python CLI project
- How to work with JSON and CSV reports
- How to write basic tests for security tooling
- How defensive monitoring tools fit into real-world security operations
