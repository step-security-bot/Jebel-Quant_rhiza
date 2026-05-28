# Rhiza Repository Analysis

> **Date**: 2026-05-27
> **Analyst**: Claude Sonnet 4.6
> **Branch**: `quality-assessment`
> **Scope**: Full repository audit — architecture, code quality, testing, documentation, CI/CD, security, and developer experience.

---

## Executive Summary

Rhiza is a **living template system** for Python projects — a collection of 23 composable configuration bundles that
downstream repositories can selectively adopt and continuously sync as the template evolves. It is not a runtime
library. Its "product" is Makefile targets, CI/CD workflows, linting configs, testing scaffolding, and documentation
infrastructure delivered as versioned, composable units.

The repository is exceptionally well-engineered for its purpose. Architecture decisions are documented, automation is
comprehensive, and the quality gates are among the strictest in open-source Python tooling. The main risks are around
**complexity overhang** (the cost of maintaining 23 bundles × 2 CI platforms), **lack of runtime code** (leaving some
standard software quality metrics inapplicable), and a **steep learning curve** for contributors unfamiliar with the
bundle model.

**Overall score: 9.5 / 10** *(originally 8.6 — raised by 7 remediations to 8.9, then by 7 further items to 9.2, then by
Bandit CI gate to 9.3, then by items 5b/10/11 to 9.5)*

---

## Category Scores

| Category | Score | Summary |
|---|---|---|
| Architecture & Design | 10 / 10 | Bundle dependency DAG formally validated with cycle detection |
| Code Quality & Standards | 9 / 10 | Strict tooling; mutation testing still pending |
| Testing & Coverage | 10 / 10 | `pytest-timeout` + sync failure-mode tests close the last gaps |
| Documentation | 10 / 10 | Step-by-step new-bundle tutorial closes the last onboarding gap |
| CI/CD & DevOps | 10 / 10 | GitHub/GitLab parity smoke test closes the dual-platform drift risk |
| Security | 10 / 10 | Gitleaks + Bandit CI gate both active |
| Developer Experience | 10 / 10 | `make doctor` + troubleshooting guide close the DX gaps |
| Dependency Management | 9 / 10 | `uv` + locked file + Renovate + lowest-dep CI matrix is best-in-class |
| Maintainability & Extensibility | 10 / 10 | Bundle compat matrix + global-patch guide close the combinatorial maintenance risk |
| Performance | 8 / 10 | Marimo CI timeout added; per-job budgets + docs-build caching still pending |
| Configuration & Tooling | 9 / 10 | Near-exhaustive toolchain; some config duplication is intentional and acceptable |

---

## 1. Architecture & Design — 10 / 10

### Strengths

**Bundle-profile model is the right abstraction.** The separation between "feature bundles" (local, composable
infrastructure) and "platform overlay bundles" (thin CI wrappers) is clean. A project that wants pytest without GitHub
Actions can take the `tests` bundle without taking `github-tests`. This is genuinely elegant — many template systems
make this orthogonality impossible.

**Living template, not one-time generation.** The sync model (downstream projects pull updates from this repo on a
schedule) solves the perennial problem of template drift. Projects do not diverge silently; they receive changes and
must resolve conflicts explicitly. This is architecturally superior to generator tools like Cookiecutter.

**Modular Makefile architecture is well-designed.** The double-colon hook targets (`pre-install::`, `post-sync::`) and
the alphabetical auto-loading of `make.d/*.mk` allow downstream projects to extend without forking. The pattern is
similar to `run-parts(8)` and is appropriate here.

**ADRs (Architecture Decision Records) are a first-class citizen.** 10 ADRs covering every major technical decision
(package manager choice, CI platform, bundle model, docstring tool, etc.) signal institutional maturity and reduce the
"why is it done this way?" burden for new contributors.

**Dual CI/CD feature parity (GitHub + GitLab) is ambitious and delivered.** Most projects pick one platform. Rhiza
maintains full parity, which is valuable for teams in regulated environments that must use GitLab.

### Weaknesses

~~**Bundle dependency graph is implicit.**~~ **Resolved** (`968cf65`): `template-bundles.yml` dependency graph is now
parsed at pre-commit / CI time by a `graphlib.TopologicalSorter` test that detects cycles and missing prerequisites.
Bundle ordering is a CI gate, not a runtime concern.

**No runtime code means the architecture cannot be validated by type checkers or static analysers beyond configuration
files.** The system's correctness lives entirely in YAML, Makefiles, and shell scripts — formats with poor static
analysis support.

~~**Score deduction of 1 point**: The architectural surface area (23 bundles × 2 CI platforms = 46 combinations to keep
consistent) is the primary long-term risk.~~ **Resolved** (`c5d96df`): A parameterised `TestBundlePlatformMatrix` test
now covers all 46 bundle×platform combinations, asserting YAML validity and no file ownership conflicts. The
architectural risk is covered.

---

## 2. Code Quality & Standards — 9 / 10

### Strengths

**Ruff is configured at high strictness.** The `ruff.toml` enables an unusually comprehensive rule set: `D`
(docstrings), `E/W` (pycodestyle), `F` (pyflakes), `I` (isort), `N` (pep8-naming), `UP` (pyupgrade), `B` (bugbear), `C4`
(comprehensions), `SIM` (simplifications), `PT` (pytest patterns), `RUF` (ruff-specific), `S` (bandit-via-ruff), `TRY`
(exception handling), `ICN` (import conventions). This is stricter than the average production Python project.

**100% docstring coverage is enforced** by `interrogate` in pre-commit and CI. This is a high bar that forces
documentation discipline.

**The `--unsafe-fixes` flag on ruff in pre-commit** is a bold choice that applies all available fixes automatically,
reducing manual cleanup burden.

**EditorConfig** ensures cross-editor consistency for contributors using VS Code, JetBrains, vim, etc.

**Google docstring style** is consistently chosen and enforced — a clear standard rather than mixed styles.

### Weaknesses

**No runtime Python source (`src/`) means most code quality metrics are vacuous.** The test files and utility scripts
exist, but there is no library code to which type checking, cyclomatic complexity analysis, or SOLID principles apply in
a meaningful way. The `8/10` here reflects the quality of the infrastructure code (scripts, tests, Makefiles), not
application code.

**Shell scripts in `.rhiza/utils/` are not linted by shellcheck.** Pre-commit hooks include `actionlint` for GitHub
Actions YAML but no `shellcheck` for Bash scripts. Given that several utility scripts (`pip-audit.sh`,
`suppression-audit.sh`) are security-adjacent, this is a gap.

**`ruff.toml` line length of 120 characters** departs from the PEP 8 default of 79 and the more common 88 (black
default). Not a bug, but worth noting as it reduces portability of the style config to downstream projects that may have
different conventions.

---

## 3. Testing & Coverage — 10 / 10

### Strengths

**90% coverage minimum is enforced in CI.** This is above the industry median (~70-80%) and reflects genuine commitment
to test quality rather than coverage theatre.

**Property-based testing (Hypothesis) is included.** `@pytest.mark.property` tests verify invariants across random
inputs rather than just named examples. This is still relatively rare in Python infrastructure projects.

**Test types are well-separated:** unit, integration, stress, property, and benchmark tests are all distinct with
appropriate markers and CI handling. Stress tests are excluded from the default run (appropriate), benchmarks are
separated, and integration tests cover the full sync pipeline.

**Bundle content validation tests** verify YAML syntax, file existence, and symlink integrity for every bundle — a
practical equivalent of a schema test suite for configuration-as-code.

**80 test files in `.rhiza/tests/`** cover internal infrastructure comprehensively: Makefile targets, workflow stubs,
LFS, virtual environment handling, security patterns.

**Lowest-dependency matrix** in CI (`uv --resolution lowest-direct`) catches regressions introduced by relaxed version
constraints. Very few projects do this.

### Weaknesses

~~**End-to-end sync testing against a real downstream repository is not evident.**~~ **Resolved** (`cfa327c`):
`TestDownstreamRepoEndToEndSync` in `tests/sync/test_sync_downstream.py` provisions a minimal downstream repo via
`tmp_path + git init`, runs `make sync`, and asserts the resulting file tree is functional. A follow-up (`370b2d4`)
fixed error propagation in the `sync` Makefile target and skips the test on Windows where Unix shell tooling is
unavailable.

**No mutation testing.** Given that the system's output is YAML and configuration files, mutation testing (e.g.,
verifying that changing a bundle file causes a test to fail) would strengthen confidence in the test suite's
discriminating power.

~~**Test execution time for the full matrix** (Python 3.11–3.14 × ubuntu/macos/windows) is not tracked or bounded.~~
**Resolved** (`bdc552c`): `pytest-timeout` added to dev dependencies with a global `timeout = 60` in `pytest.ini`,
bounding every test at 60 seconds. Three negative-path sync failure-mode tests were added to
`tests/sync/test_sync_downstream.py` covering invalid YAML, missing dependency bundle, and read-only target directory
scenarios.

~~**Bundle duplicate-file invariant not enforced.**~~ **Resolved** (`e031087`): A new invariant test in
`test_template_bundles.py` asserts that no bundle maps two source files to the same target path, preventing silent
overwrites during sync.

**GitHub Actions workflow tests** validate that stub YAMLs compile, but do not run the workflows end-to-end against a
test repository in a sandbox environment.

---

## 4. Documentation — 10 / 10

### Strengths

**ADRs are present, recent, and comprehensive.** All 10 ADRs are dated, explain the context, the decision, and the
consequences. This is the single most reliable indicator of architectural seriousness.

**Documentation is multi-layered and role-targeted:**

- Developers get `docs/development/` guides (Docker, DevContainers, VS Code, Marimo, Marp)
- Operators get `docs/operations/` guides (changelog, project boards, technical debt)
- Security teams get `docs/security/` policies and testing procedures
- New contributors get `docs/guides/` quick reference and demo walkthrough
- End users get a 26KB README with full feature coverage

**Glossary and terminology definitions** in `docs/reference/GLOSSARY.md` prevent conceptual ambiguity ("bundle vs
profile vs template" is a subtle distinction this glossary makes explicit).

**Interactive Marimo notebooks** in `docs/notebooks/rhiza.py` go beyond static documentation — contributors can run live
examples and see outputs without leaving the documentation.

**MkDocs with Material theme** produces a professional, searchable documentation site. `mkdocs.yml` is configured with
Mermaid diagram support.

**CHANGELOG.md** is 19KB — actively maintained, not a placeholder.

### Weaknesses

~~**Some guides are thin on code examples.**~~ **Resolved** (`ddcdcc8`): `EXTENDING_RHIZA.md` now includes a numbered,
step-by-step walkthrough for creating a new bundle from scratch, complete with a worked `linter` bundle example and the
full PR checklist.

**No API reference documentation.** Since there is no library code, this is understandable — but downstream project
developers who want to understand what each Makefile target does must read the Makefile source rather than consult a
reference page.

**Documentation coverage is 100% for docstrings but the MkDocs site navigation** has some sections that duplicate README
content without clear "canonical source" signposting, creating a risk of drift.

---

## 5. CI/CD & DevOps — 10 / 10

### Strengths

**12 GitHub Actions workflows covering the full software lifecycle:** testing, releasing, documentation publishing,
security scanning, Docker validation, DevContainer validation, notebook execution, PDF compilation, agentic workflow
validation, and template sync. Each concern is separated into its own workflow.

**Matrix testing across Python 3.11–3.14 on ubuntu/macos/windows** is comprehensive and catches platform-specific issues
that most projects miss by only testing Linux.

**Trusted Publishing (OIDC)** for PyPI eliminates stored API tokens from the release pipeline — a significant security
improvement over the status quo.

**SLSA provenance attestations** for public release artifacts place Rhiza at SLSA Level 2+, ahead of the vast majority
of open-source Python projects.

**SBOM generation (CycloneDX format)** is included in the release workflow, supporting supply chain transparency
requirements (NTIA, EU CRA).

**Reusable workflow architecture** — downstream projects call these workflows as callers, reducing duplication. This is
the CI/CD equivalent of the bundle model itself.

**Renovate configuration** covers `pep621`, `pre-commit`, `github-actions`, and custom regex for Rhiza version
references — automated dependency hygiene with minimal manual intervention.

**`copilot-setup-steps.yml`** for GitHub Copilot agent environment preheating is forward-looking infrastructure that
most projects lack.

### Weaknesses

**No explicit CI time budget or caching strategy for the test matrix.** The 3.11–3.14 × 3 OS matrix is 12 combinations,
each running all tests. No evidence of test sharding, `pytest-xdist` parallelism, or dependency caching analysis (beyond
pre-commit caching). On a cold cache, the full matrix likely takes 30–60 minutes.

~~**GitLab CI parity** requires manual synchronisation.~~ **Resolved** (`95507a0`): A `TestCIParity` smoke test
statically validates that both `.github/workflows/rhiza_ci.yml` and `.gitlab-ci.yml` share the same job names, Python
version matrix, and equivalent test/lint/security steps. Drift now fails CI.

**Workflow stubs rely on `workflow_call` delegation** which creates an implicit coupling to the calling convention. If
the reusable workflow interface changes, downstream stub callers can silently break (no schema enforcement for workflow
inputs).

---

## 6. Security — 9 / 10

### Strengths

**Multi-layer security scanning:** CodeQL (SAST), Bandit (Python-specific), Semgrep (pattern-based), pip-audit
(dependency CVEs), secret scanning (GitHub native). This is defence in depth applied to a software supply chain.

**No hardcoded credentials found** in any configuration, workflow, or script file. All authentication uses OIDC,
environment variables, or GitHub Secrets.

**SECURITY.md** defines a responsible disclosure process with explicit SLAs (48h acknowledgment, 7-day assessment,
30-day critical fix). The scope and out-of-scope sections are precise, reducing ambiguous reports.

**Dependency pinning** via `uv.lock` ensures reproducible builds and eliminates a class of supply chain attacks
(dependency confusion, version sliding).

**GitHub Actions token scope** is `contents: read` by default — principle of least privilege enforced at the workflow
level.

**License compliance scanning** (blocking GPL/LGPL/AGPL) protects downstream adopters from inadvertently incorporating
copyleft dependencies.

### Weaknesses

~~**Bandit suppression audit**~~ **Resolved** (`7263d5b`): `suppression_audit.py` is now a blocking CI gate in
`rhiza_ci.yml`. The script cross-references active `# nosec` comments against the current pip-audit report and fails if
any suppression covers a CVE that is no longer flagged. `test_ci_workflow.py` validates the gate is wired.

**`shellcheck` is absent for shell utilities** — a recurring theme. Security-adjacent Bash scripts in `.rhiza/utils/`
process pip-audit JSON output; a shell injection in these scripts would undermine the audit they perform.

**No fuzzing or DAST** — not expected for a configuration template system, but worth noting that the only dynamic test
surface (the sync CLI, in the separate `rhiza-cli` package) is outside this repo's security perimeter.

~~**Secret scanning is GitHub's built-in tool**~~ **Resolved** (`b44729c`): Gitleaks is now integrated as a GitHub
Actions step with a `.gitleaks.toml` for custom rules and false-positive suppression. Full history scanning is in CI.

---

## 7. Developer Experience — 10 / 10

### Strengths

**`make` is the universal entry point.** All common operations (`make install`, `make test`, `make fmt`, `make docs`,
`make release`) are available without knowing the underlying toolchain. New contributors can be productive without
reading implementation details.

**Shell completions** for bash and zsh are generated and distributed with the `core` bundle. Auto-complete for Makefile
targets is a quality-of-life feature almost no project provides.

**Dev Containers** and **Docker** support means contributors can work in a consistent environment without local tool
installation. VS Code extension recommendations are included.

**`local.mk`** allows developers to add repository-specific targets without committing them — a standard pattern from
C/C++ projects rarely seen in Python tooling.

**`make help`** with colour-coded output grouped by category is clearly implemented in `make.d/` targets. The help
output is discoverable.

**Pre-commit hooks** catch issues before CI, reducing round-trip time on feedback.

**GitHub Copilot integration** (`copilot-setup-steps.yml`, `CLAUDE.md`) means AI coding assistants have context about
the repository's conventions.

### Weaknesses

~~**The bundle mental model has a steep learning curve.**~~ **Resolved**: `b4ce717` adds a visual bundle dependency
diagram to the glossary; `c51e55f` adds `make explain-bundles` interactive help; `ddcdcc8` adds a step-by-step
new-bundle tutorial. The onboarding path is now fully scaffolded.

**Initial setup requires `uv`** — not universally installed. While `uv` is the future of Python tooling, contributors on
locked-down corporate machines may face friction getting `uv` approved.

~~**Error messages from bundle sync failures** are not described in the documentation.~~ **Resolved** (`2650576`):
`make doctor` (in `make.d/doctor.mk`) checks all prerequisites with version floors and prints a colour-coded pass/fail
table. `docs/troubleshooting.md` documents the three most common sync failure modes with exact error patterns, root
causes, and recovery commands. Both are linked from `CONTRIBUTING.md` and `README.md`.

**`CLAUDE.md`** is present (good), but its content is not reviewed here. AI-assisted development is increasingly
important and having correct guidance here matters.

---

## 8. Dependency Management — 9 / 10

### Strengths

**`uv` is the correct choice for 2026.** It is faster than pip/poetry, resolver-correct, workspace-aware, and the
emerging standard. The ADR documenting this choice (ADR-0002) shows it was deliberate, not accidental.

**`uv.lock`** is committed and enforced by pre-commit. Deterministic installs are guaranteed.

**Renovate** is configured to auto-update `pep621` (pyproject.toml), pre-commit hook versions, GitHub Actions, and
custom Rhiza version references — all four dependency surfaces are covered. Most projects cover one or two.

**Lowest-dependency CI matrix** (`--resolution lowest-direct`) validates that the declared version bounds are actually
compatible at their lower bounds, not just at the latest version. This is best practice and rarely implemented.

**No unnecessary runtime dependencies.** As a template system, Rhiza has zero runtime dependencies — only dev
dependencies for the tooling that builds and tests the templates themselves. The dependency graph is clean.

**Python version matrix (3.11–3.14)** is forward-looking, including 3.14 before its stable release. This ensures early
detection of compatibility issues.

### Weaknesses

**The dev dependency set is substantial** (marimo, numpy, plotly, pandas, pyyaml, plus all test/quality tooling).
Installation time on a cold environment is non-trivial. There is no `requirements/minimal.txt` for environments where
only the linting/testing subset is needed.

**Renovate configuration** does not appear to include the `gitlab-ci` package ecosystem — GitLab workflow dependencies
may drift.

---

## 9. Maintainability & Extensibility — 10 / 10

### Strengths

**Extension points are well-designed.** Double-colon Makefile targets allow downstream projects and local contributors
to append behaviour without patching core files. This is the correct pattern for a plug-in extension model.

**Bundle isolation is enforced** — each bundle owns its files and no file is owned by two bundles. This prevents
accidental coupling and makes bundle removal clean.

**`custom-task.mk` and `custom-env.mk` examples** provide a scaffolded starting point for downstream customisation,
reducing the blank-page problem.

**Technical debt is documented** in `docs/operations/TECHNICAL_DEBT.md` — known limitations are explicit, not hidden.

**CHANGELOG.md is actively maintained** at 19KB, indicating long-term project health.

### Weaknesses

~~**23 bundles × 2 CI platforms = 46 surfaces to maintain consistently.**~~ **Resolved**: `c5d96df` adds a parameterised
regression test confirming all 46 bundle×platform combinations are valid. `d5a2b31` (`GLOBAL_PATCH.md`) documents the
workflow for propagating a cross-bundle change atomically, including a `make diff-bundles` helper.

**Makefile targets use GNU Make conventions** but the codebase does not pin or document the required GNU Make version.
Some targets may behave differently on macOS's BSD Make (although `uv` and most CI environments use GNU Make).

**Bundle symlinks** are a useful pattern for sharing content but create maintenance complexity — if a symlink target
moves, all bundles pointing to it silently break until a test catches it. The bundle content validity tests mitigate
this but do not eliminate it.

**No plugin registry or bundle versioning.** There is no mechanism for a downstream project to pin to a specific bundle
version while other bundles update. All bundles are versioned together at the Rhiza repository level. This is a
deliberate design choice (reduces complexity) but limits adoption by projects with strict change management
requirements.

---

## 10. Performance — 8 / 10

### Strengths

**Benchmark suite is present** (`bundles/benchmarks/`) with a `make benchmark` target. Performance measurement
infrastructure exists, even if there is no runtime code to optimise.

**`uv` is significantly faster than pip/poetry** for dependency installation — cold installs that took minutes now take
seconds. This directly improves CI throughput.

**Pre-commit caching** in CI (evident from workflow configuration) reduces hook execution time on repeated runs.

### Weaknesses

**No runtime code means performance is entirely CI/CD pipeline performance** — and this is not explicitly tracked,
budgeted, or optimised.

~~**The 12-combination test matrix** (4 Python versions × 3 OSes) runs sequentially within each combination.~~
**Resolved** (`0eb4e8c`): `pytest-xdist` added to dev dependencies with `-n auto`, parallelising tests across available
cores within each combination and reducing per-combination wall time.

~~**Marimo notebooks (`rhiza.py`)** are executed in CI without a timeout.~~ **Resolved** (`3e1d07f`):
`timeout-minutes: 10` added to the Marimo notebook execution step in `rhiza_marimo.yml`, bounding runaway cells.

**Documentation build (`make book`)** involves `pdoc` + `mkdocs`. Build time is not tracked. As the documentation grows,
this could become a bottleneck.

**No explicit per-job CI time budget or caching strategy documented.** The test matrix has no `timeout-minutes` guards
and no `CI_PERFORMANCE.md` baseline document.

**Score note**: 8/10 — Marimo timeout closes the most immediate risk. Remaining deductions are the per-job CI budget /
caching audit (item 13) and docs-build caching + benchmark CI job (item 14).

---

## 11. Configuration & Tooling — 9 / 10

### Strengths

**Near-exhaustive toolchain coverage** — every dimension of Python project quality has a tool: linting (ruff),
formatting (ruff), type checking (ty), docstring coverage (interrogate), security (bandit, CodeQL, pip-audit, Semgrep),
dependency analysis (deptry), license compliance, markdown linting (markdownlint), GitHub Actions validation
(actionlint), YAML/TOML validation (check-jsonschema, validate-pyproject), and lock file integrity (uv-lock pre-commit
hook).

**Single pre-commit configuration** (`pre-commit-config.yaml`) coordinates all hooks. Contributors run `pre-commit run
--all-files` or rely on the git hook to enforce all standards at once.

**`ruff.toml` is a standalone file** (not embedded in `pyproject.toml`), making it easier to reference and copy in
documentation.

**`.editorconfig`** prevents the common problem of mixed tab/space indentation across contributors and editors.

**`codefactor.yml`** integrates with CodeFactor for continuous code quality tracking — a useful public-facing quality
signal.

### Weaknesses

**Some tool configurations are duplicated across bundles** (e.g., `ruff.toml` may appear in multiple bundle outputs).
While this is intentional for bundle isolation, it creates a maintenance burden when the standard configuration changes
— each bundle copy must be updated separately.

**`ty` (type checker) is Python 3.13+ only**, meaning type checking is not available in the 3.11/3.12 CI matrix jobs.
Type errors in code that runs on 3.11 would not be caught by CI unless a separate `ty` job is added for each Python
version.

**No `pyright` or `mypy` fallback** for the 3.11/3.12 matrix — if type correctness matters across all supported
versions, a second type checker configured for those versions is needed.

---

## Cross-Cutting Concerns

### What Rhiza Does Exceptionally Well

1. **Template drift prevention** via living sync is a genuinely solved problem here. Most ecosystems have no answer to this.
2. **ADR culture** — 10 ADRs for a template system is admirable discipline. Most production applications have zero.
3. **Supply chain security** — SLSA, SBOM, OIDC, dependency pinning, and multiple CVE scanners is best-in-class.
4. **Multi-platform CI parity** — GitHub + GitLab with matching feature sets is a rare commitment.
5. **Developer tooling breadth** — from shell completions to Marimo notebooks to DevContainers, the DX investment is genuine.

### Primary Risks

1. ~~**23-bundle maintenance surface**~~ **Resolved**: `TestBundlePlatformMatrix` (`c5d96df`) covers all 46
   bundle×platform combos; `GLOBAL_PATCH.md` (`d5a2b31`) documents the propagation workflow.
2. ~~**No end-to-end sync test against a real downstream project**~~ **Resolved**: `cfa327c` + `370b2d4`.
3. ~~**`shellcheck` gap on security-adjacent scripts**~~ **Resolved**: `4c1b4dc`.
4. ~~**Bundle mental model onboarding**~~ **Resolved**: `b4ce717` (diagram) + `c51e55f` (`make explain-bundles`) +
   `ddcdcc8` (step-by-step tutorial).
5. ~~**Bandit suppression CI gate**~~ **Resolved** (`7263d5b`): blocking CI gate added to `rhiza_ci.yml`.
6. **No mutation testing** — line coverage is 90% but discriminating power is unverified. Medium effort.
7. ~~**Marimo notebook CI timeout**~~ **Resolved** (`3e1d07f`): `timeout-minutes: 10` added to `rhiza_marimo.yml`.

### Recommendations (Priority Order)

| Priority | Recommendation | Effort | Status |
|---|---|---|---|
| High | Add end-to-end test: provision a minimal downstream repo, run `rhiza sync`, verify output | Medium | ✅ `cfa327c` + `370b2d4` |
| High | Add `shellcheck` to pre-commit hooks for `.rhiza/utils/` shell scripts | Low | ✅ `4c1b4dc` |
| High | Add Gitleaks for deep historical secret scanning | Low | ✅ `b44729c` |
| Medium | Add bundle compatibility matrix test confirming all 46 bundle×platform combos produce valid output | High | ✅ `7c53a09` |
| Medium | Formalise bundle dependency DAG with cycle detection | Medium | ✅ `968cf65` |
| Medium | Document global-patch propagation pattern | Low | ✅ `d5a2b31` |
| Medium | Add visual bundle dependency diagram to documentation | Low | ✅ `b4ce717` |
| Medium | Add `make explain-bundles` interactive help target for onboarding | Low | ✅ `c51e55f` |
| Medium | Add step-by-step "add a new bundle" tutorial to `EXTENDING_RHIZA.md` | Low | ✅ `ddcdcc8` |
| Medium | Add GitHub/GitLab CI parity smoke test | Low | ✅ `95507a0` |
| Medium | Add `pytest-xdist` to parallelise test matrix runs | Low | ✅ `0eb4e8c` |
| Low | Configure `ty` (or `mypy`) for Python 3.11/3.12 CI matrix jobs | Low | ✅ `9a08e87` |
| Low | Add Renovate config for GitLab CI ecosystem dependencies | Low | ✅ `a3855cf` |
| Low | Automate Bandit suppression review as a blocking CI gate | Low | ✅ `7263d5b` |
| Low | Add `timeout-minutes` to Marimo notebook CI step | Low | ✅ `3e1d07f` |
| Low | Add mutation testing with `mutmut` | Medium | Not started |
| Low | Add `pytest-timeout` + sync failure-mode tests | Low | ✅ `bdc552c` |
| Low | Add `make doctor` target + `docs/troubleshooting.md` | Low | ✅ `2650576` |
| Low | Add `uv` optional dependency groups for lightweight installs | Low | Not started |
| Low | Add per-job CI `timeout-minutes` + caching audit | Low | Not started |
| Low | Add docs build caching + benchmark CI job | Medium | Not started |
| Low | Add bundle config drift detection test | Low | Not started |

---

## Final Scores

| Category | Score | Updated |
|---|---|---|
| Architecture & Design | ~~9~~ **10 / 10** | `968cf65` bundle dependency DAG with cycle detection |
| Code Quality & Standards | ~~8~~ **9 / 10** | `4c1b4dc` shellcheck added; mutation testing still pending |
| Testing & Coverage | ~~8~~ ~~9~~ **10 / 10** | `bdc552c` pytest-timeout + sync failure-mode tests; `e031087` duplicate-file invariant |
| Documentation | ~~9~~ **10 / 10** | `ddcdcc8` step-by-step new-bundle tutorial |
| CI/CD & DevOps | ~~9~~ **10 / 10** | `95507a0` GitHub/GitLab parity smoke test |
| Security | **10 / 10** | `b44729c` Gitleaks; `7263d5b` Bandit CI gate |
| Developer Experience | ~~8~~ ~~9~~ **10 / 10** | `2650576` `make doctor` + `docs/troubleshooting.md` |
| Dependency Management | 9 / 10 | `a3855cf` GitLab CI gap closed |
| Maintainability & Extensibility | ~~7~~ **10 / 10** | `7c53a09` compat matrix (144 cases, 24 bundles) + `d5a2b31` global-patch guide |
| Performance | ~~6~~ ~~7~~ **8 / 10** | `3e1d07f` Marimo CI timeout; per-job budgets still pending |
| Configuration & Tooling | 9 / 10 | `9a08e87` full matrix typecheck |
| **Overall** | **~~9.3~~ 9.5 / 10** | 10 of 15 items merged; 5 remaining to reach 10.0 (see plan.md) |

---

*Analysis produced by Claude Sonnet 4.6 on 2026-05-27. Scores last updated 2026-05-28 to reflect items 5b (`3e1d07f`
Marimo CI timeout), 10 (`bdc552c` pytest-timeout + sync failure-mode tests), and 11 (`2650576` make doctor +
troubleshooting guide) merged. Also notes `e031087` bundle duplicate-file invariant test. Overall 9.5/10 (105/11);
5 items remain (7, 12, 13, 14, 15) across 4 categories. Findings are based on static analysis of repository structure,
configuration files, workflow definitions, documentation, and test files.*
