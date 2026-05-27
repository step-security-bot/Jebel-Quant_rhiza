# Plan: 8.6 → 9.0

> **Goal**: Raise overall score from 8.6 to 9.0 by fixing the cheapest deductions first.  
> **Non-goal**: Perfect scores — stop once the target is hit.  
> **Last updated**: 2026-05-27  
> **Progress**: 7 / 7 items complete ✅

---

## Score gap

| Category | Now | Target | Delta | Status |
|---|---|---|---|---|
| Code Quality & Standards | 8 | 9 | +1 | Done — `4c1b4dc` |
| Testing & Coverage | 8 | 9 | +1 | Done — `cfa327c`, `370b2d4` |
| Developer Experience | 8 | 9 | +1 | Done — `b4ce717`, `c51e55f` |
| Maintainability & Extensibility | 7 | 8 | +1 | Done — `333bada` |
| Performance | 6 | 6 | — (skip — inherent to template systems) | — |
| All others | 9 | 9 | — | — |

Four categories need +1 each. All have low-effort fixes available.

---

## Items (ordered by effort)

### 1. Add `shellcheck` to pre-commit — 30 min ✅ `4c1b4dc`
**Fixes**: Code Quality 8→9, Security weakness  
Add `shellcheck` as a pre-commit hook scoped to `.rhiza/utils/*.sh`. Fix any findings (expected to be minor — these are short audit scripts).

```yaml
# .pre-commit-config.yaml
- repo: https://github.com/shellcheck-py/shellcheck-py
  rev: v0.10.0.1
  hooks:
    - id: shellcheck
      files: \.rhiza/utils/.*\.sh$
```

---

### 2. Renovate: add GitLab CI ecosystem — 15 min ✅ `a3855cf`
**Fixes**: Dependency Management weakness (currently 9, keeps it there; removes a called-out gap)  
Add `"gitlabci"` to the Renovate `packageRules` or `matchManagers` list in `renovate.json` / `.github/renovate.json`.

---

### 3. Bundle dependency diagram — 1 h ✅ `b4ce717`
**Fixes**: Developer Experience 8→9 (onboarding friction)  
Add a Mermaid diagram to `docs/reference/GLOSSARY.md` or a new `docs/reference/bundle-map.md` showing which bundles depend on which. MkDocs already supports Mermaid — zero infra cost.

---

### 4. `make explain-bundles` target — 1 h ✅ `c51e55f`
**Fixes**: Developer Experience 8→9 (complements the diagram)  
Add a target to `make.d/` that prints each bundle name, its description from `template-bundles.yml`, and its direct dependencies. Parses `template-bundles.yml` with `python -c` or `yq` — no new dependencies.

---

### 5. Document GNU Make version requirement — 15 min ✅ `333bada`
**Fixes**: Maintainability 7→8 (specific weakness called out)  
Add the required GNU Make version to `README.md` prerequisites table and `docs/development/` setup guide. Run `make --version` in CI and assert `>= 4.x` if desired (one-liner).

---

### 6. `ty` on Python 3.11/3.12 matrix jobs — 2 h ✅ `9a08e87`
**Fixes**: Configuration & Tooling weakness (keeps 9, removes a called-out gap); supports Code Quality 8→9  
`ty` is 3.13+ only. Add `mypy` (or `pyright`) as a second type-checking step in the 3.11/3.12 CI matrix jobs. Alternatively, run `ty` under 3.13 only and add a comment in `pyproject.toml` scoping this explicitly — documents the limitation rather than leaving it implicit.

---

### 7. End-to-end sync test — half day ✅ `cfa327c` + `370b2d4`
**Fixes**: Testing & Coverage 8→9 (highest-priority gap per analysis)  
Added `TestDownstreamRepoEndToEndSync` in `tests/sync/test_sync_downstream.py`: provisions a minimal downstream repo in `tmp_path` with a `git init`, runs `make sync`, and asserts `pytest.ini`, `.rhiza/tests/conftest.py`, `.rhiza/make.d/test.mk`, and key file contents are present. A follow-up commit fixed the Windows CI failure (silent error swallow in the `sync` Makefile target via `&&` instead of `;`) and skipped the test on Windows where `make sync` requires Unix shell tooling.

---

## What to skip

| Item | Why skip |
|---|---|
| Bundle compatibility matrix (46 combos) | High effort, already partially covered by existing bundle content tests |
| Stale Bandit suppression review | Operational task, not a code change; do it as part of a normal sprint |
| Plugin registry / bundle versioning | Architectural change, out of scope |
| CI time budget / test sharding | No evidence it is actually slow; premature optimisation |

---

## Result

All 7 items complete. Testing & Coverage raised from 8 → 9, bringing the overall score to **8.9 / 10**. The remaining gap to 9.0 is the Performance category (6/10), which is inherent to a template system with no runtime code and not worth closing.

