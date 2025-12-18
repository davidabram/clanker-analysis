# Go Build Process

Complete documentation of how Railpack detects, builds, and runs Go
applications.

**Location**: `core/providers/golang/`

## Detection

The Go provider is detected if **any** of these exist (`golang.go:25-27`):

- `go.mod` file
- `go.work` file (Go workspaces)
- `main.go` file

**Provider Priority**: Go is checked 2nd in the detection order (after PHP).

## Go Version Resolution

Priority order (`golang.go:163-190`):

1. **`GO_VERSION` environment variable**
2. **`go.mod` file** - Extracts version from `go X.XX` line
3. **mise.toml or .tool-versions**
4. **Default**: Go **1.25**

Example `go.mod`:

```go
module github.com/example/app

go 1.21
```

## Build Process

### Step 1: Install Go

**Location**: `golang.go:179-191`

**Process**:

1. Install Go at resolved version via mise
2. Set `GOPATH=/go` and `GOBIN=/go/bin`

**Build Dependencies** (`golang.go:193-201`):

If `CGO_ENABLED=1`:

- Installs `gcc`, `g++`, `libc6-dev` for C compilation

### Step 2: Install Dependencies

**Location**: `golang.go:117-161`

**When**: If `go.mod` or `go.work` exists

**Process**:

```bash
# For go.mod projects
COPY go.mod
COPY go.sum  # if exists

# For go.work projects
COPY go.work
COPY go.work.sum  # if exists

# Copy workspace module files
COPY <workspace>/go.mod
COPY <workspace>/go.sum  # if exists

# Download dependencies
go mod download
```

**Environment Variables**:

- `GOPATH=/go`
- `GOBIN=/go/bin`
- `CGO_ENABLED=0` (unless explicitly set to `1`)

**Cache** (`golang.go:211-213`):

- `/root/.cache/go-build` - Build cache

### Step 3: Build Binary

**Location**: `golang.go:65-115`

**Process**:

Railpack intelligently determines what to build based on project structure:

#### Build Command Selection

Priority order:

1. **`GO_WORKSPACE_MODULE` environment variable**:

   ```bash
   go build -ldflags="-w -s" -o out ./api
   ```

   Use case: Monorepos with multiple modules

2. **`GO_BIN` environment variable**:

   ```bash
   go build -ldflags="-w -s" -o out ./cmd/server
   ```

   Use case: Multiple binaries in `cmd/` directory

3. **Root `go.mod` with `.go` files in root**:

   ```bash
   go build -ldflags="-w -s" -o out
   ```

   Use case: Simple single-file projects

4. **First directory in `cmd/`**:

   ```bash
   go build -ldflags="-w -s" -o out ./cmd/<name>
   ```

   Use case: Standard Go project layout

5. **Workspace with main package**:
   Searches workspace packages for one with `main.go`
6. **Single `main.go` file**:
   ```bash
   go build -ldflags="-w -s" -o out main.go
   ```

**Flags**:

- `-ldflags="-w -s"` - Strip debug info and symbol table (smaller binary)
- `-o out` - Output binary name

**Output**: Binary named `out` in project root

### Step 4: Deploy

**Location**: `golang.go:45-62`

**Start Command**:

```bash
./out
```

**Runtime Dependencies** (`golang.go:47-51`):

- `tzdata` - Timezone data (always included)
- `libc6` - C standard library (if CGO enabled)

**Deploy Includes**:

- Built binary (`out`)
- Any other files in project root

## Go Workspaces

**Location**: `golang.go:242-260`

**Detected if**: `go.work` exists

**Features**:

- Automatically discovers all workspace modules
- Copies all `go.mod` and `go.sum` files from workspace packages
- Can build specific workspace module with `GO_WORKSPACE_MODULE`

**Workspace Discovery**:

Finds all `go.mod` files (excluding root):

```
/
  go.work
  api/
    go.mod
  frontend/
    go.mod
```

Result: Detects `api` and `frontend` as workspace packages

## CGO Support

**Detected if**: `CGO_ENABLED=1` environment variable

**Build Dependencies**:

- `gcc`
- `g++`
- `libc6-dev`

**Runtime Dependencies**:

- `libc6`

**Use Cases**:

- Projects using C libraries
- SQLite with `go-sqlite3`
- Native extensions

## Complete Build Flow Example

### Standard Go Application

```
Project structure:
  go.mod (has "go 1.21")
  cmd/
    server/
      main.go
```

**Build Process**:

1. **Detection**: ✅ Has `go.mod` → Go detected
2. **Go Version**: Read from `go.mod` → Use Go 1.21
3. **Mise Install**: Install Go 1.21
4. **Install Step**:
   ```bash
   COPY go.mod
   COPY go.sum
   go mod download
   ```
5. **Build Step**:
   ```bash
   go build -ldflags="-w -s" -o out ./cmd/server
   ```
6. **Start Command**: `./out`

### Go Workspace

```
Project structure:
  go.work
  api/
    go.mod
    main.go
  worker/
    go.mod
    main.go
```

**With `GO_WORKSPACE_MODULE=api`**:

1. **Detection**: ✅ Has `go.work` → Go detected
2. **Install Step**:
   ```bash
   COPY go.work
   COPY go.work.sum
   COPY api/go.mod
   COPY api/go.sum
   COPY worker/go.mod
   COPY worker/go.sum
   go mod download
   ```
3. **Build Step**:
   ```bash
   go build -ldflags="-w -s" -o out ./api
   ```
4. **Start Command**: `./out`

## Gin Detection

**Location**: `golang.go:226-232`

Railpack detects if project uses Gin web framework:

```bash
# Check if go.mod contains:
github.com/gin-gonic/gin
```

**Metadata**: Sets `goGin=true` for analytics

## Metadata Collection

**Location**: `golang.go:203-209`

Tracks:

- **goMod**: Has `go.mod` file
- **goWorkspace**: Has `go.work` file
- **goRootFile**: Has `.go` files in root
- **goGin**: Uses Gin framework
- **goCGO**: CGO is enabled

## Environment Variable Overrides

- `GO_VERSION` - Override Go version
- `GO_WORKSPACE_MODULE` - Specify workspace module to build
- `GO_BIN` - Specify binary in `cmd/` directory
- `CGO_ENABLED` - Enable/disable CGO (0 or 1)

## Best Practices

1. **Use `go.mod`** - Ensures reproducible builds
2. **Specify Go version** - In `go.mod` or environment variable
3. **Standard layout** - Use `cmd/<name>/main.go` for multiple binaries
4. **Vendor dependencies** - Optional, but not required (Railpack downloads)
5. **Disable CGO if possible** - Smaller binaries, faster builds

## Common Issues

### Multiple Binaries

If you have multiple binaries in `cmd/`:

- Railpack builds the first one alphabetically
- Set `GO_BIN=<name>` to specify which to build

### Workspace Module Selection

If you have a workspace with multiple modules:

- Set `GO_WORKSPACE_MODULE=<path>` to specify which to build
- Or ensure one module has `main.go` in root

### CGO Not Working

If you need CGO:

- Set `CGO_ENABLED=1` environment variable
- Railpack will install necessary C compilers

### Missing Binary

If build succeeds but `./out` doesn't exist:

- Check if project has a `main` package
- Verify the build command matches your project structure
- Use `GO_WORKSPACE_MODULE` or `GO_BIN` to specify target

## Files Reference

- `core/providers/golang/golang.go` - Main provider logic
- `core/providers/golang/golang_test.go` - Test cases
- `examples/go-*` - Example projects for various Go setups
