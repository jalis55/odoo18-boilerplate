# Odoo 18 Development Boilerplate

A ready-to-use, batteries-included template for developing **Odoo 18** modules and
customizations. It bundles the Odoo 18.0 community source, an isolated Python
virtual environment, a PostgreSQL configuration, a dedicated folder for your own
addons, and a complete **VS Code** debugging/tooling setup out of the box.

Clone this directory and start writing modules — the plumbing is already done.

---

## Table of contents

- [What's inside](#whats-inside)
- [Project structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Quick start](#quick-start)
- [Configuration](#configuration)
- [Database setup](#database-setup)
- [Running Odoo](#running-odoo)
- [VS Code setup](#vs-code-setup)
- [Developing custom modules](#developing-custom-modules)
- [Command cheatsheet](#command-cheatsheet)
- [Troubleshooting](#troubleshooting)
- [License & credits](#license--credits)

---

## What's inside

| Component | Path | Description |
| --- | --- | --- |
| Odoo 18 source | `odoo-18.0/` | Upstream Odoo 18.0 community (`odoo-bin`, framework, 627 official addons) |
| Custom addons | `custom_modules/` | **Your modules go here.** Already on the `addons_path` |
| Server config | `conf/odoo.conf` | Database, ports, addons path, master password |
| Python venv | `.venv/` | Python 3.12 environment (created with `uv`) |
| VS Code | `.vscode/` | Debug config, interpreter settings, tasks, recommended extensions |
| Project metadata | `pyproject.toml` | `uv`-managed project definition |
| Python version | `.python-version` | Pins Python `3.12` |

Verified baseline for this boilerplate:

- **Odoo 18.0** (final) · **Python 3.12** · **PostgreSQL 18**
- **Node.js 24** / **npm 11** (only needed for the optional JS/OWL asset tooling)
- **uv** for Python environment management

---

## Project structure

```
18.0/
├── .venv/                      # Python 3.12 virtual environment (uv)
├── .vscode/                    # VS Code workspace files
│   ├── launch.json             #   → debug configuration "Odoo 18" (F5)
│   ├── settings.json           #   → interpreter + Pylance extra paths
│   ├── tasks.json              #   → start/update/install module tasks
│   └── extensions.json         #   → recommended extensions
├── conf/
│   └── odoo.conf               # Odoo server configuration
├── odoo-18.0/                  # Odoo 18.0 community source (upstream, read-mostly)
│   ├── odoo-bin                #   → server entry point
│   ├── odoo/                   #   → core framework package
│   │   └── addons/             #   → 30 core modules (base, web, ...)
│   ├── addons/                 #   → 627 official modules
│   ├── requirements.txt        #   → Python dependencies
│   └── setup/                  #   → packaging / tooling scripts
├── custom_modules/             # ⬅ YOUR custom addons live here
├── src/
│   └── 18_0/__init__.py        # uv project package
├── .gitignore
├── .python-version             # 3.12
├── pyproject.toml              # uv project metadata
└── README.md
```

> `odoo-18.0/` is the untouched upstream checkout — treat it as read-only and keep
> all of your work inside `custom_modules/`.

---

## Prerequisites

Install these once on your machine:

| Tool | Version used | Check with |
| --- | --- | --- |
| Python | 3.12 | `python3 --version` |
| uv | ≥ 0.12 | `uv --version` |
| PostgreSQL | ≥ 14 (18 used here) | `psql --version` |
| git | any recent | `git --version` |
| Node.js + npm | 24 / 11 *(optional)* | `node --version` |

The Python interpreter itself can be provided by `uv` (it downloads and manages
CPython), so you only strictly need `uv` and PostgreSQL to get started.

---

## Quick start

```bash
# 1. Install Odoo's Python dependencies into the existing virtualenv
uv pip install -r odoo-18.0/requirements.txt      # or: .venv/bin/pip install -r odoo-18.0/requirements.txt

# 2. Make sure PostgreSQL is running and the 'odoo' role exists (see Database setup)

# 3. Create + initialize a database
.venv/bin/python odoo-18.0/odoo-bin -c conf/odoo.conf -d odoo18 -i base --stop-after-init

# 4. Start the server
.venv/bin/python odoo-18.0/odoo-bin -c conf/odoo.conf
```

Then open <http://localhost:8069> and log in with `admin` / `admin`.

---

## Configuration

All server options live in [`conf/odoo.conf`](conf/odoo.conf):

```ini
[options]
addons_path = .../odoo-18.0/addons,.../odoo-18.0/odoo/addons,.../custom_modules
db_host = localhost
db_port = 5432
db_user = odoo
db_password = odoo
admin_passwd =  # master password (hashed)
http_port = 8069
log_level = info
```

| Key | Meaning | Notes |
| --- | --- | --- |
| `addons_path` | Comma-separated module search paths | **`custom_modules` must stay in this list** |
| `db_host` / `db_port` | PostgreSQL connection | `localhost:5432` by default |
| `db_user` / `db_password` | Database role | `odoo` / `odoo` |
| `admin_passwd` | Master password for DB management | Stored **hashed**; see below |
| `http_port` | Web port | `8069` |
| `log_level` | Verbosity | `info`, or `debug` while troubleshooting |

**Changing the master password:** set a new plaintext value and let Odoo hash it, e.g.

```bash
.venv/bin/python odoo-18.0/odoo-bin -c conf/odoo.conf --save \
  --admin-passwd="your-new-password"
```

> The committed `admin_passwd` is the hash of a throwaway dev password. Rotate it
> before exposing an instance to a network.

**Adding more addons directories:** append them to `addons_path` (comma separated).
Because the file contains absolute paths, use `$(pwd)`-relative editing or an
absolute path matching your machine when you clone the boilerplate elsewhere.

---

## Database setup

This boilerplate expects a PostgreSQL role named `odoo` with password `odoo`
(matching `conf/odoo.conf`). Create it once:

```bash
sudo -u postgres psql <<'SQL'
CREATE ROLE odoo WITH LOGIN CREATEDB PASSWORD 'odoo';
SQL
```

Verify the connection:

```bash
PGPASSWORD=odoo psql -h localhost -U odoo -d postgres -c 'select current_user;'
```

Then create an Odoo database (this installs the `base` module and generates the
schema). Odoo creates the DB automatically when it does not exist:

```bash
.venv/bin/python odoo-18.0/odoo-bin -c conf/odoo.conf -d odoo18 -i base --stop-after-init
```

Useful flags: `-d <db>` (database), `-i <mods>` (install), `-u <mods>` (upgrade),
`--stop-after-init` (exit when done), `--without-demo=all` (no demo data).

---

## Running Odoo

### From the command line

```bash
# Start (development mode with auto-reload on Python changes)
.venv/bin/python odoo-18.0/odoo-bin -c conf/odoo.conf --dev=reload

# Install / update a module
.venv/bin/python odoo-18.0/odoo-bin -c conf/odoo.conf -d odoo18 -u my_module --stop-after-init

# Debug-level logs
.venv/bin/python odoo-18.0/odoo-bin -c conf/odoo.conf --log-level=debug
```

### From VS Code

Press **F5** (or *Run and Debug ▸ "Odoo 18"*) and Odoo starts under the debugger.
See the [VS Code setup](#vs-code-setup) section for details.

`--dev=reload` restarts the server when Python files change. Combine with `xml` to
also reload views without a restart (`--dev=reload,xml`), or use `--dev=all`.

---

## VS Code setup

This boilerplate ships a fully wired `.vscode/` folder so that opening
**`18.0/`** as the workspace root gives you IntelliSense, one-key debugging, and
ready-made tasks. Everything below is already committed — you only need to point
VS Code at the right interpreter the first time.

### 1. Open the correct folder

Open the **`18.0/` directory itself** (the one containing `.vscode/`), not its
parent. VS Code reads `.vscode/*.json` relative to that root:

```
File ▸ Open Folder… ▸ /path/to/odoo-tuts/18.0
```

### 2. Install the recommended extensions

Open the Extensions view (`Ctrl+Shift+X`). VS Code prompts
*"This workspace has extension recommendations"* and you can **Install All**.
Or install them by ID:

| Extension | ID | Why |
| --- | --- | --- |
| Python | `ms-python.python` | Interpreter, run/debug, testing |
| Pylance | `ms-python.vscode-pylance` | IntelliSense / type checking |
| Python Debugger | `ms-python.debugpy` | Powers the `debugpy` launch config |
| Ruff | `charliermarsh.ruff` | Fast linting + formatting |
| XML | `redhat.vscode-xml` | Validation for Odoo `.xml` views |
| Odoo Snippets | `mstuttgart.odoo-snippets` | Snippets for models/views/actions |

These are declared in [`.vscode/extensions.json`](.vscode/extensions.json).

### 3. Select the Python interpreter

The workspace settings already point to the bundled virtualenv:

```jsonc
// .vscode/settings.json
{
  "python.defaultInterpreterPath": "${workspaceFolder}/.venv/bin/python",
  "python.analysis.extraPaths": [
    "${workspaceFolder}/odoo-18.0"      // lets Pylance resolve `import odoo`
  ],
  "files.exclude": { "**/__pycache__": true }
}
```

Still, confirm it once so VS Code caches the choice:

1. `Ctrl+Shift+P` → **Python: Select Interpreter**
2. Pick `./.venv/bin/python` (Python 3.12)

`python.analysis.extraPaths` adds the Odoo source root to Pylance's search path,
which is what makes `from odoo import models, fields, api` resolve and gives you
autocompletion inside your models.

> **Note:** keep dependencies installed in `.venv` (see
> [Quick start](#quick-start)). The venv shipped in a fresh clone is empty, so
> run `Install Odoo requirements` before the first launch.

### 4. Debug configuration — `launch.json`

[`.vscode/launch.json`](.vscode/launch.json) defines the **"Odoo 18"** run/debug
profile started with `F5`:

```jsonc
{
  "name": "Odoo 18",
  "type": "debugpy",
  "request": "launch",
  "program": "${workspaceFolder}/odoo-18.0/odoo-bin",
  "args": ["-c", "${workspaceFolder}/conf/odoo.conf", "--dev=reload"],
  "console": "integratedTerminal",
  "cwd": "${workspaceFolder}",
  "python": "${workspaceFolder}/.venv/bin/python",
  "justMyCode": false
}
```

| Field | Purpose |
| --- | --- |
| `program` | Runs `odoo-bin` from source (no install needed) |
| `args` | Uses `conf/odoo.conf` + `--dev=reload` (auto-restart on Python edits) |
| `cwd` | Workspace root, so relative paths resolve consistently |
| `python` | Forces the `.venv` interpreter |
| `justMyCode: false` | **Lets you step into Odoo core** when debugging |

Press **F5** to start, set breakpoints in your model code, and inspect variables
with the *Run and Debug* panel. `Shift+F5` stops the server.

**Debugging tips**

- Set breakpoints directly in your `models/*.py`; they hit on the next request
  (with `--dev=reload` no manual restart is needed after edits).
- To attach to an already-running server instead, add a config with
  `"request": "attach"` and `"port": 5678`, and start Odoo with
  `--debugpy` / `--debug`.
- Use `--dev=reload,xml` (edit the `args` array) to reload QWeb views on the fly.

### 5. Tasks — `tasks.json`

[`.vscode/tasks.json`](.vscode/tasks.json) automates the common operations.
Run them via `Ctrl+Shift+P` → **Tasks: Run Task**, or `Terminal ▸ Run Task`.

| Task | What it does |
| --- | --- |
| **Odoo: Start server** | Launches `odoo-bin -c conf/odoo.conf --dev=reload` (default *build* task, `Ctrl+Shift+B`) |
| **Odoo: Init/Update database** | Creates/initializes the DB and installs `base` |
| **Odoo: Update module** | Prompts for DB + module names, runs `-u … --stop-after-init` |
| **Odoo: Install module** | Prompts for DB + module names, runs `-i … --stop-after-init` |
| **pip: Install Odoo requirements** | `pip install -r odoo-18.0/requirements.txt` into `.venv` |

The module/database names are prompted through `inputs`, so no editing is needed
between runs.

### 6. Optional: Odoo JS / OWL asset tooling

If you write **JavaScript / OWL** components you can enable Odoo's bundled
ESLint + TypeScript tooling (requires Node/npm). Odoo ships the scripts:

```bash
cd odoo-18.0/addons/web/tooling
./enable.sh      # generates eslint/tsconfig at the repo root
./refresh.sh     # re-install after updating deps
```

After enabling, reload the ESLint service and, if desired, set the default
formatter in `settings.json`:

```jsonc
{ "editor.defaultFormatter": "dbaeumer.vscode-eslint" }
```

(This is optional — the Python/Debug/tasks setup works without Node.)

---

## Developing custom modules

Create every module inside **`custom_modules/`** (already on the `addons_path`).

```bash
mkdir -p custom_modules/my_module/{models,views,security,static/description}
touch custom_modules/my_module/__init__.py custom_modules/my_module/__manifest__.py
```

Minimal `__manifest__.py`:

```python
{
    "name": "My Module",
    "version": "18.0.1.0.0",
    "license": "LGPL-3",
    "depends": ["base"],
    "data": ["views/my_model_views.xml"],
    "installable": True,
    "application": False,
}
```

A typical layout:

```
custom_modules/my_module/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   └── my_model.py
├── views/
│   └── my_model_views.xml
├── security/
│   └── ir.model.access.csv
└── static/description/icon.png
```

Install / update your module after editing (Python changes need a reload or
update; XML-only changes reload faster with `--dev=reload,xml`):

```bash
.venv/bin/python odoo-18.0/odoo-bin -c conf/odoo.conf -d odoo18 -u my_module --stop-after-init
```

Or use the VS Code task **Odoo: Update module**.

> Versioning convention: `18.0.<module_major>.<module_minor>.<patch>.<build>`
> (e.g. `18.0.1.0.0`), and keep the module folder name lowercase with underscores.

---

## Command cheatsheet

| Goal | Command |
| --- | --- |
| Install Odoo deps | `.venv/bin/pip install -r odoo-18.0/requirements.txt` |
| Create `.venv` fresh | `uv venv --python 3.12 .venv` |
| Init/update DB | `.venv/bin/python odoo-18.0/odoo-bin -c conf/odoo.conf -d odoo18 -i base --stop-after-init` |
| Start server | `.venv/bin/python odoo-18.0/odoo-bin -c conf/odoo.conf` |
| Start + reload | `.venv/bin/python odoo-18.0/odoo-bin -c conf/odoo.conf --dev=reload` |
| Install module | `.venv/bin/python odoo-18.0/odoo-bin -c conf/odoo.conf -d odoo18 -i my_module --stop-after-init` |
| Update module | `.venv/bin/python odoo-18.0/odoo-bin -c conf/odoo.conf -d odoo18 -u my_module --stop-after-init` |
| Debug logs | `.venv/bin/python odoo-18.0/odoo-bin -c conf/odoo.conf --log-level=debug` |
| List CLI options | `.venv/bin/python odoo-18.0/odoo-bin --help` |
| Check the DB | `PGPASSWORD=odoo psql -h localhost -U odoo -d odoo18 -c 'select 1'` |

---

## Troubleshooting

**`ModuleNotFoundError: No module named 'odoo'`**
Running `odoo-bin` requires the Odoo dependencies *and* the source root on the
path. The script adds it automatically — make sure you are running
`odoo-18.0/odoo-bin`, and that deps are installed in `.venv`, then run with
`.venv/bin/python`.

**`could not connect to server: Connection refused` / `FATAL: role "odoo" does not exist`**
PostgreSQL isn't running or the role is missing. See
[Database setup](#database-setup) and confirm with `pg_isready`.

**`--dev=reload` doesn't pick up changes**
Reload watches Python imports and files that Odoo reloads; brand-new files or
manifest `data` changes still need `-u <module>`. XML view changes need
`--dev=reload,xml`.

**Pylance shows `import "odoo" could not be resolved`**
Select the `.venv` interpreter and confirm
`python.analysis.extraPaths` contains `${workspaceFolder}/odoo-18.0`, then
reload the window (`Developer: Reload Window`).

**Port 8069 already in use**
Another Odoo instance is running. Stop it, or change `http_port` in
`conf/odoo.conf` and in `launch.json`.

**Breakpoints never hit when debugging**
Ensure `justMyCode` is `false`, that you attached to the correct process, and that
the request actually executes your code path.

**Fresh clone has an empty `.venv`**
The virtualenv contains no third-party packages until you install them. Run the
**pip: Install Odoo requirements** task or the `pip install -r` command above.

---

## License & credits

- **Odoo 18.0** source in `odoo-18.0/` is © Odoo S.A. and licensed under
  **LGPL-3** (see [`odoo-18.0/LICENSE`](odoo-18.0/LICENSE)).
- Your own modules in `custom_modules/` should declare their license in
  `__manifest__.py` (typically `LGPL-3`).
- This boilerplate's configuration and VS Code files are provided as-is for
  development convenience.

Verified against: Odoo 18.0 (final), Python 3.12, PostgreSQL 18, Node.js 24.

