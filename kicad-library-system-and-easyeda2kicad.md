---
tags: [kicad, pcb, libraries, easyeda2kicad]
---

# KiCad Library System & `easyeda2kicad.py` Workflow

## 1. KiCad file types, quick reference

| Extension | What it is | Notes |
|---|---|---|
| `.kicad_sym` | A **symbol library** — a single file containing *many* schematic symbols | Opened/edited in the Symbol Editor |
| `.kicad_sch` | A schematic sheet | One project can have many sheets |
| `.kicad_mod` | A **single footprint** | One file = one footprint |
| `.pretty/` | A **footprint library** — just a folder full of `.kicad_mod` files | The folder itself is the "library"; KiCad treats any `.pretty` dir as one |
| `.step` / `.wrl` | 3D models | `.step` is the modern preferred format (solid model); `.wrl` is the legacy VRML format |
| `.kicad_pcb` | The PCB layout file | |
| `.kicad_pro` | Project file — settings, not geometry | |
| `.kicad_prl` | Local project settings (window layout etc.) — usually not worth committing to git |
| `sym-lib-table` | Maps a **nickname** → a `.kicad_sym` file path | Exists globally *and* per-project |
| `fp-lib-table` | Maps a **nickname** → a `.pretty` folder path | Exists globally *and* per-project |

A footprint's 3D model is *referenced by path* from inside the `.kicad_mod` file — it's not embedded. If that path breaks (e.g. you move folders), the footprint still works electrically but shows no 3D model.

## 2. Global vs. project libraries

KiCad has **two** library tables of each kind:

- **Global** (`sym-lib-table` / `fp-lib-table` in your KiCad user config folder) — available to *every* project on this machine. Good for parts you reuse across many designs (passives, connectors, MCUs you use repeatedly).
- **Project-local** (same filenames, but sitting in the project folder itself) — only visible inside that project. Good for one-off parts, LCSC imports, anything you don't want polluting every future project's library picker.

**Use `${KIPRJMOD}`** (KiCad's built-in "current project directory" variable) for project-local paths instead of absolute paths. That keeps the project portable — clone the repo to another machine and the libraries still resolve, no manual relinking.

Example project-local `fp-lib-table` entry:
```
(lib (name "my-project-parts")(type "KiCad")(uri "${KIPRJMOD}/libraries/my-project/my-project.pretty")(options "")(descr ""))
```

## 3. Recommended structure: one growing library per PCB project

```
<project-root>/
├── <project>.kicad_pro
├── <project>.kicad_sch
├── <project>.kicad_pcb
└── libraries/
    └── <project>/
        ├── <project>.kicad_sym       <- all imported symbols accumulate here
        ├── <project>.pretty/         <- all imported footprints accumulate here
        │   ├── PART1.kicad_mod
        │   └── PART2.kicad_mod
        └── <project>.3dshapes/       <- 3D models
            ├── PART1.step
            └── PART2.step
```

Commit the whole `libraries/<project>/` folder to git along with the schematic/PCB. The project is then fully self-contained — anyone (including future you) can clone it and open it with no missing-library errors.

**Why per-project instead of one big global library:**
- No accumulation of one-off/low-quality auto-converted parts in your permanent global library.
- Each project's library is frozen at exactly what that project used — no risk of a later "cleanup" of a shared library breaking an old design.
- Easy to delete/archive a whole project including its parts without affecting anything else.

## 4. `easyeda2kicad.py` — install & usage

### Install (WSL — keeps KiCad itself on Windows, tool stays in WSL)

KiCad runs on Windows, but since the project lives under `/mnt/c/...`, a WSL-side install of `easyeda2kicad` writes files straight onto the same filesystem KiCad reads from — no need to touch the Windows "KiCad Command Prompt" or its bundled Python at all.

Debian/Ubuntu's system Python is "externally managed" and blocks a bare `pip install`, and `python3 -m venv` needs `python3-venv` (apt, sudo) to bundle pip — so the no-sudo path is: create a venv without pip, then bootstrap pip into it directly.

```bash
# one-time setup
mkdir -p ~/.local/venvs
python3 -m venv --without-pip ~/.local/venvs/easyeda2kicad
curl -sS -o /tmp/get-pip.py https://bootstrap.pypa.io/get-pip.py
~/.local/venvs/easyeda2kicad/bin/python3 /tmp/get-pip.py
~/.local/venvs/easyeda2kicad/bin/python3 -m pip install easyeda2kicad
```

Then put this in `~/.bash_aliases` (sourced automatically by `~/.bashrc`) so the command is just on your `PATH` in every new shell, no venv activation needed:

```bash
# easyeda2kicad (KiCad part importer) - isolated venv, kept off system Python
export PATH="$HOME/.local/venvs/easyeda2kicad/bin:$PATH"
```

Open a new terminal (or `source ~/.bash_aliases`) and `easyeda2kicad` just works as a normal command.

### Core commands

```bash
# Symbol + footprint + 3D model, all at once (the one you'll use most)
easyeda2kicad --full --lcsc_id=C2040 --output "./libraries/<project>/<project>"

# Just one piece, if you need to redo only part of an import
easyeda2kicad --symbol    --lcsc_id=C2040 --output "./libraries/<project>/<project>"
easyeda2kicad --footprint --lcsc_id=C2040 --output "./libraries/<project>/<project>"
easyeda2kicad --3d        --lcsc_id=C2040 --output "./libraries/<project>/<project>"

# Several parts in one shot
easyeda2kicad --full --lcsc_id C2040 C20197 C163691 --output "./libraries/<project>/<project>"

# Preview as SVG before committing to an import
easyeda2kicad --svg --lcsc_id=C2040 --output "./libraries/<project>/<project>"
```

### Useful flags

| Flag | Purpose |
|---|---|
| `--output <path>` | **Always set this** to your project's `libraries/<project>/<project>` path — this is what makes it "per-project" instead of dumping into the tool's default global cache folder |
| `--overwrite` | Replace an already-imported component (use when re-importing after a correction) |
| `--project-relative` | Write 3D model paths as `${KIPRJMOD}`-relative instead of absolute — combine with the project-local structure above |
| `--custom-field "Key:Value"` | Stamp extra metadata onto the symbol (e.g. supplier, datasheet link) |
| `--use-cache` | Caches EasyEDA/LCSC API responses locally, avoids re-hitting the API on repeat runs |
| `--debug` | Verbose logging if an import silently fails |

Because each run **appends** to the existing `.kicad_sym`/`.pretty`/`.3dshapes` at the given `--output` path (rather than truncating), this is exactly the "toss LCSC numbers at it as I design, they all land in the same growing project library" workflow.

### Wiring the imported library into KiCad (per-project)

In KiCad: **Preferences → Manage Symbol Libraries...** and **Manage Footprint Libraries...** — make sure you pick the **Project Specific Libraries** tab, not Global, then add:

- Symbol lib: `${KIPRJMOD}/libraries/<project>/<project>.kicad_sym`
- Footprint lib: `${KIPRJMOD}/libraries/<project>/<project>.pretty`

Give it a nickname (e.g. the project name) — that's what shows up in the symbol/footprint chooser.

## 5. Best practices

- **Always verify imported parts before trusting them in a layout.** The tool's own docs state symbol/footprint correctness "can't be guaranteed" — EasyEDA's library is community-maintained, same as KiCad's. Check pin numbering against the datasheet, and check pad sizes/silkscreen against the datasheet footprint drawing, especially for fine-pitch or unusual packages.
- **Keep a running notes file of LCSC IDs used per project** (e.g. `libraries/<project>/parts.md` with `C2040 — ESP32-WROOM-32, C20197 — ...`). If you ever need to regenerate the library (corrupted file, KiCad version bump) you can replay the exact import list instead of hunting down parts again.
- **Don't mix project-local and global libraries for the same part.** Pick one location per part type so you're not duplicating/diverging copies.
- **Re-import rather than hand-edit** when a part needs fixing, if at all possible — hand edits get lost if you ever `--overwrite` that part again later.
- **Commit `libraries/<project>/` to git, but consider `.gitignore`-ing the `.easyeda_cache/` folder** if you use `--use-cache` — it's just an API response cache, not project data.
- **3D models bloat repos.** `.step` files can be several MB each. If repo size becomes an issue, consider Git LFS for `*.step`/`*.wrl`, or simply accept the cost — schematics/PCBs are small, models are the heavy part.
