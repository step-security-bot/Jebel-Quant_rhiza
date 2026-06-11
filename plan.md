# Quality Plans

> **Last updated**: 2026-06-11

---

## Plan 2 (June 2026): "10 Across the Board" — in progress (4 / 6 items merged)

> **Trigger**: An independent re-assessment (Claude Fable 5, 2026-06-11) scored the repository **9 / 10**,
> deducting one point for documentation accuracy (~17 dead links, a phantom `CONFIG.md` shipped downstream,
> stale bundle counts, a silently regressed Gitleaks claim in `analysis.md`).
> **Approach**: every fix lands with a test gate in the same PR, turning prose accuracy and the remaining
> soft spots into gated invariants — the same treatment structural invariants already receive.

| Item | Issue / PR | Status | Change | Gate |
|---|---|---|---|---|
| Links | PR #1147 | ✅ merged | All dead links fixed; `bundles/github/.github/CONFIG.md` created | `tests/docs/test_doc_consistency.py`: relative links must resolve in repo or any bundle's downstream layout; every bundle documented in CLAUDE.md |
| A — CI/CD | PR #1148 | ✅ merged | Concurrency groups in all 26 workflows; exact action pinning; fixed gh-aw bundle's broken local action path | `tests/api/test_workflow_hygiene.py` |
| B — Security | PR #1149 | ✅ merged | Gitleaks pre-commit hook (root + core bundle); `curl\|bash` installers replaced with npm; shellcheck widened to all `*.sh` | `TestPipedInstallers` in `tests/security/` (allowlist: astral.sh uv bootstrap only) |
| C — Documentation | #1150 → PR #1154 | ✅ merged | Prose gates: make-target mentions must exist, no hard-coded bundle counts, no "Last Updated" stamps; `analysis.md` corrected | `TestProseDrift` in `tests/docs/test_doc_consistency.py` |
| D — Developer Experience | #1151 | ⏳ open | Shell-completion caching; Windows/WSL quick-start | per-PR |
| E — Dependency Management | #1152 | ⏳ open | ADR 0011: Renovate vs Dependabot division of labour | per-PR |
| F — Code Quality & Maintainability | #1153 | ⏳ open | Ruff exclusion rationale; duplicate make-target gate; new-bundle checklist | per-PR |

The documentation-accuracy deduction that motivated the re-assessment score is closed by the Links + C items
(both merged 2026-06-11). Items D, E, and F are hardening work in categories already at 10/10 in the May
analysis; they keep those scores defensible rather than raising them.

### Remaining work

1. **#1151 (D)** — cache shell-completion generation; add a Windows/WSL quick-start to `CONTRIBUTING.md`.
2. **#1152 (E)** — write ADR 0011 documenting the Renovate vs Dependabot division of labour and align the
   two bot configs with it.
3. **#1153 (F)** — document why each ruff rule family is excluded (enable `A`/`ARG`/`BLE`/`PIE` where
   possible); add a duplicate make-target detection test; add a new-bundle checklist to `EXTENDING_RHIZA.md`.

---

## Plan 1 (May 2026): 9.5 → 10.0 — complete

> **Goal**: Raise overall score from 9.5 to 10.0 (110 / 11 = perfect average across all categories).
> **Completed**: 2026-05-31
> **Progress**: 14 / 14 actionable items complete (item 15 retired). Item 7 resolved via `c1fb473`. **Score: 10.0 / 10.**

### Score gap

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

### Completed items (14 / 14)

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

### What to skip

| Item | Why skip |
|---|---|
| Plugin registry / bundle versioning | Architectural change requiring rhiza-cli coordination; scope is this repo only |
| DAST / fuzzing | No dynamic attack surface — template system has no HTTP interface or user-controlled inputs |
| `pyright`/`mypy` for 3.11/3.12 | `ty` runs across full matrix per `9a08e87`; a second type checker is redundant, not additive |
| e2e GitHub Actions workflow testing | Requires a sandbox GitHub account and is operationally complex; static parity test (`95507a0`) covers structural drift |
| macOS BSD Make guard | Documented in `333bada`; all CI uses GNU Make; adding a runtime guard to every Makefile target is noise |
| Bundle config drift detection (item 15) | `e031087` invariant test already prevents any file from existing in two bundles — drift between copies is structurally impossible |
| ~~Mutation testing / mutmut (item 7)~~ | Completed — `c1fb473` merged 2026-05-31 |

### Final result

All 14 actionable items complete. Item 15 retired.

| State | Score | Formula |
|---|---|---|
| All 11 categories at 10/10 | **10.0** | 110 / 11 = 10.00 |
