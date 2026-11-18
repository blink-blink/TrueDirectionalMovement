# AGENTS.md

## Project
SKSE plugin fork of True Directional Movement 2.2.7 (`blink-blink/TrueDirectionalMovement`).
C++20, CMake, CommonLibSSE-NG. Working branch: `master`.

## Build: GitHub Actions (preferred)
Push to `master` (or any branch) of `blink-blink/TrueDirectionalMovement`; `.github/workflows/main.yml`
runs automatically and builds the `github-actions` CMake preset on `windows-2022`.

```powershell
git push origin master
gh run list --workflow Main --limit 5
gh run watch <run-id> --exit-status
```

Download the build output (artifact name `TrueDirectionalMovement NG Release`, contains
`TrueDirectionalMovement.dll` and `.pdb`):

```powershell
gh run download <run-id> -n "TrueDirectionalMovement NG Release" -D "$env:TEMP\tdm-artifact"
```

## Deploy: Custom Build mod
Overwrite the DLL/PDB in the SME load order:

```powershell
$dest = "D:\Skyrim Modding\Skyrim Modding Tools\Wabbajack\modpacks\SME\mods\True Directional Movement - Custom Build\SKSE\Plugins"
Copy-Item "$env:TEMP\tdm-artifact\TrueDirectionalMovement.dll" $dest -Force
Copy-Item "$env:TEMP\tdm-artifact\TrueDirectionalMovement.pdb" $dest -Force
```

The `Textures`, `Meshes`, `MCM`, `Interface` etc. in that mod folder are managed separately;
only replace SKSE plugin binaries.

## Local build (fallback)
- Requirements: Visual Studio 2022/2026 C++ workload, CMake >= 3.22, vcpkg
  (`VCPKG_ROOT`), CommonLibSSE-NG.
- The fork's stock CMake expects `extern/CommonLibSSE` (submodule) or the
  `CommonLibSSEPath_NG` env var. The `extern/` submodules are not initialized in this clone.
- `COMMONLIB_SSE_FOLDER` is set in the environment but only the *other* clone's local CMake
  edits consume it; the fork's CMake does not.
- Do **not** reconfigure `C:\TDMBuild`: its cache points at the other clone:
  `C:\Users\Ryzen 7\Documents\Programming\SKSE Modding\TrueDirectionalMovement`.
- If a local build is unavoidable, use a dedicated binary dir, e.g.
  `cmake -S . -B C:\TDMBuild-Fork -G "Visual Studio 18 2026" -A x64 -DCOPY_OUTPUT=OFF`
  and pass `-DCommonLibSSEPath_NG=...` explicitly.

## Safety
- The clone at `...\Programming\SKSE Modding\TrueDirectionalMovement` holds uncommitted
  experimental work (`src/LockOnCamera.h`, CMake changes). Do not copy from, build into,
  or overwrite it.
- `C:\TDMBuild` is configured against that clone; leave its cache alone.
