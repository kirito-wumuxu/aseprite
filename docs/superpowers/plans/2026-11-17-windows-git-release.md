# Windows Git Release Automation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create and land an automated GitHub Actions release workflow for Windows x64 Aseprite, package portable zip and sha256 checksums, and publish release under tag `v1.3.18.6` based on commit `331e898`.

**Architecture:** Build a dedicated `.github/workflows/release-windows.yml` pipeline that triggers on `v*` tag push or `workflow_dispatch`. It sets up MSVC x64, Ninja, and dynamically pulled official prebuilt Skia, builds Aseprite in `RelWithDebInfo` mode without tests, packages the binary and data assets with PowerShell, generates SHA256 checksum, and publishes via Runner-native GitHub CLI `gh release`. Land the workflow on baseline commit `331e898` with tag `v1.3.18.6` and retain it on `main`.

**Tech Stack:** GitHub Actions, CMake, Ninja, MSVC (x64), PowerShell, Bash, GitHub CLI (`gh`).

**Spec:** [docs/superpowers/specs/2026-11-17-windows-git-release-design.md](file:///home/wmx/workspace/repository/aseprite/docs/superpowers/specs/2026-11-17-windows-git-release-design.md)

## Global Constraints

- Target Tag: `v1.3.18.6`
- Target Baseline Commit: `331e898ef405d919539c03fd1675691cfa16c3a3`
- Operating System Runner: `windows-latest`
- Package Name Format: `Aseprite-${TAG}-windows-x64.zip`
- Checksum Format: Standard SHA256 checksum file `Aseprite-${TAG}-windows-x64.zip.sha256` (`<hash>  <filename>`)
- Publishing Mechanism: Runner native `gh release` (no untrusted third-party release actions)
- Workflow Permissions: `contents: write`

## Review Focus

1. Release Tag resolution: Handling both `refs/tags/v*` triggers and manual `workflow_dispatch` input tags without empty string errors.
2. Asset upload collision: Uploading assets to an already-created release (draft/existing) without crashing (`--clobber` support).
3. Data asset packaging: Ensuring recursive `data/` directory and required legal/documentation files (`EULA.txt`, `README.md`, `docs/LICENSES.md`) exist and are faithfully included.
4. Skia dependency resolution: Verifying `laf/misc/skia-url.sh` generates a valid URL and unzips properly on Windows runners.
5. Git ref alignment: Confirming tag `v1.3.18.6` points to a commit containing both the workflow file and commit `331e898`'s codebase state.

---

### Task 1: Create GitHub Actions Windows Release Workflow

**Files:**
- Create: `.github/workflows/release-windows.yml`

**Interfaces:**
- Consumes: `laf/misc/skia-url.sh`, `EULA.txt`, `README.md`, `docs/LICENSES.md`, CMake configuration.
- Produces: `.github/workflows/release-windows.yml` runnable workflow.

- [ ] **Step 1: Write `.github/workflows/release-windows.yml`**

Implement the complete workflow file containing:
- Triggers: `push: tags: ['v*']` and `workflow_dispatch` with `tag_name`, `is_draft`, `is_prerelease` inputs.
- Permissions: `contents: write`.
- Step 1: Checkout with `submodules: recursive`.
- Step 2: Resolve release tag and environment variables (`RELEASE_TAG`, `ZIP_FILE`, `SHA_FILE`).
- Step 3: Install Ninja via `aseprite/get-ninja@main`.
- Step 4: Setup MSVC via `TheMrMilchmann/setup-msvc-dev@v4` with `arch: x64`.
- Step 5: Dynamically download and unpack Skia using `laf/misc/skia-url.sh`.
- Step 6: CMake configure with `ENABLE_TESTS=OFF`, `ENABLE_SCRIPTING=ON`, `ENABLE_CCACHE=OFF`, `LAF_BACKEND=skia`.
- Step 7: Build `aseprite` target using Ninja.
- Step 8: Package `aseprite.exe`, `data/`, `EULA.txt`, `README.md`, and `docs/LICENSES.md` into ZIP and generate SHA256 checksum using PowerShell.
- Step 9: Publish or update release with `gh release create` / `gh release upload --clobber`.

- [ ] **Step 2: Verify YAML syntax using Python YAML parser**

Run:
```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/release-windows.yml'))"
```
Expected: Exit code 0, no YAML parsing errors.

- [ ] **Step 3: Commit workflow to `main`**

```bash
git add .github/workflows/release-windows.yml
git commit -m "ci: add windows git release workflow"
```

---

### Task 2: Validate Packaging Paths and Workflow Integrity

**Files:**
- Test / Verification script: `tests/verify-release-workflow.py`

**Interfaces:**
- Consumes: `.github/workflows/release-windows.yml`, repo layout (`EULA.txt`, `README.md`, `docs/LICENSES.md`, `laf/misc/skia-url.sh`).
- Produces: Verification test report confirming all files referenced in packaging steps exist and workflow structure is valid.

- [ ] **Step 1: Write verification test `tests/verify-release-workflow.py`**

Write a script that:
1. Loads and parses `.github/workflows/release-windows.yml`.
2. Asserts required triggers exist (`push.tags`, `workflow_dispatch`).
3. Asserts required permissions exist (`contents: write`).
4. Asserts all static files copied during packaging exist on disk (`EULA.txt`, `README.md`, `docs/LICENSES.md`).
5. Checks that `laf/misc/skia-url.sh` exists and is executable.

- [ ] **Step 2: Run verification test**

Run:
```bash
python3 tests/verify-release-workflow.py
```
Expected: PASS with "All release workflow verifications passed."

- [ ] **Step 3: Clean up verification script or commit if useful**

Run:
```bash
rm -f tests/verify-release-workflow.py
```

---

### Task 3: Git Landing on Baseline Commit `331e898` and Tagging `v1.3.18.6`

**Files / Refs:**
- Baseline: `331e898ef405d919539c03fd1675691cfa16c3a3`
- Branch: `release/v1.3.18.6`
- Tag: `v1.3.18.6`

**Interfaces:**
- Consumes: Commit `331e898` and `.github/workflows/release-windows.yml`.
- Produces: Release commit on branch `release/v1.3.18.6` tagged as `v1.3.18.6`.

- [ ] **Step 1: Create release branch from baseline commit `331e898`**

Run:
```bash
git checkout -b release/v1.3.18.6 331e898
```
Verify: HEAD is at `331e898`.

- [ ] **Step 2: Apply the release workflow file to the release branch**

Run:
```bash
git checkout main -- .github/workflows/release-windows.yml
git commit -m "ci: add windows release workflow for v1.3.18.6"
```
Verify: Release branch contains commit `331e898` codebase + `.github/workflows/release-windows.yml`.

- [ ] **Step 3: Create annotated Git tag `v1.3.18.6`**

Run:
```bash
git tag -a v1.3.18.6 -m "Release v1.3.18.6"
```
Verify: `git tag -v v1.3.18.6` or `git show v1.3.18.6` displays the tag pointing to the new release commit.

- [ ] **Step 4: Return to `main` branch and verify tree clean**

Run:
```bash
git checkout main
git status
```
Verify: Working tree is clean on `main`, and tag `v1.3.18.6` is preserved.

---

## Plan Self-Review Check
- [x] Spec coverage: All requirements in spec are addressed across Tasks 1, 2, and 3.
- [x] Step scan: Each step has explicit commands and expected outcomes.
- [x] Type consistency: Tag naming `v1.3.18.6`, package name `Aseprite-${TAG}-windows-x64.zip`, and checksum `Aseprite-${TAG}-windows-x64.zip.sha256` are consistent across all tasks.
- [x] Review focus: All 5 review focus items are addressed in workflow design and test steps.
