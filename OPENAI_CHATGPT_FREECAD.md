# OpenAI / ChatGPT FreeCAD Runtime

This file is the handoff for OpenAI/ChatGPT sessions that need to create, inspect, repair, or validate CAD files with FreeCAD.

## Purpose

Do **not** rebuild the FreeCAD toolchain from scratch unless the documented runtime path no longer works. The repository contains a GitHub Actions workflow that prepares a known-good Linux x86_64 FreeCAD runtime for use in an OpenAI container.

Workflow:

`.github/workflows/build_headless_for_chatgpt.yml`

The runtime is intended for **headless CAD automation**. A GUI is not required.

## Known-good baseline

Validated on 2026-09-03:

- FreeCAD: **1.1.3**
- Linux: x86_64
- Python in upstream AppImage: 3.11
- Official upstream AppImage SHA-256:
  `3a853eb69ee595f779f2255dbf80a765926981d8ff68903cefee4dfb03a8f5ef`
- Repository commit that introduced the connector-friendly split artifact workflow:
  `40e0153260b5f3620b41a7e1563741898bcefa9c`
- Successful GitHub Actions run:
  `33725658093`

Required and locally validated modules:

- `FreeCAD`
- `Part`
- `PartDesign`
- `Sketcher`
- `Mesh`

Validated operations:

- BRep construction and validity checking
- `PartDesign::Feature`
- Sketch creation
- save `.FCStd`
- reopen `.FCStd`
- STEP export
- STEP re-import through OpenCascade
- STL export through `Mesh`
- relocation and execution from another directory

## Fast path in an OpenAI container

### 1. Check whether the runtime already exists

First check:

```bash
ls -la /mnt/data/freecad_runtime 2>/dev/null || true
```

If this directory exists and contains `freecadcmd.sh`, use it directly.

Typical launcher:

```bash
/mnt/data/freecad_runtime/freecadcmd.sh
```

Quick verification:

```bash
/mnt/data/freecad_runtime/freecadcmd.sh -c \
  'import FreeCAD, Part, PartDesign, Sketcher, Mesh; s=Part.makeBox(2,3,4); assert s.isValid(); print(FreeCAD.Version()); print(s.Volume)'
```

Expected volume: `24.0`.

### 2. If the runtime is absent

OpenAI tool/container sessions may be ephemeral. If `/mnt/data/freecad_runtime` is absent, use the GitHub Actions workflow above rather than trying to install FreeCAD through APT.

Reason: OpenAI containers may have restricted or unavailable DNS/network access, while the GitHub connector can retrieve GitHub Actions artifacts.

The workflow downloads the official FreeCAD 1.1.3 x86_64 AppImage, verifies its upstream SHA-256, extracts it, smoke-tests the required CAD modules, then splits the AppImage into connector-friendly chunks.

The split is required because the OpenAI GitHub connector has a per-artifact download limit below the full ~821 MB AppImage size.

Artifacts from a successful run are named:

- `freecad-appimage-part-00`
- `freecad-appimage-part-01`
- `freecad-appimage-part-02`
- `freecad-appimage-part-03`
- `freecad-appimage-part-04`

Download all five artifacts with the GitHub connector. Each artifact is a ZIP containing one `FreeCAD.AppImage.part-*` file.

Then locally:

```bash
mkdir -p /mnt/data/freecad_download
cd /mnt/data/freecad_download

# Unzip each downloaded GitHub artifact ZIP here.
# Afterwards there must be exactly these files:
# FreeCAD.AppImage.part-00 ... FreeCAD.AppImage.part-04

cat FreeCAD.AppImage.part-* > FreeCAD.AppImage

echo '3a853eb69ee595f779f2255dbf80a765926981d8ff68903cefee4dfb03a8f5ef  FreeCAD.AppImage' | sha256sum -c -
chmod +x FreeCAD.AppImage
./FreeCAD.AppImage --appimage-extract
```

Only continue if the SHA-256 check succeeds.

### 3. Create the persistent-in-session launcher

Use the extracted `squashfs-root` as `/mnt/data/freecad_runtime` or move it there:

```bash
rm -rf /mnt/data/freecad_runtime
mv squashfs-root /mnt/data/freecad_runtime
```

Locate FreeCADCmd:

```bash
find /mnt/data/freecad_runtime -type f \( -name FreeCADCmd -o -name freecadcmd \) -perm -111
```

Create `/mnt/data/freecad_runtime/freecadcmd.sh` if it is not already present. The launcher must resolve its own directory so the runtime remains relocatable.

A working pattern is:

```bash
#!/usr/bin/env bash
set -euo pipefail
ROOT="$(cd "$(dirname "$0")" && pwd)"
CMD="$(find "$ROOT" -type f \( -name FreeCADCmd -o -name freecadcmd \) -perm -111 | head -n1)"
exec "$CMD" "$@"
```

Then:

```bash
chmod +x /mnt/data/freecad_runtime/freecadcmd.sh
```

## Acceptance test before doing real CAD work

Do not claim that FreeCAD is available merely because the binary starts. Run an actual CAD roundtrip.

Minimum acceptance test:

1. import `FreeCAD`, `Part`, `PartDesign`, `Sketcher`, `Mesh`
2. construct a non-trivial valid BRep
3. save an `.FCStd`
4. close and reopen it
5. verify shape validity and volume
6. export STEP
7. re-import STEP and verify validity
8. export STL and verify non-zero file size

A previously successful proof used:

- box: 40 x 30 x 12 mm
- fused cylindrical boss
- cylindrical through-hole
- one Sketcher line
- `PartDesign::Feature`

Proof output from the 2026-09-03 local validation:

- volume: `17170.8847 mm^3` approximately
- FCStd: 4372 bytes
- STEP: 9959 bytes
- STL: 51284 bytes

Exact file sizes are not API guarantees; validity and successful roundtrip are what matter.

## Recommended CAD workflow for user projects

When the user asks for a FreeCAD/CAD design:

1. Use FreeCAD's Python API rather than approximating geometry with unrelated mesh libraries.
2. Build named parametric objects and properties where practical.
3. Keep dimensions explicit in the script/model.
4. Recompute the document before saving.
5. Validate shapes with `Shape.isValid()`.
6. Check critical bounding-box dimensions and volumes where useful.
7. Save `.FCStd` as the editable source of truth.
8. Export STEP for CAD interchange.
9. Export STL only as the print mesh.
10. Reopen the `.FCStd` and re-import the STEP before declaring the result finished.
11. For mechanical/print parts, also inspect wall thicknesses, clearances, support requirements, overhangs, fastener geometry, and assembly logic explicitly.

## User-specific CAD context

This runtime was originally prepared because the user is designing mechanically functional 3D-print parts, including a bicycle luggage-rack / Eurobox mounting system. Previous geometry produced without a reliable FreeCAD runtime contained dimensional and assembly mistakes.

For those projects, prefer mechanically checked FreeCAD models over visual approximations. Preserve editable `.FCStd` and STEP files whenever possible.

## What not to do

- Do not assume FreeCAD is installed system-wide.
- Do not spend time on `apt install freecad` before checking the documented runtime route.
- Do not depend on FUSE; extract the AppImage with `--appimage-extract`.
- Do not download or execute a reconstructed AppImage unless its SHA-256 matches the value above.
- Do not treat STL generation alone as sufficient validation.
- Do not claim CAD correctness without reopening/validating the generated files.

## Short instruction for a new ChatGPT/OpenAI session

The user can simply say:

> Use `martin-77/FreeCAD` and follow `AGENTS.md` / `OPENAI_CHATGPT_FREECAD.md`. Set up the documented headless runtime if it is not already present, then use FreeCAD for this CAD task.

That should be sufficient context to resume the FreeCAD workflow without rediscovering the setup history.
