# .NET Build Process

Complete documentation of how Railpack detects, builds, and runs .NET/C#
applications.

**Location**: `core/providers/dotnet/`

## Detection

The .NET provider is detected if **any** `.csproj` or `.fsproj` file exists
(`dotnet.go:30-32`).

**Provider Priority**: .NET is checked 9th in the detection order (after
Deno).

## .NET Version Resolution

Priority order (`dotnet.go:118-135`):

1. **`DOTNET_VERSION` environment variable**
2. **`global.json`** - Reads `sdk.version` field
3. **mise.toml or .tool-versions**
4. **Default**: .NET **8.0**

Example `global.json`:

```json
{
  "sdk": {
    "version": "8.0.100"
  }
}
```

## Build Process

### Step 1: Install .NET SDK

**Location**: `dotnet.go:118-135`

**Process**:

1. Install .NET SDK at resolved version via mise
2. Set environment variables

**Environment Variables** (`dotnet.go:108-116`):

```bash
ASPNETCORE_ENVIRONMENT=production
ASPNETCORE_URLS=http://0.0.0.0:3000
DOTNET_CLI_TELEMETRY_OPTOUT=1
DOTNET_ROOT=/mise/installs/dotnet/<version>
```

### Step 2: Restore Dependencies

**Location**: `dotnet.go:89-98`

**Process**:

```bash
# Copy project files
COPY nuget.config  # if exists
COPY *.csproj
COPY global.json  # if exists

# Create NuGet packages directory
mkdir -p /root/.nuget/packages

# Restore dependencies
dotnet restore
```

**Cache**: `/root/.nuget/packages` for NuGet packages

### Step 3: Build and Publish

**Location**: `dotnet.go:100-106`

**Process**:

```bash
# Copy all source files
COPY .

# Build and publish
dotnet publish --no-restore -c Release -o out
```

**Flags**:

- `--no-restore` - Don't restore again (already done)
- `-c Release` - Build in Release configuration
- `-o out` - Output to `out/` directory

**Output**: Published application in `out/` directory

### Step 4: Deploy

**Location**: `dotnet.go:56-66`

**Start Command** (`dotnet.go:79-87`):

```bash
./out/<project-name>
```

**Project Name**: Extracted from `.csproj` filename (e.g., `myapp.csproj` →
`myapp`)

**Runtime Dependencies**:

- `libicu-dev` - Required for internationalization

**Deploy Includes**:

- .NET runtime (from mise)
- Published application (`out/` directory)

## Complete Build Flow Example

```
Project structure:
  Program.cs
  myapp.csproj
  global.json (has "version": "8.0.100")
```

**Build Process**:

1. **Detection**: ✅ Has `myapp.csproj` → .NET detected
2. **.NET Version**: Read from `global.json` → Use .NET 8.0.100
3. **Mise Install**: Install .NET SDK 8.0.100
4. **Restore Step**:
   ```bash
   dotnet restore
   ```
5. **Build Step**:
   ```bash
   dotnet publish --no-restore -c Release -o out
   ```
6. **Start Command**: `./out/myapp`

## Project Types

### ASP.NET Core Web Apps

**Detected by**: Presence of `.csproj` with `Microsoft.AspNetCore.*` packages

**Environment**:

- `ASPNETCORE_ENVIRONMENT=production`
- `ASPNETCORE_URLS=http://0.0.0.0:3000`

**Port**: Listens on port 3000 by default, but should respect `PORT`
environment variable

### Console Applications

**Standard .NET console apps work automatically**

**Start**: Runs the published executable

### Multiple Projects

If you have multiple `.csproj` files:

- Railpack uses the first one found alphabetically
- Publish all projects with `dotnet publish` (builds entire solution)

## NuGet Configuration

**Location**: `dotnet.go:92`

If `nuget.config` exists, it's included in the restore step.

Use case: Private NuGet feeds, custom package sources

Example `nuget.config`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
    <add key="myget" value="https://myget.org/F/myfeed/api/v3/index.json" />
  </packageSources>
</configuration>
```

## Environment Variable Overrides

- `DOTNET_VERSION` - Override .NET SDK version
- `ASPNETCORE_ENVIRONMENT` - Set to development, staging, or production
- `ASPNETCORE_URLS` - Override listen URLs

## Best Practices

1. **Use `global.json`** - Pin SDK version for reproducibility
2. **Include `nuget.config`** - For private package sources
3. **Use Release configuration** - Optimized builds
4. **Respect PORT environment variable** - For dynamic port assignment
5. **Keep SDK version current** - Security and performance

## Common Issues

### Wrong Project Built

If you have multiple projects:

- Ensure your main project is the first `.csproj` alphabetically
- Or restructure to have only the deployable project at root

### Missing Dependencies

If restore fails:

- Check `nuget.config` for correct package sources
- Ensure all packages are available in configured sources
- Check for authentication issues with private feeds

### Internationalization Errors

- `libicu-dev` is automatically installed
- If you still have ICU errors, check .NET version compatibility

## Files Reference

- `core/providers/dotnet/dotnet.go` - Main provider logic
- `core/providers/dotnet/dotnet_test.go` - Test cases
- `examples/dotnet-*` - Example .NET projects
