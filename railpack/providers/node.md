# Node.js Build Process

Complete documentation of how Railpack detects, builds, and runs Node.js
applications.

**Location**: `core/providers/node/`

## Detection

The Node provider is detected if `package.json` exists (`node.go:60-62`).

**Provider Priority**: Node is checked 10th in the detection order (after PHP,
Go, Java, Rust, Ruby, Elixir, Python, Deno, .NET).

## Package Manager Detection

Railpack supports **5 different Node package managers** with automatic
detection (`node.go:394-442`):

### Detection Order

1. **packageManager field** in `package.json` (highest priority)
2. **Lock files**:
   - `pnpm-lock.yaml` → pnpm
   - `bun.lockb` or `bun.lock` → Bun
   - `.yarnrc.yml` or `.yarnrc.yaml` → Yarn Berry (2+)
   - `yarn.lock` → Yarn 1
3. **engines field** in `package.json` (fallback)
4. **Default**: npm

### Package Manager Types

- **npm** - Node's default package manager
- **pnpm** - Fast, disk space efficient package manager
- **Bun** - All-in-one toolkit with built-in package manager
- **Yarn 1** - Classic Yarn
- **Yarn Berry** - Modern Yarn (2, 3, 4+)

## Version Resolution

### Node.js Version

Priority order (`node.go:286-316`):

1. **`NODE_VERSION` environment variable**
2. **`package.json` engines.node** field
3. **`.nvmrc` file**
4. **`.node-version` file**
5. **mise.toml or .tool-versions**
6. **Default**: Node.js **22**

### Package Manager Versions

Each package manager version is resolved from (`package_manager.go:256-331`):

1. **`package.json` packageManager field** (e.g., `"packageManager":
   "pnpm@9.0.0"`)
2. **`package.json` engines field** (e.g., `"engines": {"pnpm": "9"}`)
3. **Lock file version** (for pnpm, detected from lockfileVersion)
4. **mise.toml or .tool-versions**
5. **Defaults**:
   - pnpm: 9
   - Yarn 1: 1
   - Yarn Berry: 2
   - Bun: latest

### Corepack Support

If `packageManager` field exists in `package.json`, Railpack uses Corepack
instead of mise for package manager installation (`node.go:386-388,
260-274`):

```json
{
  "packageManager": "pnpm@9.15.0"
}
```

**Environment Variable**: `MISE_NODE_COREPACK=true`

## Build Process

### Step 1: Install Mise Packages

**Location**: `node.go:285-359`

**Process**:

1. Install Node.js at resolved version
2. Install Bun if needed (for Bun projects or scripts using `bun`/`bunx`)
3. Install package manager via corepack or mise
4. Add `libatomic1` apt package (required for Node.js 25+)

**Special Cases**:

- **Bun as package manager**: If Node isn't needed in final image but needed
  for native modules during install, includes Node for build steps only
- **Corepack**: Sets `MISE_NODE_COREPACK=true` to enable corepack support

### Step 2: Install Dependencies

**Location**: `node.go:248-283`, `package_manager.go:48-80`

**Smart Copy Strategy**:

Railpack optimizes Docker layer caching by only copying necessary files:

**If project has lifecycle scripts** (`preinstall`, `postinstall`, `prepare`)
or local file dependencies:

- Copies ALL source files
- Needed for tools like `patch-package`

**Otherwise** (most projects):

- Only copies dependency-related files (`package_manager.go:224-254`):
  - `package.json`, `package-lock.json`
  - `pnpm-workspace.yaml`, `pnpm-lock.yaml`
  - `yarn.lock`, `.yarnrc.yml`, `.npmrc`
  - `bun.lockb`, `bun.lock`
  - `.node-version`, `.nvmrc`
  - `patches/`, `.pnpm-patches/`
  - `prisma/`
  - Can add custom patterns via `NODE_INSTALL_PATTERNS` env var

**Install Commands** (`package_manager.go:100-137`):

```bash
# npm
npm ci  # if package-lock.json exists
npm install  # otherwise

# pnpm
pnpm add -g node-gyp  # only if not using corepack
pnpm install --frozen-lockfile --prefer-offline

# Bun
bun install --frozen-lockfile

# Yarn 1
yarn install --frozen-lockfile

# Yarn Berry
yarn install --check-cache
```

**Caches** (`package_manager.go:82-98`):

- **npm**: `/root/.npm`
- **pnpm**: `/root/.local/share/pnpm/store/v3`
- **Bun**: `/root/.bun/install/cache`
- **Yarn 1**: `/usr/local/share/.cache/yarn` (locked cache)
- **Yarn Berry**: `/app/.yarn/cache`

**Environment Variables** (`node.go:361-379`):

```bash
NODE_ENV=production
NPM_CONFIG_PRODUCTION=false  # Install devDeps for build
NPM_CONFIG_UPDATE_NOTIFIER=false
NPM_CONFIG_FUND=false
CI=true
```

**Secrets**: Exposes secrets with prefixes: `NODE_`, `NPM_`, `BUN_`, `PNPM_`,
`YARN_`, `CI_`

### Step 3: Prune Dependencies (Optional)

**Location**: `node.go:238-246`, `package_manager.go:139-195`

**When**: Only if `PRUNE_DEPS=true` environment variable is set

**Purpose**: Remove development dependencies to reduce image size

**Commands**:

```bash
# npm
npm prune --omit=dev --ignore-scripts

# pnpm (8.15.6+)
pnpm prune --prod --ignore-scripts

# Bun (no native prune support)
rm -rf node_modules && bun install --production --ignore-scripts

# Yarn 1
yarn install --production=true

# Yarn Berry (2, 4+)
yarn workspaces focus --production --all

# Yarn 3 (no prune support)
yarn install --check-cache
```

**Custom Command**: Set `NODE_PRUNE_CMD` to override default pruning behavior

### Step 4: Build Step

**Location**: `node.go:173-188`

**When**: If `build` script exists in `package.json`

**Command**:

```bash
npm run build  # or equivalent for other package managers
```

**Framework-Specific Variables**:

- **Next.js**: Sets `NEXT_TELEMETRY_DISABLED=1`
- **Astro** (non-SPA): Adds Astro-specific environment variables

**Caches** (`node.go:209-235`):

Framework-specific build caches to speed up rebuilds:

- **Next.js**: `.next/cache` (per package in monorepos)
- **Remix**: `.cache`
- **Vite**: `node_modules/.vite`
- **Astro**: `node_modules/.astro`
- **React Router**: `.react-router`
- **General**: `node_modules/.cache`

### Step 5: Deploy Strategy

Two different strategies based on project type:

#### A. SPA (Single Page Application)

**Location**: `spa.go:20-42`, `spa.go:64-123`

**Detected if**:

- Is a Vite, Astro (SPA), Create React App, Angular, or React Router SPA
  project
- Has identifiable output directory
- No custom start command defined
- `NO_SPA` is not set to true

**Process**:

1. Build the static assets
2. Install Caddy web server
3. Generate Caddyfile for serving static files
4. Serve from output directory

**Output Directories** (`spa.go:125-143`):

- **Vite**: `dist/` (or from `vite.config`)
- **Astro**: `dist/` (or from `astro.config`)
- **CRA**: `build/`
- **Angular**: `dist/<project-name>`
- **React Router**: `.output/public/`
- **Custom**: Set `SPA_OUTPUT_DIR` environment variable

**Start Command**:

```bash
caddy run --config /Caddyfile --adapter caddyfile 2>&1
```

**Deploy Includes**:

- Caddy binary
- Caddyfile
- Built static files from output directory

#### B. Server-Side Application

**Location**: `node.go:96-143`

**Deploy Includes**:

- **Mise layer**: Node.js runtime and tools
- **node_modules**: Installed dependencies (or pruned if `PRUNE_DEPS=true`)
- **Build output**: All files except `node_modules` and `.yarn`
- **Corepack**: If using corepack, includes `/opt/corepack`

**Special: Puppeteer Support** (`node.go:115-118`):

Auto-installs Chromium dependencies if `puppeteer` is detected:

- `xvfb`, `gconf-service`, `libasound2`, `libatk1.0-0`, etc. (20+ packages)

## Start Command Detection

**Location**: `node.go:158-171`

Priority order:

1. **`start` script** in `package.json`:
   ```json
   {
     "scripts": {
       "start": "node server.js"
     }
   }
   ```
   Command: `npm run start` (or equivalent)
2. **`main` field** in `package.json`:
   ```json
   {
     "main": "src/index.js"
   }
   ```
   Command: `node src/index.js`
3. **`index.js` or `index.ts`** in project root
   Command: `node index.js`
4. **Nuxt default** (if Nuxt detected):
   Command: `node .output/server/index.mjs`
5. **Empty** - No start command (requires manual configuration)

## Framework Detection

### Next.js

**Detected if**: `next` dependency exists

**Features**:

- Sets `NEXT_TELEMETRY_DISABLED=1`
- Caches `.next/cache`

### Nuxt

**Detected if**: `nuxt` dependency exists

**Default Start**: `node .output/server/index.mjs`

### Remix

**Detected if**: `@remix-run/node` dependency exists

**Features**:

- Caches `.cache` directory

### Astro

**Detected if**: `astro` dependency exists

**SPA Detection** (`astro.go`):

- Checks `astro.config` for `output: 'static'` or `output: 'hybrid'`
- Checks for `@astrojs/node` adapter (indicates SSR)

**Features**:

- Caches `node_modules/.astro`
- Sets Astro-specific environment variables for SSR

### Vite

**Detected if**: `vite` dependency exists

**SPA Detection** (`vite.go`):

- Checks `vite.config` for `plugins` array
- If contains `react()` or `vue()` without SSR plugins → SPA

**Features**:

- Caches `node_modules/.vite`

### Create React App (CRA)

**Detected if** (`cra.go`):

- `react-scripts` dependency exists

**Features**:

- Output directory: `build/`
- Served as SPA

### Angular

**Detected if** (`angular.go`):

- `@angular/core` dependency exists

**Features**:

- Output directory: `dist/<project-name>` (reads from `angular.json`)
- Served as SPA
- Default start command: `ng serve` (if not SPA)

### React Router

**Detected if** (`react_router.go`):

- `react-router` dependency exists
- Has `react-router.config.ts`

**SPA Detection**:

- Checks config for `ssr: false` or missing `ssr` property

**Features**:

- Caches `.react-router`
- Output directory: `.output/public/` (for SPA)

### TanStack Start

**Detected if**: `@tanstack/react-start` dependency exists

## Workspace Support

**Location**: `workspace.go`

**Detected if**:

- **npm**: `workspaces` field in root `package.json`
- **pnpm**: `pnpm-workspace.yaml` exists
- **Yarn**: `workspaces` field in root `package.json`
- **Bun**: `workspaces` field in root `package.json`

**Features**:

- Automatically installs all workspace dependencies
- Framework-specific caches created per package
- Handles monorepo structures

## Special Features

### Bun Detection

**Location**: `node.go:499-543`

Bun is required if:

- Package manager is Bun, OR
- Any script contains `bun` or `bunx` command (regex match), OR
- Start command (from config) contains `bun` or `bunx`

**Behavior**:

- Installs Bun runtime
- If Bun is package manager but Node is needed (for native modules), includes
  Node in build steps

### Custom Install Patterns

**Environment Variable**: `NODE_INSTALL_PATTERNS`

Add custom file patterns to copy during install:

```bash
NODE_INSTALL_PATTERNS="schema.prisma migrations"
```

### Metadata Collection

**Location**: `node.go:468-477`

Tracks:

- **nodeRuntime**: next, nuxt, remix, tanstack-start, vite, react-router,
  bun, or node
- **nodeSPAFramework**: astro, vite, cra, angular, react-router, or static
- **nodePackageManager**: npm, pnpm, bun, yarn1, or yarnberry
- **nodeIsSPA**: true/false
- **nodeUsesCorepack**: true/false

## Complete Build Flow Examples

### Example 1: Next.js with pnpm

```
Project structure:
  package.json (has "packageManager": "pnpm@9.0.0")
  pnpm-lock.yaml
  .nvmrc (contains "20")
  app/
    page.tsx
```

**Build Process**:

1. **Detection**: ✅ Has `package.json` → Node detected
2. **Package Manager**: ✅ Has `packageManager` field → Use pnpm with corepack
3. **Node Version**: Read `.nvmrc` → Use Node 20
4. **Mise Install**: Install Node 20, enable corepack
5. **Install Step**:
   ```bash
   corepack enable
   corepack prepare pnpm@9.0.0 --activate
   pnpm install --frozen-lockfile --prefer-offline
   ```
6. **Build Step**:
   ```bash
   NEXT_TELEMETRY_DISABLED=1 pnpm run build
   ```
7. **Deploy**: Server-side (not SPA)
8. **Start Command**: `pnpm run start` (from package.json)

### Example 2: Vite React SPA

```
Project structure:
  package.json
  package-lock.json
  vite.config.ts (React plugin, no SSR)
  src/
    App.tsx
```

**Build Process**:

1. **Detection**: ✅ Has `package.json` → Node detected
2. **Package Manager**: ✅ Has `package-lock.json` → Use npm
3. **Install**: `npm ci`
4. **Build**: `npm run build` → outputs to `dist/`
5. **SPA Detection**: ✅ Vite + React plugin + no SSR
6. **Deploy Strategy**: Static site with Caddy
7. **Start Command**: `caddy run --config /Caddyfile --adapter caddyfile 2>&1`

## Environment Variable Overrides

- `NODE_VERSION` - Override Node.js version
- `BUN_VERSION` - Override Bun version
- `PRUNE_DEPS=true` - Enable dependency pruning
- `NO_SPA=true` - Disable SPA detection
- `SPA_OUTPUT_DIR` - Override output directory for SPAs
- `NODE_PRUNE_CMD` - Custom prune command
- `NODE_INSTALL_PATTERNS` - Additional file patterns to copy during install

## Best Practices

1. **Use `packageManager` field** - Ensures exact version
2. **Commit lock files** - Ensures reproducible builds
3. **Specify Node version** - Use `.nvmrc` or `engines.node`
4. **Include `start` script** - Makes start command explicit
5. **Set `PORT` environment variable** - Apps should respect `${PORT}`
6. **Consider pruning** - Set `PRUNE_DEPS=true` for smaller images

## Common Issues

### Wrong Package Manager Detected

- Railpack uses packageManager field first, then lock files
- Remove old lock files if you switched package managers
- Be explicit with `packageManager` field

### SPA vs SSR Confusion

- Railpack auto-detects based on framework configuration
- Use `NO_SPA=true` to force server-side rendering
- Use `SPA_OUTPUT_DIR` to force SPA mode

### Missing Start Command

- Add `start` script to `package.json`
- Or add `main` field pointing to entry file
- Or create `index.js` in root

### Workspace/Monorepo Issues

- Ensure workspace configuration files are present
- All workspace packages are automatically discovered
- Build caches are created per-package

## Files Reference

- `core/providers/node/node.go` - Main provider logic
- `core/providers/node/package_manager.go` - Package manager handling
- `core/providers/node/spa.go` - SPA detection and deployment
- `core/providers/node/workspace.go` - Monorepo support
- `core/providers/node/vite.go` - Vite-specific detection
- `core/providers/node/astro.go` - Astro-specific detection
- `core/providers/node/angular.go` - Angular-specific detection
- `core/providers/node/cra.go` - Create React App detection
- `core/providers/node/react_router.go` - React Router detection
- `examples/node-*` - Example projects for various frameworks
