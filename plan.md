# Plan: 9.5 → 10.0

> **Goal**: Raise overall score from 9.5 to 10.0 (110 / 11 = perfect average across all categories).
> **Last updated**: 2026-05-31
> **Progress**: 14 / 14 actionable items complete (item 15 retired). Item 7 resolved via `c1fb473`. **Score: 10.0 / 10.**

---

## Score gap

| Category | Now | Target | Delta | Status |
|---|---|---|---|---|
| Architecture & Design | **10** | 10 | — | ✅ `968cf65` |
| Code Quality & Standards | **10** | 10 | — | ✅ `c1fb473` (`make mutation` + mutmut) |
| Testing & Coverage | **10** | 10 | — | ✅ `bdc552c` |
| Documentation | **10** | 10 | — | ✅ `ddcdcc8` |
| CI/CD & DevOps | **10** | 10 | — | ✅ `95507a0` |
| Security | **10** | 10 | — | ✅ `7263d5b` + `b44729c` |
| Developer Experience | **10** | 10 | — | ✅ `2650576` |
| Dependency Management | **10** | 10 | — | ✅ `ff0b4f4` |
| Maintainability & Extensibility | **10** | 10 | — | ✅ `7c53a09` + `d5a2b31` |
| Performance | **10** | 10 | — | ✅ `125135b` + `532d0ce` |
| Configuration & Tooling | **10** | 10 | — | ✅ `e031087` invariant (item 15 retired) |

All 11 categories at 10/10. **Final score: 10.0 / 10.**

---

## Completed items (14 / 14)

| Item | Commit | Category impact |
|---|---|---|
| 1 Bandit CI gate | `7263d5b` | Security 9→10 |
| 2 Gitleaks | `b44729c` | Security (partial) |
| 3 Bundle compat matrix | `7c53a09` | Maintainability 8→10 |
| 4 Global patch docs | `d5a2b31` | Maintainability 8→10 |
| 5a pytest-xdist | `0eb4e8c` | Performance 6→7 |
| 5b Marimo CI timeout | `3e1d07f` | Performance 7→8 |
| 6 DAG validation | `968cf65` | Architecture 9→10 |
| 7 Mutation testing (mutmut) | `c1fb473` | Code Quality 9→10 |
| 8 CI parity test | `95507a0` | CI/CD 9→10 |
| 9 New bundle tutorial | `ddcdcc8` | Documentation 9→10 |
| 10 pytest-timeout + sync failure tests | `bdc552c` | Testing 9→10 |
| 11 make doctor + troubleshooting guide | `2650576` | DX 9→10 |
| 12 uv dependency groups | `ff0b4f4` | Dependency Management 9→10 |
| 13 per-job timeouts + CI caching audit | `125135b` | Performance 8→9 |
| 14 docs build caching + benchmark CI job | `532d0ce` | Performance 9→10 |

---

## What to skip

| Item | Why skip |
|---|---|
| Plugin registry / bundle versioning | Architectural change requiring rhiza-cli coordination; scope is this repo only |
| DAST / fuzzing | No dynamic attack surface — template system has no HTTP interface or user-controlled inputs |
| `pyright`/`mypy` for 3.11/3.12 | `ty` runs across full matrix per `9a08e87`; a second type checker is redundant, not additive |
| e2e GitHub Actions workflow testing | Requires a sandbox GitHub account and is operationally complex; static parity test (`95507a0`) covers structural drift |
| macOS BSD Make guard | Documented in `333bada`; all CI uses GNU Make; adding a runtime guard to every Makefile target is noise |
| Bundle config drift detection (item 15) | `e031087` invariant test already prevents any file from existing in two bundles — drift between copies is structurally impossible |
| ~~Mutation testing / mutmut (item 7)~~ | Completed — `c1fb473` merged 2026-05-31 |

---

## Final result

All 14 actionable items complete. Item 15 retired.

| State | Score | Formula |
|---|---|---|
| All 11 categories at 10/10 | **10.0** | 110 / 11 = 10.00 |
