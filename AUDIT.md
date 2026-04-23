# PayloadGuard — Internal Audit

Codebase audit across detection gaps, brittle logic, scoring weaknesses, unused capabilities, security issues, and test coverage gaps.

---

## 1. Detection Gaps

### 1.1 Binary File Deletions Don't Contribute to Line Count
**File:** `analyze.py` lines 418–433  
Binary files fail UTF-8 decode silently (`errors='ignore'`) and contribute 0 to `lines_deleted`. A PR deleting critical compiled libraries, databases, or key files gets no line-count penalty — only the file count is captured.  
**Attack pattern:** Delete large binary artifacts. Score stays low.  
**Severity:** HIGH

---

### 1.2 Merge Commits — Wrong Diff Base
**File:** `analyze.py` lines 407–408  
`merge_base()` can return multiple commits for complex histories. Code assumes `[0]` is always the correct fork point. Branches with internal merge commits may be diffed against the wrong base.  
**Attack pattern:** Smuggle changes in via a merge commit from another branch.  
**Severity:** MEDIUM

---

### 1.3 Symbolic Links and Submodules Not Handled
**File:** `analyze.py`, `structural_parser.py`  
Symlinks and submodules are treated as regular files. No special detection.  
**Attack pattern:** Replace a symlink to credentials with a submodule pointing to an attacker's server.  
**Severity:** MEDIUM

---

### 1.4 CRITICAL_PATH_PATTERNS — No Path Context
**File:** `analyze.py` line 46  
`.yml`/`.yaml` matches any YAML file regardless of path. `innocent.yml` in a subdirectory scores the same as `.github/workflows/deploy.yml`.  
**Severity:** LOW

---

### 1.5 File Permission / Mode Changes Not Detected
**File:** `analyze.py`, `structural_parser.py`  
GitPython's diff API includes a mode field for permission changes (e.g., making a script executable). Completely ignored.  
**Attack pattern:** Make shell scripts executable without changing content.  
**Severity:** MEDIUM

---

### 1.6 Non-Top-Level Structural Deletions Missed
**File:** `structural_parser.py` lines 127–152  
`extract_named_nodes()` only tracks top-level functions and classes. Deleting constants, decorators, inline helpers, or nested classes is invisible to Layer 4.  
**Attack pattern:** Delete 50 lines of critical helper code that isn't a named top-level def.  
**Severity:** MEDIUM–HIGH

---

### 1.7 No Distinction Between Generated/Test/Production Code
**File:** `analyze.py` line 408  
No concept of whether deleted files are production code, test fixtures, or generated artifacts. Deleting a large generated bundle inflates deletion counts.  
**Severity:** LOW

---

## 2. Brittle Logic

### 2.1 Negative Branch Age
**File:** `analyze.py` lines 471–475  
`days_old = (target_date - branch_date).days` — if the branch is newer than the target, this goes negative. `TemporalDriftAnalyzer.analyze_drift()` raises `ValueError` at line 166, caught as a general exception and surfaced as an error. Not handled gracefully.  
**Severity:** MEDIUM

---

### 2.2 Memory Exhaustion on Large Files
**File:** `analyze.py` lines 421–433  
Every added/deleted file is fully read into memory (`data_stream.read()`) then split by newline. A single 1GB file will OOM the runner. GitPython's diff object already has `.additions` / `.deletions` counts — these could replace the manual blob reading entirely.  
**Severity:** MEDIUM

---

### 2.3 Single-Branch Clone / Detached HEAD
**File:** `analyze.py` lines 379–387  
Falls back to `origin/{ref}` for unresolved refs. If cloned with `--single-branch` or if the origin renamed a branch, this raises a `BadName` exception leading to a confusing error.  
**Severity:** MEDIUM

---

### 2.4 `iter_commits()` Loads Everything Into Memory
**File:** `analyze.py` lines 372–374  
```python
commits = list(self.repo.iter_commits(ref, since=since.isoformat()))
```
On repos with millions of commits, this loads everything into a Python list. No streaming, no limit.  
**Severity:** MEDIUM

---

### 2.5 Malformed `payloadguard.yml` Crashes Analysis
**File:** `analyze.py` line 327  
`yaml.safe_load()` is called with no try/except. Invalid YAML (tabs, wrong types) raises an uncaught exception and crashes the entire run.  
**Severity:** MEDIUM

---

### 2.6 Threshold Order Not Validated
**File:** `analyze.py` lines 307–315  
Config like `branch_age_days: [365, 90, 180]` (wrong order) is accepted and produces nonsensical scoring. No ascending-order validation.  
**Severity:** LOW

---

## 3. Scoring Model Weaknesses

### 3.1 Correlated Signals Double-Count Risk
**File:** `analyze.py` lines 564–648  
Old branch + many deletions + high deletion ratio are highly correlated. The model awards independent points for each, so a legitimately large-but-old cleanup PR can hit DESTRUCTIVE (9+ points) before structural signals fire.  
**Severity:** HIGH

---

### 3.2 No Weighting for Code Importance
**File:** `analyze.py` lines 490–494  
All deleted files score equally unless they match a `CRITICAL_PATH_PATTERNS` regex. Deleting `security/auth.py` (not matching any pattern) is treated identically to deleting a config comment file.  
**Severity:** HIGH

---

### 3.3 Thresholds Are Arbitrary Defaults
**File:** `analyze.py` lines 277–289  
`90 days = REVIEW` and `50% deletion ratio = CAUTION` have no statistical basis. Fast-moving repos treat a 90-day branch as normal; stable libraries might flag anything over 30 days.  
**Severity:** MEDIUM

---

### 3.4 Deletion Ratio Calculation Is Semantically Ambiguous
**File:** `analyze.py` lines 477–478  
Ratio = `deleted / (added + deleted)`. A PR adding 50,000 lines and deleting 5,000 reads as 9% (fine). A PR adding 10 and deleting 5 reads as 33% (CAUTION). The ratio measures proportion of churn, not absolute destructiveness.  
**Severity:** MEDIUM

---

### 3.5 Structural Deletion Ratio Ignores File Size Context
**File:** `analyze.py` lines 456–458  
A 5-function file losing 1 function = 20% ratio (flagged). A 100-function file losing 5 = 5% (not flagged). Small high-value modules are penalised; large files with meaningful deletions escape.  
**Severity:** MEDIUM

---

## 4. Available but Unused

### 4.1 GitPython — Author, Message, Rename Data Ignored
`commit.author`, `commit.message`, `diff.rename_from`, `diff.rename_to` are all accessible but never consulted. Suspicious authors, red-flag commit messages, and mass renames are invisible.  
**Impact:** Author whitelisting, commit message analysis, rename detection — all zero-cost additions.

---

### 4.2 tree-sitter — Only Deletion Tracked, Not Signatures or Imports
Full AST available for 8 languages. Currently only checks if top-level nodes were deleted. Not checked:
- Changed function signatures
- Deleted imports / dependencies
- Deleted constants or enums
- Language-specific patterns (SQL migrations, schema files)

---

### 4.3 GitPython Diff Object Has Built-In Line Counts
`diff.additions` and `diff.deletions` are computed by Git itself and available directly on the diff object. The current code manually reads blobs and counts `\n` — unnecessary, slower, and causes the memory issue in §2.2.

---

### 4.4 `git blame` Not Consulted
`git.Repo.blame()` is available. Age and authorship of deleted code could distinguish "deleting 3-year-old stable code" from "deleting code added last week."

---

### 4.5 `requests.Session` with Retry Not Used in `post_check_run.py`
`requests` supports retry adapters. Currently a single request with `timeout=15`. A transient GitHub API failure silently drops the Check Run with no retry.

---

## 5. Security Issues

### 5.1 Partial Environment Variable Validation in `post_check_run.py`
**File:** `post_check_run.py` lines 12–22  
`app_id` is checked for presence. `private_key`, `installation_id`, `head_sha`, and `GITHUB_REPOSITORY` are accessed directly with `os.environ[]` — `KeyError` if any is missing, caught by the outer try/except at line 79 with no indication of which variable failed.  
`exit_code = int(os.environ.get("PAYLOADGUARD_EXIT_CODE", "1"))` — invalid string raises `ValueError`.  
**Severity:** MEDIUM

---

### 5.2 Report File Path Not Validated as Regular File
**File:** `post_check_run.py` lines 53–55  
```python
with open(report_path, encoding="utf-8") as f:
    summary = f.read()[:65535]
```
If `report_path` is a symlink or named pipe, this could read unexpected content or hang. No check that it's a regular file.  
**Severity:** MEDIUM

---

### 5.3 Markdown Report Contains Unescaped Filenames
**File:** `analyze.py` lines 748–891  
Filenames and component names are interpolated directly into markdown:
```python
out.append(f"| `{ff['file']}` | {m['deleted_node_count']} |")
```
A filename containing backticks or pipe characters malforms the table. A controlled filename (malicious repo scan) could inject markdown.  
**Severity:** LOW–MEDIUM

---

### 5.4 No Private Key Format Validation Before JWT Signing
**File:** `post_check_run.py` lines 24–29  
If `PAYLOADGUARD_PRIVATE_KEY` is malformed, `jwt.encode()` raises a cryptic exception. No upfront validation that the key is a valid RSA PEM.  
**Severity:** LOW

---

### 5.5 Timestamp Truncation Loses Timezone in Report
**File:** `analyze.py` line 889  
```python
ts = report.get('timestamp', '')[:16].replace('T', ' ')
```
Truncated to minute precision, timezone stripped. Insufficient for audit trails.  
**Severity:** LOW

---

## 6. Test Coverage Gaps

| Gap | File | Severity |
|-----|------|----------|
| No test for binary file deletion | `test_analyzer.py` | HIGH |
| No test for negative branch age (branch newer than target) | `test_analyzer.py` | MEDIUM |
| No test for merge commits in branch history | `test_analyzer.py` | MEDIUM |
| No test for malformed `payloadguard.yml` | `test_analyzer.py` | MEDIUM |
| `post_check_run.py` has zero test coverage | `test_analyzer.py` | MEDIUM |
| No end-to-end test against a real git repo (all tests mock GitPython) | `test_analyzer.py` | MEDIUM |
| No test for empty repo / zero-commit target | `test_analyzer.py` | LOW |
| No test for unicode filenames | `test_analyzer.py` | LOW |
| No test for very large diffs (memory path) | `test_analyzer.py` | LOW |
| No test for symlinks or submodules | `test_analyzer.py` | LOW |
| No test for long filenames / special chars in markdown output | `test_analyzer.py` | LOW |

---

## Priority Actions

1. **Replace manual blob reading with `diff.additions`/`diff.deletions`** — fixes §1.1 (binary files), §2.2 (memory), §4.3 (unused capability) in one change
2. **Add try/except around `yaml.safe_load()`** — §2.5, prevents full crash on bad config
3. **Validate all env vars in `post_check_run.py` with named error messages** — §5.1
4. **Handle negative branch age** — §2.1, return a defined result rather than an exception
5. **Add test for binary deletion, merge commits, malformed YAML, and `post_check_run`** — §6
6. **Add retry logic to `post_check_run.py`** — §4.5
7. **Escape filenames in markdown output** — §5.3
8. **Validate YAML threshold order** — §2.6
