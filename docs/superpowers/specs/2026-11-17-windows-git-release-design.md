# Design Specification: Windows Git Release Automation for Aseprite

- **Status**: Validated Design
- **Date**: 2026-11-17
- **Target Tag**: `v1.3.18.6`
- **Baseline Commit**: `331e898ef405d919539c03fd1675691cfa16c3a3` (`[win] Locate vcvars64.bat via vswhere instead of hardcoding the path`)

---

## 1. Objective and Scope

Implement an automated Windows release packaging and GitHub Release publishing pipeline for Aseprite.

The pipeline will:
1. Trigger automatically when a tag matching `v*` (such as `v1.3.18.6`) is pushed, or manually triggered via `workflow_dispatch`.
2. Check out the codebase with recursive submodules and prepare the MSVC x64 build environment.
3. Automatically retrieve and configure official matching Skia prebuilt libraries.
4. Build the release binary of Aseprite with Lua scripting support and Skia backend (`ENABLE_TESTS=OFF` for release efficiency).
5. Assemble the release distribution directory containing `aseprite.exe`, `data/` assets, `EULA.txt`, `README.md`, and `docs/LICENSES.md`.
6. Package the distribution into `Aseprite-${TAG}-windows-x64.zip` and generate standard SHA256 checksum file `Aseprite-${TAG}-windows-x64.zip.sha256`.
7. Publish or update the GitHub Release using native GitHub CLI (`gh release`).
8. Land the workflow on a commit derived from baseline commit `331e898` and apply tag `v1.3.18.6`, while maintaining the workflow on `main`.

---

## 2. GitHub Actions Workflow Architecture

### 2.1 File Location
`.github/workflows/release-windows.yml`

### 2.2 Trigger Configuration
```yaml
name: release-windows

on:
  push:
    tags:
      - 'v*'
  workflow_dispatch:
    inputs:
      tag_name:
        description: 'Tag or Version name for the release (e.g. v1.3.18.6)'
        required: false
        default: ''
      is_draft:
        description: 'Publish as draft release'
        required: false
        type: boolean
        default: false
      is_prerelease:
        description: 'Publish as pre-release'
        required: false
        type: boolean
        default: false
```

### 2.3 Job Permissions
```yaml
permissions:
  contents: write
```

---

## 3. Build & Packaging Pipeline Specification

### 3.1 Runner and Toolchain
- **Runner**: `windows-latest`
- **Checkout**: `actions/checkout@v4` with `submodules: 'recursive'`
- **Ninja**: `aseprite/get-ninja@main`
- **MSVC Dev Environment**: `TheMrMilchmann/setup-msvc-dev@v4` with `arch: x64`

### 3.2 Dynamic Skia Retrieval
To ensure maximum compatibility with the codebase's specific Skia dependency:
```bash
this_dir=$(cygpath "${{ github.workspace }}")
skia_url=$(source $this_dir/laf/misc/skia-url.sh | xargs)
skia_file=$(basename $skia_url)
curl --ssl-revoke-best-effort -L -o "$skia_file" "$skia_url"
unzip "$skia_file" -d skia
```

### 3.3 CMake Configuration and Compilation
- **Build Type**: `RelWithDebInfo`
- **Options**:
  - `ENABLE_TESTS=OFF` (speeds up release packaging)
  - `ENABLE_SCRIPTING=ON` (Lua scripting enabled)
  - `ENABLE_CCACHE=OFF`
  - `LAF_BACKEND=skia`
  - `SKIA_DIR=skia`
  - `SKIA_LIBRARY_DIR=skia/out/Release-x64`
- **Build Target**: `ninja -C build aseprite` (triggers `copy_data` target automatically)

### 3.4 Packaging Layout & Checksum
Create release packaging directory `package/Aseprite-${TAG}-windows-x64/`:
```text
Aseprite-${TAG}-windows-x64/
├── aseprite.exe
├── data/
├── EULA.txt
├── README.md
└── docs/
    └── LICENSES.md
```

Packaging script commands (PowerShell):
```powershell
$tag = "${{ env.RELEASE_TAG }}"
$packageDir = "Aseprite-$tag-windows-x64"
$zipFile = "Aseprite-$tag-windows-x64.zip"
$shaFile = "$zipFile.sha256"

New-Item -ItemType Directory -Path "package/$packageDir" -Force
Copy-Item -Path "build/bin/aseprite.exe" -Destination "package/$packageDir/"
Copy-Item -Path "build/bin/data" -Destination "package/$packageDir/" -Recurse
Copy-Item -Path "EULA.txt" -Destination "package/$packageDir/"
Copy-Item -Path "README.md" -Destination "package/$packageDir/"
New-Item -ItemType Directory -Path "package/$packageDir/docs" -Force
Copy-Item -Path "docs/LICENSES.md" -Destination "package/$packageDir/docs/"

Compress-Archive -Path "package/$packageDir/*" -DestinationPath "$zipFile" -Force
$hash = (Get-FileHash -Path "$zipFile" -Algorithm SHA256).Hash.ToLower()
"$hash  $zipFile" | Out-File -FilePath "$shaFile" -Encoding utf8
```

---

## 4. GitHub Release Publication Specification

Release is published via Runner native GitHub CLI (`gh release`):
```bash
if gh release view "$RELEASE_TAG" >/dev/null 2>&1; then
  echo "Release $RELEASE_TAG already exists. Uploading/updating assets..."
  gh release upload "$RELEASE_TAG" "$ZIP_FILE" "$SHA_FILE" --clobber
else
  echo "Creating release $RELEASE_TAG..."
  gh release create "$RELEASE_TAG" "$ZIP_FILE" "$SHA_FILE" \
    --title "Aseprite $RELEASE_TAG" \
    --generate-notes \
    $DRAFT_FLAG $PRERELEASE_FLAG
fi
```
Environment token passed as `env: GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}`.

---

## 5. Git Landing and Tag Binding Strategy

Because GitHub Actions triggers workflows based on the `.github/workflows/` files in the commit referenced by the tag:
1. Commit `.github/workflows/release-windows.yml` to the repository.
2. In order to bind Tag `v1.3.18.6` to commit `331e898`:
   - Create a release branch/commit branching from `331e898`.
   - Apply `.github/workflows/release-windows.yml` onto that commit.
   - Tag `v1.3.18.6` on that commit.
3. Also commit `.github/workflows/release-windows.yml` to `main` so subsequent tags on `main` inherit the workflow seamlessly.

---

## 6. Verification and Testing

1. **Workflow Syntax & Schema**: Verify YAML structure, step IDs, conditional expressions, and environment variables.
2. **Local Packaging Check**: Validate file existence and paths (`EULA.txt`, `README.md`, `docs/LICENSES.md`, `laf/misc/skia-url.sh`, data directories) against current repository tree.
3. **Git Tag & Branch State**: Verify `git tag v1.3.18.6` is successfully created and points to the intended commit tree.
