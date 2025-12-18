# Deno Build Process

Complete documentation of how Railpack detects, builds, and runs Deno
applications.

**Location**: `core/providers/deno/`

## Detection

The Deno provider is detected if **any** of these exist (`deno.go:27-29`):

- `deno.json`
- `deno.jsonc`

**Provider Priority**: Deno is checked 8th in the detection order (after
Python).

## Deno Version Resolution

Priority order (`deno.go:84-91`):

1. **`DENO_VERSION` environment variable**
2. **mise.toml or .tool-versions**
3. **Default**: Deno **2**

## Build Process

### Step 1: Install Deno

**Location**: `deno.go:84-92`

**Process**:

1. Install Deno at resolved version via mise

### Step 2: Find Main File

**Location**: `deno.go:32-34`, `deno.go:94-107`

**Detection Order**:

1. `main.ts`
2. `main.js`
3. `main.mjs`
4. `main.mts`
5. First `.ts` file found
6. First `.js` file found  
7. First `.mjs` file found
8. First `.mts` file found

**Stored** during Initialize phase for use in build and start commands.

### Step 3: Cache Dependencies

**Location**: `deno.go:73-82`

**When**: If main file was found

**Process**:

```bash
# Copy all source files
COPY .

# Cache dependencies for main file
deno cache <main-file>
```

**Cache Location**: `/root/.cache`

**Purpose**: Pre-downloads and caches all dependencies for faster runtime
startup.

### Step 4: Deploy

**Location**: `deno.go:45-53`

**Start Command** (`deno.go:65-71`):

```bash
deno run --allow-all <main-file>
```

**Deploy Includes**:

- Deno runtime
- All source files
- Cached dependencies (`/root/.cache`)

## Complete Build Flow Example

```
Project structure:
  deno.json
  main.ts
  lib/
    utils.ts
```

**Build Process**:

1. **Detection**: ✅ Has `deno.json` → Deno detected
2. **Deno Version**: Default → Deno 2
3. **Main File**: Found `main.ts`
4. **Mise Install**: Install Deno 2
5. **Build Step**:
   ```bash
   deno cache main.ts
   ```
6. **Start Command**: `deno run --allow-all main.ts`

## Permissions

**Default**: `--allow-all` grants all permissions

**Why**: Simplifies deployment, most production apps need broad permissions

**Custom Permissions**: Can override with manual start command in config or
Procfile

## Environment Variable Overrides

- `DENO_VERSION` - Override Deno version

## Best Practices

1. **Name entry file `main.ts`** - Makes detection explicit
2. **Use `deno.json`** - Configure tasks, imports, and permissions
3. **Lock dependencies** - Use `deno.lock` for reproducible builds
4. **Specify Deno version** - In `.tool-versions` or environment variable

## Common Issues

### No Main File Found

If Railpack can't find a main file:

- Create `main.ts` or `main.js`
- Or set a custom start command

### Permission Denied Errors

All permissions are granted by default (`--allow-all`), so this is unlikely.

If you need specific permissions:

- Set custom start command with specific `--allow-*` flags
- Use Procfile or `START_CMD` environment variable

## Files Reference

- `core/providers/deno/deno.go` - Main provider logic
- `core/providers/deno/deno_test.go` - Test cases
- `examples/deno-*` - Example Deno projects
