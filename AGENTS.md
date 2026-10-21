# AGENTS.md

MCP for Blender: a community MCP server (PyPI `mcp-for-blender`, not the Blender Foundation's) that lets
an LLM drive Blender. Two halves talk JSON over a TCP socket (default `localhost:9876`):

    MCP client <-stdio-> src/blender_mcp/server.py <-TCP-> addon.py running inside Blender (needs `bpy`)

## Layout

- `src/blender_mcp/server.py` - every MCP tool (`@mcp.tool()`) and `main()`, the `mcp-for-blender` entry point.
- `addon.py` (root) - the Blender add-on, about 6,800 lines. `src/blender_mcp/bundled/addon.py` is a byte-identical
  copy that ships in the wheel.
- `src/blender_mcp/blender_scripts.py` - scripts the server runs inside Blender through the add-on's `execute_code`.
- `addon_manager.py`, `setup_cli.py`, `update_cli.py` - the `install-addon` / `setup` / `update` subcommands.
- `telemetry*.py`, `trajectory.py`, `context_log.py` - usage logging. `generation.py` - paid 3D generation jobs.

## Commands

`src/blender_mcp/config.py` is gitignored (it holds the maintainer's telemetry keys), but `telemetry.py` imports it.
Without it, 6 tests in `test_compat_matrix.py` fail with `No module named 'blender_mcp.config'`. Create a stub
and never commit it:

    printf 'from types import SimpleNamespace\ntelemetry_config = SimpleNamespace(enabled=False, supabase_url="", supabase_anon_key="", supabase_bucket="")\n' > src/blender_mcp/config.py

- Install: `uv sync` (Python 3.11 from `.python-version`).
- Test: `DISABLE_TELEMETRY=1 uv run --with pytest pytest -q` (~35s, 549 passed, 1 skipped). pytest isn't a
  dependency, so `uv run pytest` alone dies at collection. Use a full clone: `test_compat_matrix.py` replays
  old `addon.py` versions with `git show`, and a shallow clone skips them quietly.
- The tests only read `addon.py` as text and never run it in Blender, so a green suite says nothing about panels,
  operators or `blender_scripts.py`. Report changes there as untested until they're checked in a real Blender.
- Run: `uv run mcp-for-blender [--host H --port P]`. It then waits silently on stdin for a client. That is not a hang.
- Lint/format: none configured. Match the surrounding code, and don't reformat files you aren't changing.

## Conventions

- The model sees exactly 9 tools, frozen in `MODEL_TOOLS` in `tests/test_consolidated_tools.py`. Extend an existing
  tool or add a script to `blender_scripts.py` instead. A new tool is a product decision, so ask first.
- Prefer server-side scripts over new add-on commands, since they work with add-ons users already installed.
  `test_compat_matrix.py` checks every tool against 5 historical add-on versions.
- Every model tool takes `ctx: Context` and `user_prompt: str = ""`. Copy the `@telemetry_tool` /
  `@trajectory_tool` decorators from the closest existing tool. Tool docstrings are prompt text the model reads.
- Tests are pytest functions named as sentences, faking Blender by monkeypatching `server.get_blender_connection`.
- Commit messages use a prefix: `fix:`, `feat:`, `docs:`, `hygiene:`, `test:`.

## Guardrails

- Edit the root `addon.py`, then `cp addon.py src/blender_mcp/bundled/addon.py` (a test checks they match).
- When add-on commands change, bump `ADDON_PROTOCOL_VERSION` in `addon.py` and `EXPECTED_ADDON_PROTOCOL_VERSION`
  in `addon_manager.py` together.
- Don't run `mcp-for-blender setup`, `install-addon` or `update`. They rewrite this machine's real
  Claude/Cursor configs and Blender add-on folder.
- `generate_3d` and the Meshy/Hunyuan/Tripo paths spend paid credits. Tests fake them; don't call them for real.
