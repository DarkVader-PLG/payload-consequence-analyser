# PayloadGuard — System Architecture Whitepaper

> *"Because AI doesn't feel bad about what it breaks."*

---

## Contents

1. [Origin](#1-origin)
2. [Problem Statement](#2-problem-statement)
3. [Threat Model](#3-threat-model)
4. [System Overview](#4-system-overview)
5. [Layer 1 — Surface Scan](#5-layer-1--surface-scan)
6. [Layer 2 — Forensic Analysis](#6-layer-2--forensic-analysis)
7. [Layer 3 — Consequence Model](#7-layer-3--consequence-model)
8. [Layer 4 — Structural Drift](#8-layer-4--structural-drift)
9. [Layer 5a — Temporal Drift](#9-layer-5a--temporal-drift)
10. [Layer 5b — Semantic Transparency](#10-layer-5b--semantic-transparency)
11. [Configuration System](#11-configuration-system)
12. [CI Integration](#12-ci-integration)
13. [GitHub App Integration](#13-github-app-integration)
14. [Output Formats](#14-output-formats)
15. [Packaging and Distribution](#15-packaging-and-distribution)
16. [File Structure](#16-file-structure)
17. [Limitations and Future Work](#17-limitations-and-future-work)

---

## 1. Origin

In April 2026 a developer received a code suggestion from an AI coding assistant (Codex). The suggestion was described as a *"minor syntax fix"*. The branch carrying it had been open for 10 months. The change passed an informal review without scrutiny.

Had it been merged, it would have:

- Deleted **61 files**
- Removed **11,967 lines** of code
- Wiped **217 tests**
- Destroyed the entire application architecture

The signals were all there:
- 312 days of branch age on a fast-moving repo
- 98.2% deletion ratio
- Critical structural nodes gone from core auth and session modules
- A description that directly contradicted the diff

No existing tool surfaced any of it. PayloadGuard was built to ensure that combination of signals is never invisible again.

---

## 2. Problem Statement

Modern development pipelines face a class of risk that standard code review tooling was not designed to detect: **high-impact destructive changes disguised as low-impact modifications**.

This risk is amplified by:

- **AI-assisted code generation** — tools that produce large, confident diffs with no understanding of architectural consequence
- **Long-lived branches** — semantic drift accumulates silently as the target branch evolves
- **Social engineering via PR description** — a benign description reduces reviewer scrutiny regardless of diff content
- **Volume pressure** — reviewers on high-velocity teams cannot read every diff with the depth it requires

Standard CI tooling (linters, test runners, type checkers) verifies correctness within a changeset. None of them answer the question: *how destructive is this merge relative to what currently exists?*

PayloadGuard answers that question.

---

## 3. Threat Model

PayloadGuard is designed to detect the following patterns:

| Pattern | Description |
|---|---|
| **Architectural gutting** | A file is "modified" but its entire class/function surface is deleted |
| **Silent mass deletion** | Large numbers of files deleted across critical paths (tests, CI, config) |
| **Stale branch injection** | A branch with extreme temporal drift carries changes whose context is invalidated |
| **Deceptive payload** | PR description claims low impact while diff is catastrophically destructive |
| **Ratio inversion** | Changeset is overwhelmingly deletions — additions used as cover |

PayloadGuard does **not** detect:

- Logic errors within unchanged functions
- Security vulnerabilities in new code
- Performance regressions
- Dependency confusion attacks

It is a **destructive merge detector**, not a general-purpose code quality tool.

---

## 4. System Overview

```
┌─────────────────────────────────────────────────────────┐
│                        analyze.py                        │
│                                                         │
│  PayloadAnalyzer.analyze()                              │
│  ├── Layer 1: Surface Scan          (inline)            │
│  ├── Layer 2: Forensic Analysis     (inline)            │
│  ├── Layer 3: Consequence Model     (_assess_consequence)│
│  ├── Layer 4: Structural Drift      (structural_parser) │
│  ├── Layer 5a: Temporal Drift       (TemporalDriftAnalyzer)│
│  └── Layer 5b: Semantic Transparency(SemanticTransparencyAnalyzer)│
│                                                         │
│  Output: unified report dict                            │
│  ├── print_report()    → stdout (terminal)              │
│  ├── save_json_report() → .json                         │
│  └── save_markdown_report() → .md (CI comment body)    │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│              structural_parser.py                        │
│                                                         │
│  extract_named_nodes(source, path) → set[str]           │
│  ├── .py  → stdlib ast                                  │
│  ├── .js/.jsx  → tree-sitter-javascript                 │
│  ├── .ts  → tree-sitter-typescript (language_typescript)│
│  ├── .tsx → tree-sitter-typescript (language_tsx)       │
│  ├── .go  → tree-sitter-go                              │
│  ├── .rs  → tree-sitter-rust                            │
│  └── .java → tree-sitter-java                           │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│              post_check_run.py                           │
│                                                         │
│  GitHub App authentication:                             │
│  private key → JWT → installation token                 │
│  → POST /repos/{repo}/check-runs                        │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│         .github/workflows/payloadguard.yml               │
│                                                         │
│  Trigger: pull_request (opened/synchronize/reopened)    │
│  ├── checkout (fetch-depth: 0)                          │
│  ├── fetch base branch as local ref                     │
│  ├── pip install -r requirements.txt                    │
│  ├── analyze.py → exit code + markdown report           │
│  ├── Post sticky PR comment (github-script)             │
│  ├── Post Check Run (post_check_run.py, optional)       │
│  └── Enforce verdict (exit 1/2 blocks merge)            │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│              action.yml (GitHub Marketplace)             │
│                                                         │
│  Composite action wrapping the full workflow above.     │
│  Inputs: repo-token, pr-description, save-markdown,     │
│          app-id, private-key, installation-id           │
│  Outputs: exit-code, report-path, verdict               │
└─────────────────────────────────────────────────────────┘
```

---

## 5. Layer 1 — Surface Scan

**File:** `analyze.py` → `PayloadAnalyzer.analyze()`

**Purpose:** Establish the raw scope of the changeset.

**Mechanism:** GitPython `merge_base().diff()` against the resolved branch ref. Counts change types: Added (A), Deleted (D), Modified (M), Renamed (R), Copied (C), Type-changed (T).

Line counts are derived by reading blob content directly — added files count all lines, deleted files count all lines, modified files count net delta via diff output.

**Outputs:**
```python
files: { added, deleted, modified, renamed, copied, type_changed, total_changed }
lines: { added, deleted, net_change, deletion_ratio_percent }
```

**Key signal:** `deletion_ratio_percent` — fraction of total line churn that is removal. Feeds into Layer 3 scoring.

---

## 6. Layer 2 — Forensic Analysis

**File:** `analyze.py` — `CRITICAL_PATH_PATTERNS`

**Purpose:** Identify which deleted files are architecturally significant.

**Mechanism:** Regex matching against deleted file paths. Patterns target:

```python
CRITICAL_PATH_PATTERNS = [
    r"(^|/)tests?(/|$)",           # test directories
    r"(^|/)test_[^/]+$",           # test_*.py files
    r"(^|/)\.github/",             # CI/workflow files
    r"(^|/)requirements[^/]*\.txt$",
    r"(^|/)setup\.py$",
    r"(^|/)__init__\.py$",
    r"(^|/)core(/|$)",
    r"(^|/)modules(/|$)",
    r"(^|/)config(/|$)",
    r"\.(yml|yaml)$",
]
```

Regex over substring matching prevents false positives (e.g. `protest.py` matching `test`).

**Output:** `deleted_files: { total, critical[], all[] }`

---

## 7. Layer 3 — Consequence Model

**File:** `analyze.py` → `PayloadAnalyzer._assess_consequence()`

**Purpose:** Produce a single weighted verdict from all signals.

**Mechanism:** Additive scoring. Points accumulate across independent signals — no single signal triggers DESTRUCTIVE. It is the combination that matters.

| Signal | Thresholds | Points |
|---|---|---|
| Branch age | > 90d / 180d / 365d | 1 / 2 / 3 |
| Files deleted | > 10 / 20 / 50 | 1 / 2 / 3 |
| Deletion ratio | > 50% / 70% / 90% | 1 / 2 / 3 |
| Structural severity | CRITICAL | 3 |
| Lines deleted | > 5k / 10k / 50k | 1 / 2 / 3 |

**Verdicts:**

| Score | Verdict | Severity |
|---|---|---|
| 0 | SAFE | LOW |
| 1–2 | REVIEW | MEDIUM |
| 3–4 | CAUTION | HIGH |
| ≥ 5 | DESTRUCTIVE | CRITICAL |

**Exit codes:**
- `0` — SAFE or REVIEW
- `1` — Analysis error
- `2` — DESTRUCTIVE (blocks merge in CI)

All thresholds are configurable via `payloadguard.yml`.

---

## 8. Layer 4 — Structural Drift

**Files:** `analyze.py` → `StructuralPayloadAnalyzer`, `structural_parser.py`

**Purpose:** Detect API surface removal that line diffs cannot reveal. A file can be "modified" with a net delta of +5 lines while losing every class and function it contained.

### Architecture

`structural_parser.py` is a standalone language-dispatch module:

```
extract_named_nodes(source: str, path: str) → set[str]
```

Language is determined by file extension. Python uses stdlib `ast` (zero additional deps, always available). All other languages use tree-sitter with a recursive field-name node walker.

### Why Node Walking, Not Query Strings

Tree-sitter's query execution API (`query.captures()`) changed breaking contracts between versions 0.20, 0.21, and 0.25. The node walker approach — iterating `node.children` and calling `node.child_by_field_name(field)` — is stable across all versions.

### Language Rules

```python
_LANG_RULES = {
    "javascript": {
        "function_declaration": "name",
        "class_declaration":    "name",
        "method_definition":    "name",
        # + variable_declarator with arrow/function value
    },
    "typescript": {   # JS rules +
        "interface_declaration":  "name",
        "type_alias_declaration": "name",
    },
    "go": {
        "function_declaration": "name",
        "method_declaration":   "name",
        "type_spec":            "name",
    },
    "rust": {
        "function_item": "name",
        "struct_item":   "name",
        "enum_item":     "name",
        "trait_item":    "name",
    },
    "java": {
        "class_declaration":     "name",
        "interface_declaration": "name",
        "enum_declaration":      "name",
        "method_declaration":    "name",
    },
}
```

TSX shares the TypeScript rule set. `.jsx` uses JavaScript rules.

### Drift Calculation

For each modified file in a supported language:

```
original_nodes = extract_named_nodes(before, path)
modified_nodes = extract_named_nodes(after, path)
deleted_nodes  = original_nodes - modified_nodes
deletion_ratio = len(deleted_nodes) / len(original_nodes)
```

Flags `CRITICAL` only when **both** conditions are met:
- `deletion_ratio > threshold` (default 0.20)
- `len(deleted_nodes) >= min_count` (default 3)

The dual gate prevents false positives on small utility files where removing one of two functions is 50% ratio but not architecturally significant.

### Graceful Degradation

If a grammar package is not installed, `_load_language()` catches the `ImportError` and returns `None`. The file is silently skipped. The tool remains fully functional for Python-only repos with no tree-sitter packages installed.

---

## 9. Layer 5a — Temporal Drift

**File:** `analyze.py` → `TemporalDriftAnalyzer`

**Purpose:** Quantify how semantically stale a branch is relative to the target, accounting for repo velocity.

**Mechanism:**

```
drift_score = branch_age_days × target_velocity_commits_per_day
```

Target velocity is calculated as commits to the target branch over the trailing 90 days.

Raw age is a weak signal in isolation — a 90-day branch on a repo with 0.1 commits/day (drift = 9) is nothing; the same age on a 10 commits/day repo (drift = 900) is serious.

**Thresholds:**

| Status | Drift Score | Meaning |
|---|---|---|
| CURRENT | < 250 | Branch context is valid |
| STALE | 250–999 | Moderate drift — manual review required |
| DANGEROUS | ≥ 1000 | Rebase mandatory before merge |

**Output:** status, severity, drift score, velocity, recommendation directive.

---

## 10. Layer 5b — Semantic Transparency

**File:** `analyze.py` → `SemanticTransparencyAnalyzer`

**Purpose:** Detect the deceptive payload pattern — a benign PR description masking a destructive diff.

**Mechanism:** Keyword matching against the PR description (lowercased). If a benign keyword is matched **and** the structural verdict is CRITICAL, status is `DECEPTIVE_PAYLOAD`.

**Default benign keywords:**
```
minor fix, minor syntax fix, typo, formatting, cleanup,
docs, refactor whitespace, small tweak, cosmetic, minor update
```

**Statuses:**

| Status | Condition |
|---|---|
| UNVERIFIED | No PR description provided |
| TRANSPARENT | Description provided, not deceptive |
| DECEPTIVE_PAYLOAD | Benign keyword matched + structural verdict CRITICAL |

This is an **advisory signal** — it does not add to the Layer 3 score directly, but surfaces prominently in the report and Check Run output.

Keyword list is fully configurable via `payloadguard.yml`.

---

## 11. Configuration System

**File:** `analyze.py` → `load_config()`, `PayloadGuardConfig`

PayloadGuard ships with sensible defaults and accepts a `payloadguard.yml` in the repo root. Absent keys fall back to defaults via deep merge.

```yaml
thresholds:
  branch_age_days: [90, 180, 365]
  files_deleted:   [10, 20, 50]
  lines_deleted:   [5000, 10000, 50000]
  temporal:
    stale:     250
    dangerous: 1000
  structural:
    deletion_ratio:    0.20
    min_deleted_nodes: 3

semantic:
  benign_keywords:
    - minor fix
    - typo
    - cleanup
```

`_deep_merge()` recursively merges user config over defaults — partial overrides are safe.

---

## 12. CI Integration

**File:** `.github/workflows/payloadguard.yml`

### Trigger
```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened]
```

### Key Design Decisions

**`fetch-depth: 0` + explicit base branch fetch**
GitHub Actions PR checkouts do not create local branches for the base ref — only the PR head is checked out. `git fetch origin main:main` creates the local ref that GitPython needs for `merge_base()` calculation. `fetch-depth: 0` ensures full history for velocity calculation.

**`_resolve_ref()` fallback**
Belt-and-suspenders in Python: if a ref isn't found as a local branch, falls back to `origin/<ref>`. Handles edge cases where the explicit fetch step hasn't run or is unavailable.

**PR body via env var**
`PR_BODY: ${{ github.event.pull_request.body }}` — passing PR body as an env var rather than inline shell interpolation prevents backtick substitution. Inline code in PR descriptions (e.g. `` `--save-markdown` ``) would otherwise be executed as shell commands.

**`PYTHONUTF8: "1"`**
Forces Python's stdout to UTF-8 on GitHub's Ubuntu runners (default locale is ASCII), required for emoji output in the report.

**Sticky PR comment**
`actions/github-script@v7` posts the markdown report as a PR comment. On subsequent pushes to the same PR it finds the existing comment by the `<!-- payloadguard-report -->` marker and updates it rather than creating a new one.

**Merge blocking**
Exit code `2` from `analyze.py` propagates through the Enforce step. A branch protection rule requiring the `scan` check blocks the merge button.

### GitHub Marketplace Action

`action.yml` wraps the full workflow as a composite action. External repos use:

```yaml
- uses: DarkVader-PLG/payload-consequence-analyser@v1
  with:
    repo-token: ${{ secrets.GITHUB_TOKEN }}
    pr-description: ${{ github.event.pull_request.body }}
```

**Inputs:** `repo-token`, `pr-description`, `save-markdown`, `app-id`, `private-key`, `installation-id`
**Outputs:** `exit-code`, `report-path`, `verdict`

---

## 13. GitHub App Integration

**File:** `post_check_run.py`

### Purpose
Post a named **PayloadGuard** Check Run to the PR checks tab, providing a green/red badge distinct from the workflow status check.

### Authentication Flow

```
App private key (RSA PEM)
    ↓
JWT (RS256, iss=App ID, iat=now-60s, exp=now+540s)
    ↓
POST /app/installations/{id}/access_tokens
    ↓
Installation access token (short-lived)
    ↓
POST /repos/{repo}/check-runs
```

### Check Run Conclusions

| Exit code | Conclusion | Title |
|---|---|---|
| 0 | success | PayloadGuard — SAFE |
| 2 | failure | PayloadGuard — DESTRUCTIVE: do not merge |
| other | action_required | PayloadGuard — analysis error |

The full markdown report (up to 65,535 chars, GitHub's limit) is posted as the Check Run summary body.

### Graceful Skip
If `PAYLOADGUARD_APP_ID` is not set, `post_check_run.py` exits cleanly without error. The App integration is entirely optional — the sticky comment and merge blocking work without it.

### Required Secrets

| Secret | Value |
|---|---|
| `PAYLOADGUARD_APP_ID` | GitHub App ID |
| `PAYLOADGUARD_PRIVATE_KEY` | Full PEM private key contents |
| `PAYLOADGUARD_INSTALLATION_ID` | Installation ID from github.com/settings/installations |

---

## 14. Output Formats

### Terminal (stdout)
Human-readable report with emoji section headers. Controlled by `print_report()`.

### JSON (`--save-json`)
Full structured report dict serialised with `json.dump`. All raw metrics included. Useful for downstream tooling.

### Markdown (`--save-markdown`)
GitHub-flavoured markdown generated by `format_markdown_report()`. Used as the PR comment body and Check Run summary. Includes:
- Verdict header with emoji
- Flag bullet list
- Tables for all 5 layers
- Collapsible deleted files section (via `<details>`)
- Timestamp footer

---

## 15. Packaging and Distribution

### PyPI
**Package:** `payloadguard`
**Entry point:** `payloadguard = analyze:main`
**Install:** `pip install payloadguard`

`pyproject.toml` uses setuptools flat layout — `py-modules` lists the three top-level modules (`analyze`, `structural_parser`, `post_check_run`). No package directory restructuring required.

### PyPI Publish Workflow
`.github/workflows/publish.yml` triggers on version tags (`v*`). Builds sdist + wheel via `python -m build`, uploads via `twine` using `PYPI_API_TOKEN` secret. Zero manual steps for future releases — tag and done.

### GitHub Marketplace
Listed under **Security** category. Branding: shield icon, red colour. `action.yml` at repo root satisfies Marketplace requirements. Referenced as `uses: DarkVader-PLG/payload-consequence-analyser@v1`.

---

## 16. File Structure

```
payload-consequence-analyser/
├── analyze.py                          # Core engine + CLI entry point
├── structural_parser.py                # Tree-sitter multi-language Layer 4
├── post_check_run.py                   # GitHub App Check Run posting
├── action.yml                          # GitHub Marketplace composite action
├── pyproject.toml                      # PyPI packaging
├── requirements.txt                    # All dependencies
├── test_analyzer.py                    # Test suite
├── README.md                           # User documentation
├── LICENSE                             # MIT
└── .github/
    └── workflows/
        ├── payloadguard.yml            # PR scan CI workflow
        └── publish.yml                 # PyPI auto-publish on tag
```

---

## 17. Limitations and Future Work

### Current Limitations

**Layer 4 — Modified files only**
Structural drift is only computed for `change_type == 'M'` (modified). Renamed files with content changes (`R`) are not analysed. Added files have no prior state to diff against.

**Layer 4 — Top-level names only**
The node walker extracts top-level and class-level names. Nested functions, closures, and anonymous constructs are not tracked.

**Layer 5b — Keyword matching only**
Semantic transparency uses keyword matching, not NLP or embedding-based similarity. Sophisticated descriptions that avoid trigger words are not detected.

**No history awareness**
Each scan is stateless — no memory of previous verdicts on the same PR. A branch that was DESTRUCTIVE yesterday and had files restored today is treated as a fresh scan.

### Future Work

| Item | Priority |
|---|---|
| Renamed file structural analysis | Medium |
| Additional languages (C/C++, Ruby, PHP, Swift) | Medium |
| NLP-based semantic transparency (embedding similarity) | Low |
| VS Code extension — scan before push | Low |
| Web dashboard for team-level reporting | Low |
| Scan history / PR verdict timeline | Low |

---

*PayloadGuard v1.0.0 — April 2026*
*Built in 2 days, on a phone, for £5, after AI nearly deleted everything.*
