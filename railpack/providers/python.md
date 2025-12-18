# Python Build Process

Complete documentation of how Railpack detects, builds, and runs Python
applications.

**Location**: `core/providers/python/`

## Detection

The Python provider is detected if **any** of these files exist
(`python.go:33-40`):

- `main.py`
- `requirements.txt`
- `pyproject.toml`
- `Pipfile`

**Provider Priority**: Python is checked 7th in the detection order (after
PHP, Go, Java, Rust, Ruby, Elixir).

## Package Manager Detection

Railpack supports **5 different Python package managers** with a clear
priority order (`python.go:54-68`):

1. **pip** - If `requirements.txt` exists
2. **uv** - If `pyproject.toml` AND `uv.lock` exist
3. **Poetry** - If `pyproject.toml` AND `poetry.lock` exist
4. **PDM** - If `pyproject.toml` AND `pdm.lock` exist
5. **Pipenv** - If `Pipfile` exists

**Key Insight**: Lock files take precedence! If you have `pyproject.toml` +
`poetry.lock`, it uses Poetry even if you also have a `requirements.txt`.

## Python Version Resolution

Python version is determined in this priority order (`python.go:305-318`):

1. **`PYTHON_VERSION` environment variable** - Highest priority
2. **`runtime.txt` file** - Common for Heroku-style deployments
3. **`Pipfile`** - Checks `python_full_version` or `python_version` fields
4. **`mise.toml` or `.tool-versions`** - If present in project
5. **Default**: Python **3.13**

Example `runtime.txt`:

```
python-3.11.5
```

## Build Process

### Step 1: Install Mise Packages

**Location**: `python.go:305-359`

**Purpose**: Install Python interpreter and package manager tools

**Process**:

1. Install Python at resolved version
2. Detect which package managers are needed
3. Install package manager tools:
   - **Poetry/PDM/Pipenv**: Installed via `pipx` (using `uv` backend)
   - **uv**: Installed directly as a mise package

**Environment Variables**:

- `MISE_PYTHON_COMPILE=false` - Disables Python compilation to avoid package
  incompatibility
- `MISE_PIPX_UVX=true` - Uses `uv` as the backend for `pipx` (faster, more
  reliable)

**Example packages installed**:

- For Poetry project: `python`, `pipx`, `pipx:poetry`
- For uv project: `python`, `uv`
- For pip project: just `python`

### Step 2: Install Build Dependencies

**Location**: `python.go:280-303`

**Purpose**: Install system packages needed to **build** Python packages with
C extensions

**Detection Logic**:

- Scans dependency files for packages that need compilation
- Special detection for databases (PostgreSQL, MySQL)

**Examples** (`python.go:543-545`):

- `pycairo` → installs `libcairo2-dev`
- `psycopg2` or Django with PostgreSQL → installs `libpq-dev`
- `mysqlclient` or Django with MySQL → installs `default-libmysqlclient-dev`

**Note**: These are build-time only. Runtime packages are installed
separately.

### Step 3: Install Dependencies

Different process for each package manager:

#### pip (`python.go:241-261`)

```bash
# Create virtual environment
python -m venv /app/.venv

# Add venv to PATH
export PATH="/app/.venv/bin:$PATH"

# Copy requirements.txt (optimized for caching)
COPY requirements.txt

# Install dependencies
pip install -r requirements.txt
```

**Cache**: `/opt/pip-cache` for pip downloads

#### uv (`python.go:145-170`)

```bash
# Copy only dependency files (not full source)
COPY pyproject.toml
COPY uv.lock

# Install dependencies (but NOT the project itself yet)
uv sync --locked --no-dev --no-install-project
```

**Why `--no-install-project`?**

- Optimizes Docker layer caching
- Source code isn't needed for dependencies
- Project is installed later in build step

**Special handling**: If the project uses uv workspaces or local path
dependencies (`file://`), it copies ALL files (`python.go:372-426`).

**Environment Variables**:

- `UV_COMPILE_BYTECODE=1` - Pre-compile Python files
- `UV_LINK_MODE=copy` - Copy packages instead of symlinking
- `UV_PYTHON_DOWNLOADS=never` - Don't download Python (we use mise)
- `VIRTUAL_ENV=/app/.venv` - Use project-local venv

**Cache**: `/opt/uv-cache` for uv's cache

#### Poetry (`python.go:221-239`)

```bash
# Copy dependency files
COPY pyproject.toml
COPY poetry.lock

# Install only production dependencies, skip installing project root
poetry install --no-interaction --no-ansi --only main --no-root
```

**Key flags**:

- `--only main` - Skip dev dependencies
- `--no-root` - Don't install the project itself (optimization)

**Environment Variables**:

- `POETRY_VIRTUALENVS_IN_PROJECT=true` - Create venv in `/app/.venv`

#### PDM (`python.go:203-219`)

```bash
# Copy dependency files
COPY pyproject.toml
COPY pdm.lock

# Install dependencies
pdm install --check --prod --no-editable
```

**Key flags**:

- `--check` - Verify lock file is up to date
- `--prod` - Skip dev dependencies
- `--no-editable` - Install packages normally (not in editable mode)

#### Pipenv (`python.go:172-201`)

```bash
# If Pipfile.lock exists (recommended):
COPY Pipfile
COPY Pipfile.lock
pipenv install --deploy --ignore-pipfile

# If no lock file (not recommended for production):
COPY Pipfile
pipenv install --skip-lock
```

**Environment Variables**:

- `PIPENV_VENV_IN_PROJECT=1` - Create venv in project
- `PIPENV_IGNORE_VIRTUALENVS=1` - Force new venv creation

### Step 4: Build Step

**Location**: `python.go:51-73`

For **uv only**:

```bash
# Now install the project (with source code available)
uv sync --locked --no-dev --no-editable
```

**Why?** The install step only installed dependencies. Now we install the
actual project.

For **other package managers**: Build step does nothing (project installed
during install step).

### Step 5: Install Runtime Dependencies

**Location**: `python.go:263-278`

**Purpose**: Install system packages needed to **run** Python packages

**Examples** (`python.go:547-552`):

- `pycairo` → installs `libcairo2` (runtime library)
- `pdf2image` → installs `poppler-utils`
- `pydub` → installs `ffmpeg`
- `pymovie` → installs `ffmpeg`, Qt5 libraries
- `psycopg2` → installs `libpq5` (PostgreSQL client)
- `mysqlclient` → installs `default-mysql-client`

### Step 6: Deploy Layer

**Location**: `python.go:84-91`

**What gets included in final image**:

1. **Mise layer** - Python interpreter and tools
2. **Virtual environment** - All installed packages (`/app/.venv`)
3. **Application source** - All files except `.venv` itself

**Environment Variables** (`python.go:361-370`):

```bash
PYTHONFAULTHANDLER=1          # Show tracebacks on crashes
PYTHONUNBUFFERED=1            # Don't buffer stdout/stderr
PYTHONHASHSEED=random         # Randomize hash seed
PYTHONDONTWRITEBYTECODE=1     # Don't create .pyc files
PIP_DISABLE_PIP_VERSION_CHECK=1
PIP_DEFAULT_TIMEOUT=100
```

## Start Command Detection

**Location**: `python.go:96-123`

Detection order (first match wins):

### 1. Django

**Location**: `django.go:37-45`

**Detected if**: `manage.py` exists AND `django` in dependencies

**Process**:

1. Find Django app name by searching for `WSGI_APPLICATION` in Python files
2. Or use `DJANGO_APP_NAME` environment variable

**Start command**:

```bash
python manage.py migrate && gunicorn --bind 0.0.0.0:${PORT:-8000} myapp.wsgi:application
```

**Note**: Auto-runs migrations before starting!

### 2. FastHTML

**Location**: `python.go:106-108`

**Detected if**:

- `python-fasthtml` dependency exists
- Has main file (main.py, app.py, etc.)
- Has `uvicorn` dependency

**Start command**:

```bash
uvicorn main:app --host 0.0.0.0 --port ${PORT:-8000}
```

### 3. FastAPI

**Location**: `python.go:110-112`

**Detected if**:

- `fastapi` dependency exists
- Has main file
- Has `uvicorn` dependency

**Start command**:

```bash
uvicorn main:app --host 0.0.0.0 --port ${PORT:-8000}
```

### 4. Flask

**Location**: `python.go:114-116`

**Detected if**:

- `flask` dependency exists
- Has main file
- Has `gunicorn` dependency

**Start command**:

```bash
gunicorn --bind 0.0.0.0:${PORT:-8000} main:app
```

### 5. Generic Python

**Location**: `python.go:118-120, 125-132`

**Fallback**: Look for these files in order:

1. `main.py`
2. `app.py`
3. `bot.py`
4. `hello.py`
5. `server.py`

**Start command**:

```bash
python <filename>
```

## Special Features

### Database Detection

**Location**: `python.go:428-438`

**PostgreSQL** - detected if:

- `psycopg2`, `psycopg2-binary`, or `psycopg` in dependencies, OR
- Django project with `django.db.backends.postgresql` in any `.py` file

**MySQL** - detected if:

- `mysqlclient` in dependencies, OR
- Django project with `django.db.backends.mysql` in any `.py` file

### Workspace Support

**Location**: `python.go:398-426`

For **uv workspaces** or projects with local path dependencies:

- Detects `file://` in `requirements.txt`
- Detects `path =` in `pyproject.toml`
- Detects `[tool.uv.workspace]` in `pyproject.toml`

**Behavior**: Copies ALL source files during install (not just dependency
files)

### Dependency Detection

**Location**: `python.go:459-469`

The `usesDep()` function checks if a dependency exists by:

1. Searching `requirements.txt` for the package name
2. Searching `pyproject.toml` for the package name
3. Searching `Pipfile` for the package name

**Note**: Uses case-insensitive string matching (not perfect, but works for
most cases)

## Complete Build Flow Example

Django project with uv:

```
Project structure:
  manage.py
  myapp/
    settings.py (has WSGI_APPLICATION = "myapp.wsgi.application")
  pyproject.toml
  uv.lock
  .python-version (contains "3.11")
```

**Build Process**:

1. **Detection**: ✅ Has `pyproject.toml` → Python detected
2. **Package Manager**: ✅ Has `uv.lock` → Use uv
3. **Python Version**: Read `.python-version` → Use Python 3.11
4. **Mise Install**: Install Python 3.11 + uv
5. **Builder Deps**: Scan dependencies, install `libpq-dev` for psycopg2
6. **Install Step**:
   ```bash
   uv sync --locked --no-dev --no-install-project
   ```
7. **Build Step**:
   ```bash
   uv sync --locked --no-dev --no-editable
   ```
8. **Runtime Deps**: Install `libpq5` for PostgreSQL
9. **Start Command**:
   ```bash
   python manage.py migrate && gunicorn --bind 0.0.0.0:${PORT:-8000} myapp.wsgi:application
   ```

## Environment Variable Overrides

- `PYTHON_VERSION` - Override Python version
- `DJANGO_APP_NAME` - Override Django app name detection
- Secrets with prefixes: `PYTHON_`, `PIP_`, `PIPX_`, `UV_`, `PDM_`, `POETRY_`
  are passed to install step

## Best Practices

1. **Always use lock files** - Ensures reproducible builds
2. **Specify Python version** - Use `.python-version` or `runtime.txt`
3. **Use uv for new projects** - Fastest and most reliable
4. **Include gunicorn/uvicorn** - Required for production web servers
5. **Set `PORT` environment variable** - Apps should respect `${PORT}`

## Common Issues

### Missing Start Command

If no framework is detected and no main file exists, you need to:

- Set `START_CMD` environment variable, OR
- Add a `Procfile` with a `web` process, OR
- Create one of the expected main files (`main.py`, `app.py`, etc.)

### Build Failures with Native Extensions

If packages like `psycopg2` fail to build:

- Railpack should auto-detect and install build dependencies
- If not, report as a bug with the package name

### Wrong Package Manager Detected

If Railpack picks the wrong package manager:

- Remove unused lock files (e.g., old `poetry.lock` if you switched to uv)
- The first matching lock file wins in priority order

## Files Reference

- `core/providers/python/python.go` - Main provider logic
- `core/providers/python/django.go` - Django-specific detection
- `core/providers/python/python_test.go` - Test cases
- `examples/python-*` - Example projects for each package manager
