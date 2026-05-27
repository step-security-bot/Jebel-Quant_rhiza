# Plan: 8.9 → 9.5

> **Goal**: Raise overall score from 8.9 to 9.5.  
> **Non-goal**: Perfect scores in every category — Performance stays capped below 10 by design.  
> **Last updated**: 2026-05-27  
> **Progress**: 7 / 9 items complete

---

## Score gap

| Category | Now | Target | Delta | Status |
|---|---|---|---|---|
| Performance | ~~6~~ **7** | 8 | +1 remaining | Item 5a ✅ `0eb4e8c`; item 5b pending |
| Maintainability & Extensibility | ~~8~~ **10** | 10 | — | Items 3+4 ✅ `c5d96df` + `d5a2b31` |
| Security | 9 | 10 | +1 remaining | Item 2 ✅ `b44729c`; item 1 pending |
| Documentation | ~~9~~ **10** | 10 | — | Item 9 ✅ `ddcdcc8` |
| Architecture & Design | ~~9~~ **10** | 10 | — | Item 6 ✅ `968cf65` |
| CI/CD & DevOps | ~~9~~ **10** | 10 | — | Item 8 ✅ `95507a0` |
| Code Quality & Standards | 9 | 10 | +1 remaining | Item 7 pending |
| All others | 9 | 9 | — | — |

Seven category-points needed. Performance and Maintainability carry the largest individual gaps.

---

## Items (ordered by effort)

### 1. Automate Bandit suppression review in CI — 1 h ⏳ Pending
**Fixes**: Security 9→10  
Currently `suppression-audit.sh` exists but the suppressions themselves are never validated in CI — a suppression added for a now-fixed CVE lingers indefinitely. Add a CI step that cross-references active `# nosec` comments against the current `pip-audit` report and fails if any suppression covers a CVE that is no longer flagged (i.e., the code was fixed but the suppression was not removed). The existing `suppression-audit.sh` already parses this data; the job is to make it a blocking CI gate.

---

### 2. Add Gitleaks for deep historical secret scanning — 1 h ✅ `b44729c`
**Fixes**: Security 9→10 (complements item 1)  
GitHub's built-in secret scanning only covers pushed commits going forward. Add a Gitleaks GitHub Actions step (or pre-commit hook) to scan the full history and active working tree for credential patterns. Gitleaks provides a `.gitleaks.toml` for custom rules and false-positive suppression. This closes the gap called out in the analysis without replacing GitHub's native scanner.

```yaml
# .github/workflows/rhiza_ci.yml — security job
- uses: gitleaks/gitleaks-action@v2
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

### 3. Bundle compatibility matrix test — 4 h ✅ `c5d96df`
**Fixes**: Maintainability 8→10 (primary gap)  
Add a parameterised pytest test in `tests/bundles/` that iterates over all 46 bundle×platform combinations (23 bundles × GitHub + GitLab) and asserts: (a) no YAML parse errors, (b) no file owned by two bundles in the same profile, (c) all bundle dependencies are present. This replaces the implicit assumption that combinations are valid with an explicit regression test. Use `itertools.combinations` over `template-bundles.yml` to generate the matrix — no manual enumeration.

---

### 4. Document "global patch" propagation pattern — 2 h ✅ `d5a2b31`
**Fixes**: Maintainability 8→10 (complements item 3)  
The single largest maintainability risk is that propagating a change to all bundles (e.g., adopting a new tool) requires editing 23 files with no atomic workflow. Add a `docs/operations/GLOBAL_PATCH.md` describing the pattern: (1) edit the relevant files in each bundle, (2) run `make validate` to catch inconsistencies, (3) use the new `--all-bundles` flag (or a helper script) to diff affected files across bundles. Optionally add a `make diff-bundles VAR=filename` target that shows the same file across all bundles side-by-side.

---

### 5. Add pytest-xdist and Marimo notebook timeout — 2 h ⏳ Partial
**Fixes**: Performance 6→8  
Two independent sub-items:  
(a) ✅ `0eb4e8c` — Add `pytest-xdist` to dev dependencies and `-n auto` to `pytest.ini` (or a `pytest -n auto` invocation in `make test`). This parallelises test execution across available CPU cores on each CI runner, directly reducing wall time for the 12-combination matrix.  
(b) ⏳ Pending — Add a `timeout-minutes: 10` on the Marimo notebook execution step in `rhiza_marimo.yml` so a runaway computation cannot block the workflow indefinitely.

---

### 6. Formalise bundle dependency DAG with cycle detection — 3 h ✅ `968cf65`
**Fixes**: Architecture 9→10  
`template-bundles.yml` encodes dependencies but they are only consumed at runtime by the rhiza CLI. Add a validation test (or a `make validate` sub-check) that: (a) parses the dependency graph from `template-bundles.yml`, (b) runs cycle detection (topological sort), and (c) fails loudly if a bundle is used without its declared prerequisite in a given profile. This moves bundle correctness from a runtime concern to a pre-commit / CI gate. A simple `graphlib.TopologicalSorter` (stdlib, Python 3.9+) handles this in ~20 lines.

---

### 7. Add mutation testing with mutmut — 3 h ⏳ Pending
**Fixes**: Code Quality 9→10  
The current suite has 90% line coverage but no verification that the tests are discriminating. Add `mutmut` (or `cosmic-ray`) targeting `tests/` utility functions and the bundle sync logic. Run it in a separate `make mutation-test` target (not in the default `make test` — too slow). Add a CI job that runs `mutmut run` on a subset of files and fails if the mutation score drops below a threshold (e.g., 80%). Document the target in `Makefile` help output. This directly closes the "no mutation testing" weakness in the analysis and is the highest-signal remaining Code Quality improvement.

---

### 8. CI/CD: GitLab/GitHub parity smoke test — 2 h ✅ `95507a0`
**Fixes**: CI/CD 9→10  
The analysis flags that GitHub Actions and GitLab CI parity requires manual synchronisation. Add a test (or `make validate` sub-check) that parses both `.github/workflows/rhiza_ci.yml` and `.gitlab-ci.yml` and asserts that: (a) the same job names exist in both, (b) the same Python version matrix is referenced, (c) both have equivalent test/lint/security steps. This does not run both CI systems end-to-end — it statically validates that the structural invariants hold, catching drift before it causes a silent breakage.

---

### 9. Step-by-step "add a new bundle" tutorial — 2 h ✅ `ddcdcc8`
**Fixes**: Documentation 9→10  
`EXTENDING_RHIZA.md` describes the extension mechanism conceptually but lacks a worked example. Add a numbered walkthrough: (1) create the bundle directory, (2) add files, (3) declare dependencies in `template-bundles.yml`, (4) add bundle content tests, (5) run `make validate`, (6) open a PR. Include one real example (e.g., a minimal `linter` bundle that adds a `.ruff.toml` override). This is the highest-friction onboarding gap remaining after the diagram and `make explain-bundles` additions.

---

## What to skip

| Item | Why skip |
|---|---|
| Performance 8→10 | No runtime code; CI parallelism already addresses the actionable gaps |
| Plugin registry / bundle versioning | Architectural change; requires rhiza-cli coordination |
| Stale Bandit suppression manual review | Covered by item 1 (automated) |
| DAST / fuzzing | No dynamic attack surface in this repo |
| `pyright`/`mypy` for 3.11/3.12 | `ty` already runs on full matrix per `9a08e87`; additional checker is redundant |

---

## Expected result

Seven items now complete. Remaining gaps:

| Done | Item | Category impact |
|---|---|---|
| ✅ | 2 Gitleaks | Security partial |
| ✅ | 3 Bundle compat matrix | Maintainability 8→10 |
| ✅ | 4 Global patch docs | Maintainability 8→10 |
| ✅ | 5a pytest-xdist | Performance 6→7 |
| ✅ | 6 DAG validation | Architecture 9→10 |
| ✅ | 8 CI parity test | CI/CD 9→10 |
| ✅ | 9 New bundle tutorial | Documentation 9→10 |
| ⏳ | 1 Bandit CI gate | Security 9→10 |
| ⏳ | 5b Marimo timeout | Performance 7→8 |
| ⏳ | 7 Mutation testing | Code Quality 9→10 |

Current score: **(10+9+9+10+10+9+9+9+10+7+9)/11 = 101/11 ≈ 9.2**  
Remaining 3 items deliver +3 points → 104/11 ≈ **9.45**, rounding to **9.5**.
