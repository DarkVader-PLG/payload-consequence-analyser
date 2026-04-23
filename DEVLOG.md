# PayloadGuard — Developer Log

Reverse-chronological. Most recent entry first.

---

## 2026-04-23 — Audit Hardening Session

Full-day session working through the internal audit (`AUDIT.md`). The audit identified 42 findings across six categories. Today's session addressed 18+ of them across four commits, followed by report contextualisation and the PEM key validation fix.

### Commits (newest first)

- `7307093` — Validate PEM key format before jwt.encode() (§5.4)
- `3cbfd66` — Add contextual interpretation to markdown report sections
- `e111ce9` — Audit hardening: 18 fixes across detection, scoring, security, and test coverage
- `607ed0c` — Harden scoring model and fix detection gaps (§3.1, §3.2, §2.6, §5.3, §6)
- `6d188a2` — Audit hardening — HIGH/MEDIUM fixes (round 1)

---

### Round 1 — HIGH/MEDIUM fixes (`6d188a2`)

**§1.1 / §2.2 / §4.3 — Binary file deletions + memory exhaustion (HIGH)**

Bug: `analyze.py` lines 418–433 read every added/deleted file blob into memory (`data_stream.read()`) and counted newlines manually. Binary files silently decode with `errors='ignore'` and contribute 0 to `lines_deleted`, meaning a PR that deletes large compiled libraries or key files gets no line-count penalty. A single 1 GB file would OOM the runner.

Fix: Replaced the entire manual blob-reading path with `git --numstat`. Git's own output gives integer `added`/`deleted` counts per file (including binary files, which git reports as `-`/`-`); binary entries are counted as 1 line each. Three issues fixed in one change.

**§2.5 — Malformed `payloadguard.yml` crashes analysis (MEDIUM)**

Bug: `yaml.safe_load()` in `analyze.py` line 327 had no try/except. A config file with tabs, wrong types, or truncated YAML raised an uncaught exception and killed the entire run.

Fix: Wrapped `yaml.safe_load()` in try/except; on any `yaml.YAMLError` the loader logs a warning and falls back to defaults, keeping the run alive.

**§5.1 — Partial env var validation in `post_check_run.py` (MEDIUM)**

Bug: `app_id` was checked for presence; `private_key`, `installation_id`, `head_sha`, and `GITHUB_REPOSITORY` were accessed with bare `os.environ[]` — a `KeyError` on any missing variable was swallowed by the outer try/except with no indication of which variable failed. `int(os.environ.get("PAYLOADGUARD_EXIT_CODE", "1"))` raised `ValueError` on non-integer input.

Fix: Added `_require_env(name)` helper that raises `EnvironmentError` with the variable name in the message. Applied to all required variables. `exit_code` parse wrapped in try/except with a sensible fallback.

**§5.2 — Report file path not validated as regular file (MEDIUM)**

Bug: `post_check_run.py` opened `report_path` with no check that it was an actual file. A symlink or named pipe would read unexpected content or hang.

Fix: Added `os.path.isfile(report_path)` guard; non-regular paths raise `EnvironmentError` before the `open()`.

**§4.5 — No retry on GitHub API call in `post_check_run.py`**

Bug: A single `requests` call with `timeout=15`. Any transient GitHub API failure silently dropped the Check Run result.

Fix: Added `requests.Session` with `HTTPAdapter(max_retries=Retry(3, backoff_factor=1, status_forcelist=[502, 503, 504]))`.

**§2.1 — Negative branch age (MEDIUM)**

Bug: `analyze.py` lines 471–475 computed `days_old = (target_date - branch_date).days`. If the branch was newer than the target date, this went negative and `TemporalDriftAnalyzer.analyze_drift()` raised `ValueError` at line 166, caught as a generic exception.

Fix: Clamped `days_old = max(0, (target_date - branch_date).days)`. A branch newer than target is treated as age 0.

**Test infrastructure update**: Updated `_setup_repo` in `test_analyzer.py` to mock `repo.git.diff` with numstat-format output so all existing line-count tests continue to pass against mock repos.

---

### Round 2 — Scoring model and detection gaps (`607ed0c`)

**§3.1 — Correlated signals double-count risk (HIGH)**

Bug: `analyze.py` lines 564–648 awarded independent points for file count, deletion ratio, and line count — three highly correlated signals. A legitimately large cleanup PR could hit DESTRUCTIVE (9+ points) before any structural signals fired.

Fix: The three deletion signals (file count, ratio, lines deleted) are now scored independently then capped: `score = max(individual_scores) + 1` if two or more fire, hard-capped at 4 points total. No single PR can reach DESTRUCTIVE on numbers alone.

**§3.2 — No weighting for code importance (HIGH)**

Bug: `analyze.py` lines 490–494 scored all deleted files equally unless they matched `CRITICAL_PATH_PATTERNS`. Deleting `security/auth.py` (not matching any pattern) was treated identically to deleting a comment-only config file.

Fix: `CRITICAL_PATH_PATTERNS` check now runs before the verdict call; the count of critical file deletions is passed to `_assess_consequence()`. More than 5 critical file deletions adds +2 points; any critical file deletions add +1 point.

**§2.6 — Threshold order not validated (LOW)**

Bug: `analyze.py` lines 307–315 accepted user config like `branch_age_days: [365, 90, 180]` (wrong order) and produced nonsensical tier comparisons.

Fix: Threshold lists loaded from user config are sorted ascending after merge, so out-of-order values are silently corrected.

**§5.3 — Markdown report contains unescaped filenames (LOW–MEDIUM)**

Bug: `analyze.py` lines 748–891 interpolated filenames directly into markdown: `` f"| `{ff['file']}` | ... |" ``. A filename with backticks or pipe characters malformed the table; a controlled filename in a malicious repo scan could inject markdown.

Fix: Added `_md_escape(s)` helper that escapes backticks and pipe characters. Applied to all filename interpolations in `format_markdown_report()`.

**§6 — Test coverage (+18 tests)**

Added test coverage for: binary file deletion, negative branch age, malformed YAML config, threshold order validation, critical path scoring, markdown filename escaping, and `post_check_run._require_env`.

---

### Round 3 — 18-fix consolidation commit (`e111ce9`)

This was a merge of four sub-commits addressing remaining HIGH/MEDIUM items.

**§3.4 — Deletion ratio semantically ambiguous (MEDIUM)**

Bug: Ratio = `deleted / (added + deleted)`. A PR adding 10 lines and deleting 5 reads as 33% (flagged CAUTION). A PR adding 50,000 and deleting 5,000 reads as 9% (fine). The ratio flagged proportional churn, not absolute destructiveness.

Fix: The ratio gate now only fires when `lines_deleted >= 100`. A 10-line PR with a 33% ratio is no longer flagged.

**§2.4 — `iter_commits()` loads everything into memory (MEDIUM)**

Bug: `analyze.py` lines 372–374: `commits = list(self.repo.iter_commits(ref, since=since.isoformat()))` — on repos with millions of commits this loaded the entire result into a Python list.

Fix: Added `max_count=1000` to `iter_commits()` calls in the velocity window to cap memory use.

**§5.5 — Timestamp truncation loses timezone in report (LOW)**

Bug: `analyze.py` line 889: `ts = report.get('timestamp', '')[:16].replace('T', ' ')` stripped timezone and truncated to minute precision.

Fix: Timestamp generation switched to `datetime.now(timezone.utc)`; markdown rendering preserves seconds precision and appends a `UTC` suffix.

**§1.6 — Non-top-level structural deletions missed (MEDIUM–HIGH)**

Bug: `structural_parser.py` lines 127–152 only tracked top-level functions and classes. Deleting constants, annotated assignments, or module-level helpers was invisible to Layer 4.

Fix: `structural_parser.py` now tracks module-level named assignments (`assignment`, `augmented_assignment`) and annotated assignments (`annotated_assignment`) in Python, so `SECRET_KEY = '...'` and `MAX_RETRIES: int = 5` are visible to Layer 4. Rust rules extended with `const_item` and `static_item`; Go rules extended with `const_spec`. +5 tests covering constant extraction and structural detection of deleted constants.

**§1.2 — Merge commits / wrong diff base (MEDIUM)**

Bug: `merge_base()` can return multiple commits for complex histories. Code assumed `[0]` is always the correct fork point; unrelated histories returned an empty list, causing an `IndexError`.

Fix: `merge_base()` result is now checked for empty list; if empty, the analysis returns a structured error (`"unrelated_histories"`) rather than raising.

**§4.1 — Commit author and message data ignored**

GitPython's `commit.author` and `commit.message` were available but never consulted.

Fix: Added `_COMMIT_RED_FLAG_PATTERNS` constant (covering patterns like `remove all tests`, `disable auth`, `bypass security`, `drop database`). Up to 50 commits between merge base and branch tip are scanned; flagged commits are surfaced in `commit_flags` key, printed in the terminal report, and included in the markdown report as advisory signals (no score impact).

+3 tests: empty merge_base error, `commit_flags` key present in result, red-flag commit message detected and surfaced correctly.

**§1.5 — File permission / mode changes not detected (MEDIUM)**

Bug: GitPython's diff API includes `a_mode`/`b_mode` fields for permission changes (e.g. making a script executable). Completely ignored.

Fix: Diff objects are now scanned for `a_mode`/`b_mode` mismatches. Files where the `b_mode` gains executable bits (`& 0o111`) are surfaced in `permission_changes` in the result and printed in the terminal report. +1 test covering executable permission change detection.

**§1.3 — Symlinks and submodules not handled (MEDIUM)**

Bug: Symlinks (mode `0o120000`) and submodules/gitlinks (mode `0o160000`) appeared as regular file changes with no special handling.

Fix: Diff mode bits are inspected for these special values; both are surfaced in `special_files` in the result with file path, type, and change type.

---

### Report contextualisation (`3cbfd66`)

Each section of the markdown report was augmented with three layers of human context:

1. A one-liner explaining what the layer measures and why it matters.
2. Plain-English interpretation of the numbers (e.g. "Branch is 5 days old — context is fresh" rather than just "5 days").
3. Threshold context inline where relevant (e.g. the deletion ratio section shows the CAUTION and REVIEW thresholds so reviewers can see how far from a flag the PR is).

Additional changes: net change now renders as a signed `+/-` value; the deleted files section labels critical-path matches with a note explaining what patterns triggered; the commit message flags section explains what patterns were scanned.

---

### PEM key validation (`7307093`)

**§5.4 — No private key format validation before JWT signing (LOW)**

Bug: `post_check_run.py` lines 24–29 passed `PAYLOADGUARD_PRIVATE_KEY` directly to `jwt.encode()`. A malformed key (missing PEM header/footer) produced a cryptic Rust-level crypto panic that gave no indication of what was wrong.

Fix: Added an upfront check that the key value contains `-----BEGIN RSA PRIVATE KEY-----` (or the PKCS#8 equivalent); if not, `EnvironmentError` is raised with a clear message before the key reaches `jwt.encode()`.

---

## 2026-04-22 / Pre-Audit — Infrastructure, PyPI, README, and Audit Doc

### Test harness wiring (payloadguard-test-harness repo)

Installed the PayloadGuard GitHub App on the `payloadguard-test-harness` repository. Configured three repository secrets: `PAYLOADGUARD_APP_ID`, `PAYLOADGUARD_PRIVATE_KEY`, and `PAYLOADGUARD_INSTALLATION_ID`. Added a workflow file using the composite GitHub Action so that any PR to the test-harness repo triggers a PayloadGuard analysis and posts the result as a Check Run.

### PyPI name clash — renamed to `payloadguard-plg`

The package name `payloadguard` was already taken on PyPI by an unrelated project. Renamed the package to `payloadguard-plg` in `pyproject.toml`, updated all install instructions, and published v1.0.2.

### README rewrite

Rewrote the opening section of `README.md` to explain what PayloadGuard actually does rather than opening with implementation detail. Added a five-layer summary table mapping each layer to what it detects (temporal drift, volume signals, deletion ratio, structural AST changes, pattern matching), a verdict scale explanation (SAFE / CAUTION / REVIEW / DESTRUCTIVE), and a CI integration quickstart.

### Internal audit document (`AUDIT.md`)

Generated a full internal audit covering 42 findings across six categories: detection gaps (§1.1–§1.7), brittle logic (§2.1–§2.6), scoring model weaknesses (§3.1–§3.5), available-but-unused capabilities (§4.1–§4.5), security issues (§5.1–§5.5), and test coverage gaps (§6). Committed as `AUDIT.md` with the note "temporary — delete after review."
