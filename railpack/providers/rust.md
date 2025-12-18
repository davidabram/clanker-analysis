# Rust Build Process

Complete documentation of how Railpack detects, builds, and runs Rust
applications.

**Location**: `core/providers/rust/`

## Detection

The Rust provider is detected if `Cargo.toml` exists (`rust.go:29-32`).

**Provider Priority**: Rust is checked 4th in the detection order (after
Java).

## Rust Version Resolution

Priority order (`rust.go:314-372`):

1. **`RUST_VERSION` environment variable**
2. **`rust-toolchain.toml`** - Channel or version field
3. **`rust-toolchain`** - Plain text file with version
4. **`Cargo.toml`** - `rust-version` field or `edition` field
5. **`.rust-version` or `rust-version.txt`**
6. **mise.toml or .tool-versions**
7. **Default**: Rust **1.89**

## Build Process

### Step 1: Dependency Caching

**Location**: `rust.go:144-193`

**Strategy**: Compile dependencies separately from project code for optimal
caching

**Process**:

```bash
# Copy dependency manifests
COPY Cargo.toml
COPY Cargo.lock

# Create dummy source files
mkdir -p src
echo "fn main() { }" > src/main.rs
echo "fn main() { }" > src/lib.rs  # if lib exists in Cargo.toml

# Compile dependencies only
cargo build --release

# Remove dummy artifacts
rm -rf src target/release/<app-name>*
```

**Purpose**: Dependencies rarely change, so this layer is cached

**Caches**:
- `/root/.cargo/registry` - Downloaded crates
- `/root/.cargo/git` - Git dependencies

### Step 2: Build Application

**Location**: `rust.go:196-241`

**Process**:

```bash
# Copy all source files
COPY .

# Create bin directory
mkdir -p bin

# Build
cargo build --release

# Copy binary/binaries to bin/
cp target/release/<binary> bin/
```

**Target Support** (`rust.go:135-142`, `377-380`):

If `.cargo/config.toml` specifies `target = "wasm32-wasi"`:
- Adds `rustup target add wasm32-wasi`
- Builds with `--target wasm32-wasi`
- Binary path: `target/wasm32-wasi/release/`
- Binary extension: `.wasm`

**Cache**: `target/` directory for incremental compilation

### Step 3: Binary Detection

**Location**: `rust.go:250-292`

**Single Binary**:
- If project has `src/main.rs` → binary named after package

**Multiple Binaries**:
- Checks `src/bin/*.rs` for additional binaries
- Each file becomes a binary

**Binary Selection for Start Command** (`rust.go:96-133`):

1. If only one binary: use it
2. If `RUST_BIN` environment variable set: use that binary
3. If `default-run` in `Cargo.toml`: use that binary
4. Otherwise: first binary found

## Workspace Support

**Location**: `rust.go:382-490`

**Detected if**: `Cargo.toml` has `[workspace]` section

**Process**:

1. Check `CARGO_WORKSPACE` environment variable
2. Search `default-members` for binary packages
3. Search all `members` for binary packages
4. Build specific workspace member:
   ```bash
   cargo build --release --package <workspace-member>
   ```

**Binary Detection in Workspaces**:

For each member, checks:
- Has `src/main.rs`
- Has `src/bin/` directory
- Has `[[bin]]` entries in Cargo.toml
- No `src/lib.rs` (library-only packages excluded)

## Start Command

**Location**: `rust.go:77-133`

```bash
./bin/<binary-name>
# or
./bin/<binary-name>.wasm  # for wasm32-wasi targets
```

**Workspace Start**:
```bash
./bin/<workspace-package-name>
```

## Deploy

**Location**: `rust.go:56-66`

**Deploy Includes**:
- Built binaries in `bin/` directory
- All project files (excluding `target/`)

**No runtime required**: Rust compiles to native binaries

## Complete Build Flow Example

### Single Binary Application

```
Project structure:
  Cargo.toml (name = "myapp", edition = "2021")
  src/
    main.rs
```

**Build Process**:

1. **Detection**: ✅ Has `Cargo.toml` → Rust detected
2. **Rust Version**: Edition 2021 → Use Rust 1.84.0+
3. **Install Dependencies**:
   ```bash
   cargo build --release  # with dummy src/
   ```
4. **Build Application**:
   ```bash
   cargo build --release  # with real src/
   cp target/release/myapp bin/
   ```
5. **Start Command**: `./bin/myapp`

### Workspace with Multiple Packages

```
Project structure:
  Cargo.toml ([workspace])
  api/
    Cargo.toml (name = "api")
    src/main.rs
  worker/
    Cargo.toml (name = "worker")
    src/main.rs
```

**With `CARGO_WORKSPACE=api`**:

1. **Build**:
   ```bash
   cargo build --release --package api
   ```
2. **Start Command**: `./bin/api`

## Environment Variable Overrides

- `RUST_VERSION` - Override Rust version
- `RUST_BIN` - Specify which binary to run (if multiple)
- `CARGO_WORKSPACE` - Specify workspace package to build

## Best Practices

1. **Use `Cargo.lock`** - Commit for reproducible builds
2. **Specify Rust version** - In `Cargo.toml` or `rust-toolchain.toml`
3. **Single binary** - Simplifies deployment
4. **Use `default-run`** - For projects with multiple binaries
5. **Workspace members** - Set `CARGO_WORKSPACE` explicitly

## Common Issues

### Multiple Binaries, Wrong One Runs

- Set `RUST_BIN=<name>` environment variable
- Or set `default-run` in `Cargo.toml`

### Workspace Build Fails

- Set `CARGO_WORKSPACE=<package>` to specify which to build
- Ensure package has a binary target (not library-only)

### WASM Target Not Working

- Ensure `.cargo/config.toml` has correct target specification
- Binary will have `.wasm` extension

## Files Reference

- `core/providers/rust/rust.go` - Main provider logic
- `core/providers/rust/rust_test.go` - Test cases
- `examples/rust-*` - Example Rust projects
