# Project Index: addon-grocy

**Generated**: 2026-01-20 (Updated after modernization)
**Project**: Home Assistant Community Add-on for Grocy
**Type**: Docker-based Home Assistant Add-on
**License**: MIT

## ⚡ Recent Major Update (2026-01-20)

**Modernization Complete**: Upgraded from archived base image to actively maintained base
- Base Image: `base-nodejs:0.2.5` (Alpine 3.19, ARCHIVED) → `base:19.0.0` (Alpine 3.22, Active)
- All dependencies updated to latest Alpine 3.22 packages
- See `MIGRATION.md` for upgrade instructions

---

## 📁 Project Structure

```
addon-grocy/
├── grocy/                      # Add-on container definition
│   ├── rootfs/                # Container filesystem overlay
│   │   ├── etc/               # Configuration files
│   │   │   ├── nginx/         # Web server configuration
│   │   │   ├── php82/         # PHP-FPM configuration
│   │   │   └── s6-overlay/    # Service management
│   │   ├── patches/           # Grocy patches
│   │   └── var/               # Variable data
│   ├── Dockerfile             # Container build definition
│   ├── build.yaml             # Build configuration
│   ├── config.yaml            # Add-on configuration schema
│   └── DOCS.md                # User documentation
├── .github/                   # GitHub automation
│   ├── workflows/             # CI/CD pipelines
│   └── *.md                   # Community guidelines
├── images/                    # Documentation assets
└── README.md                  # Project overview
```

---

## 🚀 Entry Points

**Container Build**: `grocy/Dockerfile`
- Base image: `ghcr.io/hassio-addons/base-nodejs:0.2.5`
- Grocy version: `v4.5.0` (from upstream)
- Entry: S6-overlay service manager

**Add-on Configuration**: `grocy/config.yaml`
- Defines Home Assistant add-on interface
- Schema validation for user options
- Ingress support enabled

**Service Management**: `grocy/rootfs/etc/s6-overlay/s6-rc.d/`
- PHP-FPM service: `php-fpm/run`
- Nginx service: `nginx/run`
- Initialization: `init-*` services

---

## 📦 Core Components

### Container Build (Dockerfile:9-70)
**Purpose**: Build Grocy PHP application in Alpine Linux container

**Key Operations**:
- Installs PHP 8.2 with required extensions
- Clones Grocy v4.5.0 from upstream
- Runs composer and yarn for dependencies
- Applies patches and cleanup

**Dependencies** (Updated 2026-01-20):
```
Base: ghcr.io/hassio-addons/base:19.0.0 (Alpine 3.22)
nginx: 1.28.0-r3
php83: 8.3.29-r0
php83-fpm, php83-gd, php83-intl, php83-ldap
php83-opcache, php83-pdo_sqlite
composer: 2.8.10-r0
git: 2.49.1-r0
nodejs/npm/yarn: latest (for Grocy frontend build)
```

### Configuration (config.yaml:1-76)
**Purpose**: Home Assistant add-on interface definition

**Options**:
- `culture`: Language/locale (30+ languages)
- `currency`: ISO 4217 currency code
- `entry_page`: Default homepage
- `features`: Enable/disable Grocy modules
- `tweaks`: Fine-tune Grocy behavior
- `ssl`: HTTPS configuration

### Web Server (rootfs/etc/nginx/)
**Configuration Files**:
- `nginx.conf`: Main nginx configuration
- `templates/ingress.gtpl`: Home Assistant Ingress mode
- `templates/direct.gtpl`: Direct access mode
- `includes/`: Shared configurations (SSL, FastCGI, MIME types)

### PHP Configuration (rootfs/etc/php82/)
**Configuration Files**:
- `conf.d/99-grocy.ini`: PHP settings for Grocy
- `templates/php-fpm.gtpl`: PHP-FPM pool configuration

### Patches (rootfs/patches/)
- `fix_braindamage.patch`: Custom patches for Grocy

---

## 🔧 Configuration Schema

### Add-on Options (config.yaml:21-45)
```yaml
Default Configuration:
  culture: en                    # English
  currency: USD                  # US Dollar
  entry_page: stock              # Stock overview
  features:                      # All features enabled
    batteries: true
    calendar: true
    chores: true
    equipment: true
    recipes: true
    shoppinglist: true
    stock: true
    tasks: true
  tweaks:                        # All tracking enabled
    chores_assignment: true
    multiple_shopping_lists: true
    stock_best_before_date_tracking: true
    stock_location_tracking: true
    stock_price_tracking: true
    stock_product_freezing: true
    stock_product_opened_tracking: true
  ssl: true                      # HTTPS enabled
  grocy_ingress_user: ""         # Default: login required
```

### Architecture Support (config.yaml:12-14)
- `aarch64`: ARM 64-bit (Raspberry Pi 4, etc.)
- `amd64`: x86 64-bit (Intel/AMD)

---

## 📚 Documentation

### User Documentation
- **README.md**: Project overview, badges, quick start
- **grocy/DOCS.md**: Installation, configuration, troubleshooting

### Community Guidelines
- **.github/CODE_OF_CONDUCT.md**: Community standards
- **.github/CONTRIBUTING.md**: Contribution guidelines
- **.github/SECURITY.md**: Security policy
- **.github/ISSUE_TEMPLATE.md**: Bug report template
- **.github/PULL_REQUEST_TEMPLATE.md**: PR template

### Configuration Reference
**Languages Supported** (config.yaml:49):
ca, cs, da, de, el_GR, en, en_GB, es, et, fi, fr, he_IL, hu, it, ja, ko_KR, lt, nl, no, pl, pt_BR, pt_PT, ro, ru, sk_SK, sl, sv_SE, ta, tr, uk, zh_CN, zh_TW

**Entry Pages** (config.yaml:51):
stock, shoppinglist, recipes, chores, tasks, batteries, equipment, calendar, mealplan

---

## 🔄 CI/CD Workflows

### GitHub Actions (.github/workflows/)
**Active Workflows** (98 total lines):
- `ci.yaml`: Continuous integration
- `deploy.yaml`: Deployment automation
- `labels.yaml`: Label management
- `lock.yaml`: Lock inactive issues/PRs
- `pr-labels.yaml`: PR label automation
- `release-drafter.yaml`: Release notes generation
- `stale.yaml`: Stale issue management

---

## 🔗 Key Dependencies

### Runtime Dependencies (Updated 2026-01-20)
- **nginx**: 1.28.0-r3 - Web server
- **php83**: 8.3.29-r0 - PHP runtime
- **composer**: 2.8.10-r0 - PHP dependency manager
- **nodejs/npm/yarn**: Latest - Frontend build tools
- **grocy**: v4.5.0 - Upstream Grocy application

### Build Dependencies (Updated 2026-01-20)
- **git**: 2.49.1-r0 - Clone upstream Grocy
- **yarn**: Node.js package manager (frontend build)
- **modclean**: Clean node_modules

### Upstream Project
- **Grocy**: https://github.com/grocy/grocy
- **Version**: v4.5.0
- **License**: MIT
- **Demo**: https://demo-en.grocy.info

---

## 📝 Quick Start

### For Users
1. Install add-on from Home Assistant
2. Configure options in `config.yaml`
3. Start add-on
4. Access via Ingress or direct port
5. Default login: `admin` / `admin`

### For Developers
```bash
# Build container
cd grocy
docker build -t addon-grocy .

# Test locally
docker run -p 80:80 addon-grocy

# Modify configuration
vim grocy/config.yaml
vim grocy/rootfs/etc/nginx/nginx.conf
```

---

## 🛡️ Security

**Default Credentials**: `admin` / `admin` (change immediately)
**SSL Support**: Yes (configurable)
**Ingress Mode**: Home Assistant authentication integration
**Security Policy**: See `.github/SECURITY.md`

---

## 📊 Project Metadata

**Repository**: https://github.com/hassio-addons/addon-grocy
**Maintainer**: Franck Nijhof (@frenck)
**Community**: Home Assistant Community Add-ons
**Support**: Discord, Community Forum, GitHub Issues
**Status**: Experimental (project-stage-shield)

**Latest Commit**: 84c3853 - Update grocy/grocy to v4.5.0
**Recent Changes**:
- Drop armv7 support (#504)
- Remove codenotary fields (#503)
- Update git to v2.43.7-r0 (#492)

---

## 🎯 What is Grocy?

**Grocy** - ERP beyond your fridge: A self-hosted groceries & household management solution

**Features**:
- Stock management with expiration tracking
- Shopping list with barcode support
- Recipe management and meal planning
- Chores scheduling and assignment
- Task management
- Battery tracking
- Equipment management
- Inventory system

**Use Case**: Home inventory management, reduce food waste, track household tasks

---

## 📏 Index Metrics

**Total Files**: ~50 files
**Documentation**: 8 markdown files
**Configuration**: 11 YAML files
**Code**: 1 Dockerfile, PHP config, Nginx config
**Workflows**: 7 GitHub Actions
**Index Size**: ~4.5 KB
**Token Reduction**: ~95% vs full codebase read

---

**Index Version**: 1.0
**Last Updated**: 2026-01-20
**Next Update**: When significant changes occur
