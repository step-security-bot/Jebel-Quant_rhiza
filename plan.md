# Plan: 9.5 → 10.0

> **Goal**: Raise overall score from 9.5 to 10.0 (110 / 11 = perfect average across all categories).
> **Non-goal**: Skipping categories — every category must reach 10; no floor of 9 is acceptable.
> **Last updated**: 2026-05-28
> **Progress**: 10 / 15 items complete

---

## Score gap

| Category | Now | Target | Delta | Status |
|---|---|---|---|---|
| Architecture & Design | **10** | 10 | — | ✅ `968cf65` |
| Code Quality & Standards | 9 | 10 | +1 | Item 7 ⏳ pending |
| Testing & Coverage | **10** | 10 | — | ✅ `bdc552c` |
| Documentation | **10** | 10 | — | ✅ `ddcdcc8` |
| CI/CD & DevOps | **10** | 10 | — | ✅ `95507a0` |
| Security | **10** | 10 | — | ✅ `7263d5b` + `b44729c` |
| Developer Experience | **10** | 10 | — | ✅ `2650576` |
| Dependency Management | 9 | 10 | +1 | Item 12 ⏳ pending |
| Maintainability & Extensibility | **10** | 10 | — | ✅ `7c53a09` + `d5a2b31` |
| Performance | 8 | 10 | +2 | Items 13, 14 ⏳ pending |
| Configuration & Tooling | 9 | 10 | +1 | Item 15 ⏳ pending |

Five category-points needed across 4 categories. Performance carries the largest individual gap (+2).

---

## Completed items (10 / 15)

| Item | Commit | Category impact |
|---|---|---|
| 1 Bandit CI gate | `7263d5b` | Security 9→10 |
| 2 Gitleaks | `b44729c` | Security (partial) |
| 3 Bundle compat matrix | `7c53a09` | Maintainability 8→10 |
| 4 Global patch docs | `d5a2b31` | Maintainability 8→10 |
| 5a pytest-xdist | `0eb4e8c` | Performance 6→7 |
| 5b Marimo CI timeout | `3e1d07f` | Performance 7→8 |
| 6 DAG validation | `968cf65` | Architecture 9→10 |
| 8 CI parity test | `95507a0` | CI/CD 9→10 |
| 9 New bundle tutorial | `ddcdcc8` | Documentation 9→10 |
| 10 pytest-timeout + sync failure tests | `bdc552c` | Testing 9→10 |
| 11 make doctor + troubleshooting guide | `2650576` | DX 9→10 |

---

## Pending items (ordered by effort)

### 7. Add mutation testing with mutmut — 3 h ⏳ Pending · [#1109](https://github.com/Jebel-Quant/rhiza/issues/1109)

**Fixes**: Code Quality 9→10
The suite has 90% line coverage but no verification that tests are discriminating — a test can pass even if the logic it
covers is inverted. Add `mutmut` targeting `.rhiza/utils/` and `tests/` utility modules. Run via `make mutation-test`
(separate from `make test` — mutation runs are slow). Add a CI job (`rhiza_mutation.yml`) triggered on `push` to `main`
that runs `mutmut run` on a declared subset of files and fails if the mutation score drops below 80%. Document the
target in `make help` output.

```toml
# pyproject.toml
[tool.mutmut]
paths_to_mutate = [".rhiza/utils/", "tests/"]
tests_dir = "tests/"
```

---

### 12. Add `uv` optional dependency groups — 1 h ⏳ Pending · [#1112](https://github.com/Jebel-Quant/rhiza/issues/1112)

**Fixes**: Dependency Management 9→10
The dev dependency set pulls in Marimo, NumPy, Pandas, and Plotly even for contributors who only need to run linting or
tests. Restructure `pyproject.toml` into named `uv` dependency groups:

```toml
[dependency-groups]
lint   = ["ruff>=0.4", "interrogate>=1.7", "pre-commit>=3.7"]
test   = ["pytest>=8", "pytest-cov>=5", "hypothesis>=6", "mutmut>=2",
          "pytest-timeout>=2", "pytest-xdist>=3"]
docs   = ["marimo>=0.7", "mkdocs-material>=9", "pdoc>=14",
          "numpy>=1.26", "pandas>=2", "plotly>=5"]
```

Contributors who only want to run the linting suite: `uv sync --group lint`. CI matrix jobs that don't need
documentation: `uv sync --group test`. Document the groups in `CONTRIBUTING.md` and update the DevContainer setup
script. Maintains a top-level `uv sync` (all groups) for full development.

---

### 13. Add per-job `timeout-minutes` + CI caching audit — 2 h ⏳ Pending · [#1113](https://github.com/Jebel-Quant/rhiza/issues/1113)

**Fixes**: Performance 8→9
Item 5b closed the Marimo gap (Performance now 8). The remaining deduction is "no explicit CI time budget or caching
strategy." Two sub-items:

(a) **Per-job timeouts**: audit every job in `.github/workflows/rhiza_ci.yml` and `.gitlab-ci.yml` and add
`timeout-minutes` (GitHub) / `timeout` (GitLab) values grounded in measured baseline. Proposed budgets: lint/format job
≤ 5 min, test matrix job ≤ 20 min, security scan job ≤ 10 min, docs build job ≤ 10 min. Add a comment in each workflow
documenting the budget and the date it was last measured.

(b) **Caching audit**: verify that all 12 test matrix jobs share a common `uv` cache key (`${{ runner.os }}-uv-${{
hashFiles('uv.lock') }}`) and a `pre-commit` cache key (`${{ runner.os }}-pre-commit-${{
hashFiles('.pre-commit-config.yaml') }}`). Add a `docs/operations/CI_PERFORMANCE.md` documenting expected cache hit
rates, cache TTL, and how to force a cold run for debugging.

---

### 14. Docs build caching + benchmark CI job — 3 h ⏳ Pending · [#1115](https://github.com/Jebel-Quant/rhiza/issues/1115)

**Fixes**: Performance 9→10
After item 13 closes the time-budget gap (8→9), the remaining deductions are "documentation build time not tracked" and
"benchmark infrastructure exists but is not active in CI." Two sub-items:

(a) **MkDocs build caching**: add `actions/cache` for the `.cache/plugin/` directory used by MkDocs Material. Set
`timeout-minutes: 10` on the `make book` CI step and add a build-time annotation using `GITHUB_STEP_SUMMARY` that
reports wall time:

```yaml
- name: Build docs
  timeout-minutes: 10
  run: |
    START=$(date +%s)
    make book
    echo "Docs build: $(($(date +%s) - START))s" >> $GITHUB_STEP_SUMMARY
```

(b) **Benchmark CI job**: add a `rhiza_benchmark.yml` workflow (triggered on `push` to `main`, not on every PR) that
runs `make benchmark` and posts results to the workflow summary. This activates the existing `bundles/benchmarks/`
infrastructure and creates a measurable CI performance baseline over time.

---

### 15. Bundle config drift detection test — 2 h ⏳ Pending · [#1116](https://github.com/Jebel-Quant/rhiza/issues/1116)

**Fixes**: Configuration & Tooling 9→10
The remaining Configuration & Tooling deduction is config duplication across bundles. Bundle isolation requires each
bundle to own its files, but when a canonical config (e.g., `ruff.toml`) appears in multiple bundles, all copies must
stay in sync. Currently this is enforced only by the `GLOBAL_PATCH.md` workflow — a human process.

Add a machine-enforced gate: a YAML manifest `bundle-config-manifest.yml` at the repo root listing files that must be
byte-identical across all bundles that carry them:

```yaml
# bundle-config-manifest.yml
shared_configs:
  ruff.toml:
    canonical: bundles/core/.rhiza/
    copies:
      - bundles/tests/.rhiza/
      - bundles/github-tests/.rhiza/
```

A pytest test (`tests/bundles/test_config_drift.py`) reads this manifest and fails if any copy's SHA-256 diverges from
the canonical. The test message names the file and the diverging bundle. Register in `make validate`. Intentional
divergences are opt-out via a `diverges: true` flag in the manifest entry.

---

## What to skip

| Item | Why skip |
|---|---|
| Plugin registry / bundle versioning | Architectural change requiring rhiza-cli coordination; scope is this repo only |
| DAST / fuzzing | No dynamic attack surface — template system has no HTTP interface or user-controlled inputs |
| `pyright`/`mypy` for 3.11/3.12 | `ty` runs across full matrix per `9a08e87`; a second type checker is redundant, not additive |
| e2e GitHub Actions workflow testing | Requires a sandbox GitHub account and is operationally complex; static parity test (`95507a0`) covers structural drift |
| macOS BSD Make guard | Documented in `333bada`; all CI uses GNU Make; adding a runtime guard to every Makefile target is noise |

---

## Expected result

All 15 items complete. Score progression:

| Milestone | Score | Formula |
|---|---|---|
| Current (10 items done) | **9.5** | 105 / 11 |
| After item 7 (Code Quality 9→10) | **9.6** | 106 / 11 ≈ 9.64 |
| After item 12 (Dependency 9→10) | **9.7** | 107 / 11 ≈ 9.73 |
| After items 13 + 14 (Performance 8→10) | **9.9** | 109 / 11 ≈ 9.91 |
| After item 15 (Config & Tooling 9→10) | **10.0** | 110 / 11 = 10.00 |

Final category targets:

| Category | Score |
|---|---|
| Architecture & Design | 10 / 10 |
| Code Quality & Standards | 10 / 10 |
| Testing & Coverage | 10 / 10 |
| Documentation | 10 / 10 |
| CI/CD & DevOps | 10 / 10 |
| Security | 10 / 10 |
| Developer Experience | 10 / 10 |
| Dependency Management | 10 / 10 |
| Maintainability & Extensibility | 10 / 10 |
| Performance | 10 / 10 |
| Configuration & Tooling | 10 / 10 |
| **Overall** | **10.0 / 10** |
