# Agent instructions for this fork

If this repository is being used from ChatGPT, OpenAI Work, Codex, or another OpenAI containerized environment for CAD work, read and follow:

`OPENAI_CHATGPT_FREECAD.md`

Do this **before** rebuilding, reinstalling, or attempting to rediscover the FreeCAD runtime setup.

Key points:

- Headless FreeCAD is sufficient; a GUI is not required.
- The documented known-good baseline is FreeCAD 1.1.3 Linux x86_64.
- First check whether `/mnt/data/freecad_runtime/freecadcmd.sh` already exists in the current container/session.
- If it does not exist, use `.github/workflows/build_headless_for_chatgpt.yml` and its split GitHub Actions artifacts as documented in `OPENAI_CHATGPT_FREECAD.md`.
- Verify the official AppImage SHA-256 before execution.
- For real CAD tasks, validate `.FCStd`, STEP, and STL roundtrips before claiming completion.
- Prefer FreeCAD's Python API and editable `.FCStd`/STEP sources over approximate standalone STL generation.
