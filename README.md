 #PayloadGuard: Destructive Merge Detection
 
 Detect catastrophic code payloads before they're merged.
 
##The Problem

41% of shipped code is AI-generated. Dependency automation merges PRs without review. Supply chain attacks hide destructive payloads in legitimate-looking suggestions.

**April 2026 Real Incident:** User received Codex suggestion for "minor syntax fix." Branch was 10 months old, would have deleted 60 files, 11,967 lines, 217 tests, and entire architecture.

## The Solution

PayloadGuard analyzes branches before merge and answers five critical questions:

1. **Scope Question** — How many files/lines change?
2. **Impact Question** — What gets removed?
3. **Temporal Question** — Is this branch current?
4. **Consequence Question** — What breaks after merge?
5. **Transparency Question** — Does the PR description match the actual diff?

## Quick Start
```bashpip
 install -r requirements.txt
python analyze.py /path/to/repo feature-branch main
