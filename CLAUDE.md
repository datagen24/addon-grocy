# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Home Assistant Community Add-on** that packages [Grocy](https://grocy.info/) (a self-hosted groceries & household management solution) as a containerized add-on for Home Assistant. The add-on is built using Docker with Alpine Linux, nginx, PHP-FPM, and the S6-overlay process supervisor.

**Key Characteristics**:
- Not a typical application repository - this is an **add-on packaging project**
- Contains minimal custom code - primarily configuration and integration glue
- Upstream Grocy source is cloned during Docker build (not committed to this repo)
- CI/CD and deployment handled by external workflows (`hassio-addons/workflows`)

## Architecture

### Three-Layer Structure

1. **Add-on Definition Layer** (`grocy/config.yaml`, `grocy/build.yaml`)
   - Defines Home Assistant add-on metadata, options schema, and architecture support
   - User-configurable options: language, currency, feature toggles, SSL settings

2. **Container Build Layer** (`grocy/Dockerfile`)
   - Builds on `ghcr.io/hassio-addons/base:19.0.0` (Alpine 3.22)
   - Clones upstream Grocy v4.5.0 during build
   - Installs PHP 8.3, nginx, and dependencies
   - Applies custom patches to fix Home Assistant Ingress compatibility

3. **Runtime Configuration Layer** (`grocy/rootfs/`)
   - Overlay filesystem mounted into container at runtime
   - S6-overlay service definitions for nginx, PHP-FPM, and Grocy initialization
   - Dynamic configuration generation using `tempio` templates

### Dual-Mode Operation

The add-on operates in **two modes simultaneously**:

**Ingress Mode** (Primary):
- Accessed via Home Assistant UI (proxy through Home Assistant)
- nginx listens on port 8099 (internal)
- PHP-FPM pool "ingress" on port 9002
- `GROCY_BASE_URL` environment variable set to ingress path
- Optional auto-login via `grocy_ingress_user` configuration
- Access restricted to Home Assistant supervisor IP (172.30.32.2)

**Direct Mode** (Optional):
- Accessed via add-on's exposed port 80
- nginx listens on port 80 (configurable SSL)
- PHP-FPM pool "www" on port 9001
- Requires user authentication (default: admin/admin)
- Only enabled if user maps port 80 in add-on configuration

### Service Initialization Flow

S6-overlay services start in dependency order:

```
base (s6-overlay base services)
  ↓
init-grocy → Copies Grocy data to /data/grocy (persistent storage)
          → Symlinks /var/www/grocy/data to /data/grocy
          → Applies fix_braindamage.patch for URL handling
  ↓
init-php-fpm → Generates PHP-FPM pool configs from templates
             → Creates "ingress" pool (always)
             → Creates "www" pool (if port 80 enabled)
  ↓
init-nginx → Generates nginx server configs from templates
           → Creates ingress.conf (always)
           → Creates direct.conf (if port 80 enabled)
  ↓
php-fpm → Starts PHP-FPM with generated pools
  ↓
nginx → Starts nginx with generated server blocks
```

### Critical Patch: fix_braindamage.patch

This patch fixes Grocy's JavaScript URL handling for Home Assistant Ingress:

**Problem**: Grocy's BaseUrl calculation fails when accessed via Ingress proxy path
**Solution**: Detects Ingress context and constructs BaseUrl from window.location.origin
**Location**: Applied to `/var/www/grocy/views/layout/default.blade.php` during init-grocy

## Recent Changes (2026-01-20)

**Major Modernization Update**:
- **Base Image**: Migrated from archived `base-nodejs:0.2.5` to active `base:19.0.0`
  - Rationale: base-nodejs was archived Feb 2025, Grocy doesn't actually use Node.js (PHP app)
  - Alpine Linux: 3.19 → 3.22 (latest stable with security updates)
- **Dependencies Updated**:
  - nginx: 1.24.0-r16 → 1.28.0-r3
  - PHP: 8.2.26-r0 → 8.3.29-r0 (major version upgrade)
  - Composer: 2.7.7-r0 → 2.8.10-r0
  - Git: 2.43.7-r0 → 2.49.1-r0
  - Patch: 2.7.6-r10 → 2.8-r0
- **Node.js/Yarn**: Added explicit installation (needed for Grocy frontend build)
- **Grocy**: Already at latest v4.5.0 (released 2025-03-28)
- **Migration Guide**: See `MIGRATION.md` for backup/upgrade procedures

## Development Workflow

### Building the Add-on Locally

```bash
# Build for your architecture
cd grocy
docker build -t addon-grocy .

# Build for specific architecture
docker build --build-arg BUILD_FROM=ghcr.io/hassio-addons/base:19.0.0 -t addon-grocy .
```

### Testing Locally (Without Home Assistant)

```bash
# Run in direct mode only
docker run -p 80:80 addon-grocy

# Access at http://localhost
# Default login: admin / admin
```

### Modifying Configuration

**Add-on Options** (`grocy/config.yaml`):
- Modify `options:` section to change defaults
- Update `schema:` section when adding new configuration options
- Changes require add-on restart in Home Assistant

**Nginx Configuration**:
- Edit templates: `grocy/rootfs/etc/nginx/templates/*.gtpl`
- Templates use Go template syntax (processed by `tempio`)
- Variables injected from `bashio::var.json` in init scripts

**PHP Configuration**:
- Edit: `grocy/rootfs/etc/php82/conf.d/99-grocy.ini`
- Edit pool template: `grocy/rootfs/etc/php82/templates/php-fpm.gtpl`

**S6 Service Definitions**:
- Service folders: `grocy/rootfs/etc/s6-overlay/s6-rc.d/{service-name}/`
- `run` script: Main service execution (must not fork)
- `finish` script: Cleanup on service stop
- `dependencies.d/`: Service startup dependencies

### Updating Grocy Version

1. Update `GROCY_VERSION` ARG in `grocy/Dockerfile` (e.g., `v4.5.0`)
2. Test build and verify compatibility
3. Update version dependencies if Grocy requires newer PHP/nginx
4. Test both Ingress and Direct modes
5. Verify patches still apply cleanly

### Updating Dependencies

Dependencies are pinned to specific versions in Dockerfile:
- nginx: `1.24.0-r16`
- php82: `8.2.26-r0`
- composer: `2.7.7-r0`

**To update**:
1. Modify version in Dockerfile `apk add` commands
2. Test build and runtime functionality
3. Update all related PHP extensions to match PHP version
4. Verify Grocy compatibility with PHP version changes

## CI/CD Pipeline

**Important**: This repository uses **external workflow definitions** from `hassio-addons/workflows`.

### CI Workflow (`.github/workflows/ci.yaml`)
- Triggered on: push, pull_request, workflow_dispatch
- Delegates to: `hassio-addons/workflows/.github/workflows/addon-ci.yaml@main`
- Runs linting, builds multi-arch images, security scanning

### Deploy Workflow (`.github/workflows/deploy.yaml`)
- Triggered on: releases, CI completion on main branch
- Delegates to: `hassio-addons/workflows/.github/workflows/addon-deploy.yaml@main`
- Publishes images to GitHub Container Registry
- Updates add-on repository metadata

**You cannot modify these workflows directly** - they are maintained in the `hassio-addons/workflows` repository.

## Data Persistence

**User Data Location**: `/data/grocy/` (persistent across container restarts)

**Initialization Logic**:
- First run: Copies `/var/www/grocy/data` template to `/data/grocy`
- Subsequent runs: Symlinks `/var/www/grocy/data` → `/data/grocy`
- Database: `/data/grocy/grocy.db` (SQLite)
- User uploads: `/data/grocy/storage/`
- View cache: `/data/grocy/viewcache/`

**Important**: Never modify files in `/var/www/grocy/data` directly - changes will be lost on restart.

## Configuration Template System

The add-on uses **tempio** (Go template engine) to generate runtime configuration from templates.

**Pattern**:
```bash
bashio::var.json \
    key1 "value1" \
    key2 "^value2" \    # ^ prefix = boolean conversion
    | tempio \
        -template /path/to/template.gtpl \
        -out /path/to/output.conf
```

**Example** (from init-nginx):
```bash
bashio::var.json \
    interface "$(bashio::addon.ip_address)" \
    grocy_user "$(bashio::config 'grocy_ingress_user')" \
    | tempio \
        -template /etc/nginx/templates/ingress.gtpl \
        -out /etc/nginx/servers/ingress.conf
```

Template variables accessed with `{{ .variable_name }}`.

## Common Issues and Solutions

### Ingress Mode Not Working
- Check `fix_braindamage.patch` applied successfully in logs
- Verify `GROCY_BASE_URL` environment variable set in PHP-FPM pool
- Ensure nginx ingress.conf allows supervisor IP (172.30.32.2)

### Direct Mode SSL Issues
- Verify SSL certificate files exist in `/ssl/` directory
- Check `certfile` and `keyfile` configuration match actual filenames
- Ensure `ssl: true` in add-on configuration

### Database Locked Errors
- Check `/data/grocy/` permissions (nginx:nginx ownership)
- Verify SQLite not accessed by multiple processes
- Check PHP-FPM pool settings (max_children limit)

### Grocy UI Shows Wrong URLs
- Indicates `fix_braindamage.patch` failed to apply
- Check patch still compatible with Grocy version
- Review patch application logs in init-grocy output

## Contributing

Before submitting changes:

1. Discuss significant changes via GitHub issue first
2. Test both Ingress and Direct modes thoroughly
3. Verify multi-architecture builds (aarch64, amd64)
4. Follow existing patterns for configuration and services
5. Update this CLAUDE.md if adding new architectural components

Pull requests require sign-off from two reviewers before merging.
