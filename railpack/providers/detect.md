# How Railpack Detects Project Types

Railpack uses a provider-based architecture to automatically detect and build
different types of applications. This document explains how project type
detection works.

## Language-Specific Documentation

For detailed build processes of each language, see:

- [Python Build Process](python.md) - Complete Python build documentation
- [Node.js Build Process](node.md) - Complete Node.js build documentation  
- [Go Build Process](golang.md) - Complete Go build documentation
- [Java Build Process](java.md) - Complete Java build documentation
- [Rust Build Process](rust.md) - Complete Rust build documentation
- [PHP Build Process](php.md) - Complete PHP build documentation
- [Deno Build Process](deno.md) - Complete Deno build documentation
- [.NET Build Process](dotnet.md) - Complete .NET/C# build documentation

## Detection Flow

When Railpack analyzes a project, it follows this process:

1. **Sequential Provider Checking**: Providers are checked in a specific order
   (see Provider Priority below)
2. **First Match Wins**: The first provider whose `Detect()` method returns
   `true` is selected
3. **Provider Initialization**: Once selected, the provider's `Initialize()`
   method is called
4. **Build Plan Generation**: The provider generates a build plan specific to
   that project type

## Provider Priority

Providers are checked in this order (defined in
`core/providers/provider.go:31-48`):

1. PHP
2. Go
3. Java
4. Rust
5. Ruby
6. Elixir
7. Python
8. Deno
9. .NET
10. Node
11. Gleam
12. C++
13. Staticfile
14. Shell

The order matters because some file patterns could match multiple providers.
More specific providers are checked before more general ones.

## Detection Methods by Provider

### PHP (`core/providers/php/php.go:38-41`)

Detects if:

- `index.php` exists, OR
- `composer.json` exists

### Go (`core/providers/golang/golang.go:25-27`)

Detects if:

- `go.mod` exists, OR
- `go.work` exists, OR
- `main.go` exists

### Java (`core/providers/java/java.go`)

Detects if:

- `pom.xml` exists (Maven), OR
- `build.gradle` or `build.gradle.kts` exists (Gradle)

### Rust (`core/providers/rust/rust.go:29-32`)

Detects if:

- `Cargo.toml` exists

### Ruby (`core/providers/ruby/ruby.go:30-33`)

Detects if:

- `Gemfile` exists

### Elixir (`core/providers/elixir/elixir.go`)

Detects if:

- `mix.exs` exists

### Python (`core/providers/python/python.go:33-40`)

Detects if:

- `main.py` exists, OR
- `requirements.txt` exists, OR
- `pyproject.toml` exists, OR
- `Pipfile` exists

### Deno (`core/providers/deno/deno.go`)

Detects if:

- `deno.json` exists, OR
- `deno.jsonc` exists

### .NET (`core/providers/dotnet/dotnet.go`)

Detects if:

- Any `.csproj` or `.fsproj` file exists

### Node (`core/providers/node/node.go:60-62`)

Detects if:

- `package.json` exists

### Gleam (`core/providers/gleam/gleam.go`)

Detects if:

- `gleam.toml` exists

### C++ (`core/providers/cpp/cpp.go`)

Detects if:

- `CMakeLists.txt` exists (CMake), OR
- `meson.build` exists (Meson)

### Staticfile (`core/providers/staticfile/staticfile.go:45-52`)

Detects if:

- `STATIC_FILE_ROOT` environment variable is set, OR
- `Staticfile` config file exists with a `root` field, OR
- `public/` directory exists, OR
- `index.html` exists in the project root

### Shell (`core/providers/shell/shell.go`)

This is a fallback provider that always matches. It allows running custom
commands for projects that don't fit other categories.

## Manual Provider Override

You can override automatic detection in two ways:

1. **Config File** (`railpack.json`):
   ```json
   {
     "provider": "node"
   }
   ```

2. **Environment Variable**:
   Set the provider via the config file path or manually specify steps

## Provider Interface

All providers implement the `Provider` interface
(`core/providers/provider.go:22-29`):

```go
type Provider interface {
    Name() string
    Detect(ctx *GenerateContext) (bool, error)
    Initialize(ctx *GenerateContext) error
    Plan(ctx *GenerateContext) error
    CleansePlan(buildPlan *BuildPlan)
    StartCommandHelp() string
}
```

### Key Methods

- `Detect()`: Returns true if the project matches this provider's criteria
- `Initialize()`: Performs any setup needed after detection
- `Plan()`: Generates the build plan (install dependencies, build, deploy)
- `CleansePlan()`: Post-processes the build plan
- `StartCommandHelp()`: Returns help text for configuring the start command

## Detection Best Practices

### For Users

1. **Use standard project structures**: Providers look for conventional files
   like `package.json`, `Cargo.toml`, etc.
2. **One project type per directory**: Mixing multiple project types can lead
   to unexpected detection results
3. **Use manual override when needed**: If detection picks the wrong provider,
   specify it explicitly

### For Contributors

1. **Order matters**: Place more specific providers before general ones
2. **Keep detection simple**: Check for canonical files that uniquely identify
   a project type
3. **Avoid false positives**: Don't match files that might exist in unrelated
   projects
4. **Document edge cases**: Note any special detection logic in provider
   comments

## Example Detection Scenarios

### Monorepo with Multiple Languages

If you have a repository with both Python and Node:

```
/
  api/
    requirements.txt
    main.py
  frontend/
    package.json
    src/
```

Railpack will detect **Python** because:

- `main.py` exists at the root
- Python is checked before Node in the priority order

To build the Node frontend, either:

- Move to the `frontend/` directory before running Railpack
- Use a `railpack.json` config to specify the provider

### Full-Stack Application

For a Rails app with Node assets:

```
/
  Gemfile
  config.ru
  package.json
  app/
    javascript/
```

Railpack will detect **Ruby** because:

- `Gemfile` exists
- Ruby is checked before Node

The Ruby provider will automatically detect and handle Node dependencies if
`package.json` exists (see `core/providers/ruby/ruby.go:44-50`).

## How Railpack Determines the Start Command

After detecting the project type and building it, Railpack needs to determine
how to start the application. Each provider has its own logic for finding the
appropriate start command.

### Start Command Priority

The start command is determined in this order:

1. **Procfile**: If a `Procfile` exists, Railpack checks for `web`, `worker`,
   or any other process type (`core/providers/procfile/procfile.go:12-42`)
2. **Manual Override**: `START_CMD` environment variable or `railpack.json`
   config
3. **Provider-Specific Detection**: Each provider has custom logic to find the
   start command

### Start Command Detection by Provider

#### Node (`core/providers/node/node.go:158-171`)

Checks in order:

1. **"start" script** in `package.json`:
   ```json
   {
     "scripts": {
       "start": "node server.js"
     }
   }
   ```
2. **"main" field** in `package.json`:
   ```json
   {
     "main": "src/index.js"
   }
   ```
3. **index.js or index.ts** in project root
4. **Nuxt default**: For Nuxt apps, uses `node .output/server/index.mjs`

**SPA Detection**: If the app is a Single Page Application (React, Vue,
Angular, etc.), Railpack serves it with Caddy instead of using Node at runtime.

#### Python (`core/providers/python/python.go:96-123`)

Checks in order:

1. **Django**: If `manage.py` exists and Django is installed:
   - Finds the WSGI app name from settings
   - Runs: `python manage.py migrate && gunicorn --bind
     0.0.0.0:${PORT:-8000} <app>:application`
2. **FastHTML**: If `python-fasthtml` dependency exists:
   - Runs: `uvicorn main:app --host 0.0.0.0 --port ${PORT:-8000}`
3. **FastAPI**: If `fastapi` dependency exists:
   - Runs: `uvicorn main:app --host 0.0.0.0 --port ${PORT:-8000}`
4. **Flask**: If `flask` and `gunicorn` dependencies exist:
   - Runs: `gunicorn --bind 0.0.0.0:${PORT:-8000} main:app`
5. **Generic Python**: Looks for main file in order: `main.py`, `app.py`,
   `bot.py`, `hello.py`, `server.py`
   - Runs: `python <file>.py`

#### Go (`core/providers/golang/golang.go:45`)

Simple and deterministic:

- Compiles to a binary named `out`
- Start command: `./out`

The build process handles finding the right Go module or command to compile.

#### Rust (`core/providers/rust/rust.go:77-133`)

Determines the binary to run:

1. **Workspace**: If `CARGO_WORKSPACE` env var is set, uses that binary
2. **Single binary**: If only one binary in `Cargo.toml`, uses it
3. **Multiple binaries**: Uses `RUST_BIN` env var or `default-run` from
   `Cargo.toml`
4. **WASM**: For wasm32-wasi targets, adds `.wasm` extension

Start command: `./bin/<binary-name>`

#### Ruby (`core/providers/ruby/ruby.go:117-142`)

Checks in order:

1. **Rails**: If `config/application.rb` contains `Rails::Application`:
   - With `rails` file: `bundle exec rails server -b 0.0.0.0 -p
     ${PORT:-3000}`
   - With `bin/rails`: `bundle exec bin/rails server -b 0.0.0.0 -p
     ${PORT:-3000} -e $RAILS_ENV`
2. **Legacy Rails**: If `config/environment.rb` and `script/` exist:
   - `bundle exec ruby script/server -p ${PORT:-3000}`
3. **Rack**: If `config.ru` exists:
   - `bundle exec rackup config.ru -p ${PORT:-3000}`
4. **Rake**: If `Rakefile` exists:
   - `bundle exec rake`

#### PHP (`core/providers/php/php.go:99`)

Uses a custom start script:

- Start command: `/start-container.sh`
- The script runs FrankenPHP (a PHP application server) with Caddy
- Automatically handles Laravel apps with Octane when detected

#### Java (`core/providers/java/java.go:83-93`)

Runs the built JAR file:

1. **Gradle**:
   - `java $JAVA_OPTS -jar <gradle-port-config> */build/libs/*jar` (excludes
     `*-plain.jar`)
2. **Maven**:
   - `java <maven-port-config> $JAVA_OPTS -jar target/*jar`

#### Elixir (`core/providers/elixir/elixir.go:83-86`)

Uses Mix releases:

- Finds the app name from `mix.exs`
- Start command: `/app/_build/prod/rel/<app-name>/bin/<app-name> start`

#### Deno (`core/providers/deno/deno.go:65-71`)

Finds the main file:

1. Looks for `main.ts`, `main.js`, `main.mjs`, or `main.mts`
2. Falls back to first `.ts`, `.js`, `.mjs`, or `.mts` file found

Start command: `deno run --allow-all <main-file>`

#### .NET (`core/providers/dotnet/dotnet.go:79-87`)

Runs the published binary:

1. Finds the `.csproj` file
2. Uses the project name as the binary name
3. Start command: `./out/<project-name>`

#### Staticfile (`core/providers/staticfile/staticfile.go:74`)

Serves static files with Caddy:

- Start command: `caddy run --config Caddyfile --adapter caddyfile 2>&1`

### Procfile Support

All providers support Procfile for overriding the start command
(`core/providers/procfile/procfile.go`).

The Procfile provider runs **after** the language provider and can override
its start command. It looks for:

1. **web** process type (highest priority)
2. **worker** process type
3. Any other process type (first one found)

Example Procfile:

```yaml
web: gunicorn myapp:app
worker: celery -A myapp worker
```

### Start Command Override

You can override the start command in multiple ways (in order of priority):

1. **Procfile**: Add a `Procfile` with a `web` process
2. **Environment Variable**: Set `START_CMD`
3. **Config File**: Set `deploy.startCmd` in `railpack.json`

Example `railpack.json`:

```json
{
  "deploy": {
    "startCmd": "npm run start:prod"
  }
}
```

### Port Configuration

Most providers configure apps to listen on the `PORT` environment variable:

- **Default ports** vary by framework (e.g., 3000 for Rails/Node, 8000 for
  Python)
- **Runtime**: Railway sets `PORT` dynamically
- **Best practice**: Use `${PORT:-3000}` to fall back to a default for local
  development

### Missing Start Command

If Railpack cannot determine a start command, the build will succeed but you
must manually set the start command before deployment.

To see what start command would be used:

```bash
railpack build --show-plan
```

This shows the complete build plan including the detected start command.
