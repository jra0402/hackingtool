# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

HackingTool is an interactive terminal menu (Python 3.10+, Rich UI) that aggregates 185+ security tools into a single launcher. It does not implement any attack logic itself — it installs external tools and wraps their CLI invocations. All use is intended for authorized security testing and CTF/research contexts.

## Commands

**Run the app (after install):**
```bash
hackingtool
```

**Run from source (development):**
```bash
python3 hackingtool.py
```

**Install (requires root):**
```bash
sudo python3 install.py
```

**Install Python dependencies only:**
```bash
pip install -r requirements.txt   # only `rich>=13.0.0`
```

**Lint (CI uses ruff):**
```bash
pip install ruff
ruff check . --select=E9,F63,F7,F82,PLE,YTT   # must-fix errors
ruff check . --select=ALL --statistics --target-version=py310   # all issues
```

**Format check:**
```bash
pip install black
black --check .
```

**Type check:**
```bash
pip install mypy
mypy --ignore-missing-imports .
```

**Regenerate README.md from template:**
```bash
python3 generate_readme.py
```

**Smoke test (CI pattern — no interactive TTY needed):**
```bash
echo -e "99\n" | hackingtool || true
```

## Architecture

### Core abstractions (`core.py`)

Two classes power every tool and category:

- **`HackingTool`** — base class for a single external tool. Subclasses declare class-level attributes:
  - `TITLE`, `DESCRIPTION`, `PROJECT_URL`
  - `INSTALL_COMMANDS`, `RUN_COMMANDS`, `UNINSTALL_COMMANDS` — lists of shell strings run via `os.system()`
  - `SUPPORTED_OS` — `["linux"]` or `["linux", "macos"]`; used to hide tools on incompatible systems
  - `TAGS` — list of strings (e.g. `["osint", "web"]`) for search/filter
  - `REQUIRES_ROOT`, `REQUIRES_GO`, `REQUIRES_RUBY`, `REQUIRES_DOCKER`, `REQUIRES_JAVA`, `REQUIRES_WIFI` — metadata flags
  - `ARCHIVED`, `ARCHIVED_REASON` — marks deprecated tools; they stay visible under option 98 instead of the main list
  - The `is_installed` property checks `shutil.which()` on the first `RUN_COMMANDS` binary or whether the git clone target dir exists

- **`HackingToolsCollection`** — groups multiple tools into a category menu. Subclasses set `TITLE` and `TOOLS` (a list of `HackingTool` instances). The `show_options()` loop renders the table, handles OS filtering, batch install (option 97), and archived sub-menu (option 98). Both classes share an **iterative menu loop** (no recursion).

### Module layout

```
hackingtool.py        entry point — builds all_tools list, main menu, search/tag/recommend
core.py               HackingTool + HackingToolsCollection base classes + shared console
constants.py          paths, version, theme tokens, DEFAULT_CONFIG
config.py             load/save ~/.hackingtool/config.json; get_tools_dir(), get_sudo_cmd()
os_detect.py          CURRENT_OS singleton (OSInfo); pkg manager detection and install helpers
install.py            system installer (requires root) — sets up venv, copies files, creates /usr/bin/hackingtool symlink
generate_readme.py    regenerates README.md by filling README_template.md with tool lists
tools/                one module per category, each exports a *Collection class
tools/others/         sub-tools with custom logic (social media, homograph, payload injection)
tools/tool_manager.py UpdateTool + UninstallTool (category 21 in the main menu)
```

### Shared `console`

All tool files import the single Rich `Console` instance from `core`:
```python
from core import console
```
Never create a second `Console`. The theme tokens (`THEME_PRIMARY`, `THEME_SUCCESS`, etc.) are defined in `constants.py` and registered in `core.py`.

### OS awareness

`os_detect.CURRENT_OS` is computed once at import time and is a module-level singleton. `HackingToolsCollection._active_tools()` filters tools by `CURRENT_OS.system` against `SUPPORTED_OS`. Linux-only tools are automatically hidden on macOS without any changes needed in the menu code.

### Config and paths

User config lives at `~/.hackingtool/config.json`. **Never hardcode `/home/username`** — always use `Path.home()` (enforced in `constants.py`). System install paths differ by OS: `/usr/share/hackingtool` on Linux, `/usr/local/share/hackingtool` on macOS.

Privilege escalation: use `PRIV_CMD` from `constants.py` or `config.get_sudo_cmd()` — it prefers `doas` over `sudo` if available.

## Adding a New Tool

1. Find the right `tools/*.py` module for the category.
2. Create a class inheriting `HackingTool` with all required attributes (`TITLE`, `DESCRIPTION`, `INSTALL_COMMANDS`, `RUN_COMMANDS`, `PROJECT_URL`, `SUPPORTED_OS`, `TAGS`).
3. Add an instance to the `TOOLS` list in the module's `*Collection` class at the bottom of the file.
4. No changes needed in `hackingtool.py` or `core.py`.

PR title must follow: `[New Tool] ToolName — Category`. See `.github/PULL_REQUEST_TEMPLATE.md` for the full checklist.

## CI

Two GitHub Actions workflows run on push/PR to `master`:

- **`lint_python.yml`** — ruff (must-fix + all), black (check only), codespell, mypy, pytest
- **`test_install.yml`** — installs deps, runs `sudo python3 install.py 1`, smoke-tests the menu with piped input

The ruff must-fix rule set is `E9,F63,F7,F82,PLE,YTT`. Violations in that set block CI; everything else is advisory.
