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

These are optional if the intern finishes the core project early.

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

The intern should follow these rules:

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

## Suggested Intern Timeline

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

The intern should be able to demonstrate:

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

By completing this project, the intern should understand:

- How file hashing works
- Why baselines are useful in cybersecurity
- How to detect unauthorized file changes
- How to structure a small Python CLI project
- How to work with JSON and CSV reports
- How to write basic tests for security tooling
- How defensive monitoring tools fit into real-world security operations
