# PHP Build Process

Complete documentation of how Railpack detects, builds, and runs PHP
applications.

**Location**: `core/providers/php/`

## Detection

The PHP provider is detected if **any** of these exist (`php.go:38-41`):

- `index.php`
- `composer.json`

**Provider Priority**: PHP is checked 1st in the detection order (highest
priority).

## PHP Version Resolution

Priority order:

1. **`composer.json` require.php** field - e.g., `"php": "^8.3"`
2. **mise.toml or .tool-versions**
3. **Default**: PHP **8.4**

Example `composer.json`:

```json
{
  "require": {
    "php": "^8.3"
  }
}
```

## Build Process

### Step 1: Install PHP Runtime

**Location**: `php.go:397-445`

**Base Image**: Uses FrankenPHP (PHP + Caddy web server)

```
dunglas/frankenphp:php<version>-bookworm
```

**Process**:

1. Detect PHP version from `composer.json`
2. Verify version exists on Docker Hub
3. Use FrankenPHP image with that PHP version

**Why FrankenPHP?**: Modern PHP application server with:
- Built-in Caddy web server
- HTTP/2 and HTTP/3 support
- High performance
- Laravel Octane compatibility

### Step 2: Install PHP Extensions

**Location**: `php.go:145-155`, `php.go:271-324`

**Auto-Detection**:

Railpack automatically installs PHP extensions based on:

1. **`composer.json` require field** - e.g., `"ext-redis": "*"`
2. **`PHP_EXTENSIONS` environment variable** - Custom extensions
3. **Laravel detection** - Auto-installs required extensions
4. **Database detection** - Based on `DB_CONNECTION` environment variable

**Laravel Extensions** (auto-installed if `artisan` exists):
- ctype, curl, dom, fileinfo, filter, hash
- mbstring, openssl, pcre, pdo, session, tokenizer, xml

**Database Extensions**:
- `DB_CONNECTION=mysql` → `pdo_mysql`
- `DB_CONNECTION=pgsql` → `pdo_pgsql`

**Redis Detection**:
- `REDIS_HOST` or `REDIS_URL` set, OR
- `CACHE_DRIVER=redis`, OR
- `composer.json` contains redis packages

**Command**:

```bash
install-php-extensions redis pdo_mysql gd ...
```

### Step 3: Install Composer Dependencies

**Location**: `php.go:157-179`

**Process**:

```bash
# Copy composer binary from official image
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Copy dependency files
COPY composer.json
COPY composer.lock
COPY artisan  # if Laravel

# Install dependencies
composer install --optimize-autoloader --no-scripts --no-interaction
```

**Flags**:
- `--optimize-autoloader` - Generate optimized autoloader
- `--no-scripts` - Skip post-install scripts
- `--no-interaction` - Non-interactive mode

**Cache**: `/opt/cache/composer` for Composer cache

**Environment Variables**:
```bash
COMPOSER_FUND=0
COMPOSER_CACHE_DIR=/opt/cache/composer
```

### Step 4: Build Assets (if Node.js detected)

**Location**: `php.go:181-235`

**When**: If `package.json` exists

**Process**:

1. Install Node.js and package manager
2. Install Node dependencies
3. Run build script (if exists)
4. Prune dev dependencies
5. For Laravel: Run artisan commands

**Laravel Build Commands**:

```bash
# Create required directories
mkdir -p storage/framework/{sessions,views,cache,testing} storage/logs bootstrap/cache
chmod -R a+rw storage

# Cache Laravel configuration
php artisan config:cache
php artisan event:cache
php artisan route:cache
php artisan view:cache
```

### Step 5: Configure Server

**Location**: `php.go:104-143`

**Configuration Files**:

1. **Caddyfile** - Web server configuration
2. **php.ini** - PHP configuration
3. **start-container.sh** - Startup script

**Caddyfile Template**:
- Serves from `/app/public` for Laravel
- Serves from `/app` for other PHP apps
- Can be overridden with custom `Caddyfile`

**Environment Variables**:
```bash
APP_ENV=production
APP_DEBUG=false
APP_LOCALE=en
LOG_CHANNEL=stderr
LOG_LEVEL=debug
SERVER_NAME=:80
PHP_INI_DIR=/usr/local/etc/php
OCTANE_SERVER=frankenphp
IS_LARAVEL=true  # if Laravel detected
```

## Laravel Detection

**Location**: `php.go:352-354`

**Detected if**: `artisan` file exists

**Features**:
- Auto-installs Laravel required PHP extensions
- Sets `RAILPACK_PHP_ROOT_DIR=/app/public`
- Runs artisan cache commands during build
- Supports Laravel Octane with FrankenPHP

## Start Command

**Location**: `php.go:99`

```bash
/start-container.sh
```

The start script:
1. Starts FrankenPHP server
2. Serves from configured root directory
3. Handles PHP requests via FastCGI or Octane

## Deploy

**Deploy Includes**:
- FrankenPHP runtime with PHP
- Installed Composer dependencies
- Built assets (if Node.js was used)
- Application source code
- Configuration files

**Note**: If no additional mise packages specified, mise is not included in
final image (to reduce size).

## Complete Build Flow Example

### Laravel Application with Vite

```
Project structure:
  composer.json (requires php ^8.3, laravel/framework)
  artisan
  package.json
  vite.config.js
```

**Build Process**:

1. **Detection**: ✅ Has `composer.json` → PHP detected
2. **Laravel Detection**: ✅ Has `artisan` → Laravel app
3. **PHP Version**: Read from `composer.json` → PHP 8.3
4. **Base Image**: `dunglas/frankenphp:php8.3-bookworm`
5. **Extensions**: Install Laravel + database extensions
6. **Composer Install**:
   ```bash
   composer install --optimize-autoloader --no-scripts
   ```
7. **Node Build**:
   ```bash
   npm ci
   npm run build
   ```
8. **Laravel Cache**:
   ```bash
   php artisan config:cache
   php artisan route:cache
   php artisan view:cache
   ```
9. **Start**: `/start-container.sh` (runs FrankenPHP on port 80)

## Environment Variable Overrides

- `PHP_EXTENSIONS` - Additional PHP extensions (comma or space separated)
- `RAILPACK_PHP_ROOT_DIR` - Override document root (default: `/app` or
  `/app/public` for Laravel)
- `DB_CONNECTION` - Database type (mysql, pgsql) for auto-extension install
- `REDIS_HOST`, `REDIS_URL` - Triggers Redis extension install
- Various Laravel-specific env vars (`APP_ENV`, `APP_DEBUG`, etc.)

## Best Practices

1. **Specify PHP version** - In `composer.json` require field
2. **Use Composer lock file** - Commit `composer.lock` for reproducibility
3. **Laravel optimization** - Artisan cache commands run automatically
4. **Environment variables** - Set `APP_KEY`, database credentials, etc.
5. **Custom extensions** - List in `composer.json` or `PHP_EXTENSIONS`

## Common Issues

### Missing PHP Extensions

- Add to `composer.json`: `"ext-<name>": "*"`
- Or set `PHP_EXTENSIONS=<name>` environment variable

### Wrong Document Root

- Laravel: Should auto-detect and use `/app/public`
- Override with `RAILPACK_PHP_ROOT_DIR=/custom/path`

### Composer Install Fails

- Ensure `composer.lock` is committed
- Check for authentication issues with private packages

### Laravel Key Missing

- Set `APP_KEY` environment variable
- Or run `php artisan key:generate` locally first

## Files Reference

- `core/providers/php/php.go` - Main provider logic
- `core/providers/php/php_test.go` - Test cases
- `core/providers/php/Caddyfile` - Default Caddyfile template
- `core/providers/php/php.ini` - PHP configuration template
- `core/providers/php/start-container.sh` - Startup script
- `examples/php-*` - Example PHP projects
