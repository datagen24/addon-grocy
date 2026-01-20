# Quick Migration Reference

**For users upgrading from old Grocy add-on without SQL export.**

## 🚀 5-Minute Migration

### Prerequisites
- SSH access to Home Assistant
- 10-20 minutes of downtime acceptable
- Old Grocy add-on currently running

---

## Step-by-Step Commands

### 1. Stop Old Add-on
```bash
# Via Home Assistant UI
Settings → Add-ons → Grocy → Stop

# Or via SSH (if Terminal add-on installed)
ha addons stop a0d7b954_grocy
```

### 2. Backup Database (SSH Required)

```bash
# SSH into Home Assistant
ssh root@homeassistant.local

# Create backup directory
mkdir -p /backup/grocy_$(date +%Y%m%d)

# Copy entire data directory
cp -r /data/addons/a0d7b954_grocy /backup/grocy_$(date +%Y%m%d)/

# Verify backup
ls -lh /backup/grocy_$(date +%Y%m%d)/a0d7b954_grocy/grocy.db
# Should show your database file with size (e.g., 2.5M)
```

### 3. Install New Add-on

```bash
# Via Home Assistant UI
Settings → Add-ons → Add-on Store
Search: "Grocy"
Click: Install (on the NEW version)

# ⚠️ IMPORTANT: Do NOT click "Start" yet!
```

### 4. Copy Database to New Instance

```bash
# Still via SSH

# Navigate to data directory
cd /data/addons/a0d7b954_grocy

# If new add-on created default files, back them up
if [ -f grocy.db ]; then
    mv grocy.db grocy_default.db.bak
fi
if [ -d storage ]; then
    mv storage storage_default.bak
fi

# Copy your database from backup
cp /backup/grocy_$(date +%Y%m%d)/a0d7b954_grocy/grocy.db ./

# Copy storage directory (images, files)
cp -r /backup/grocy_$(date +%Y%m%d)/a0d7b954_grocy/storage ./

# Set correct permissions
chown -R root:root /data/addons/a0d7b954_grocy
chmod 644 grocy.db
chmod -R 755 storage

# Verify files are in place
ls -lh grocy.db
ls -lh storage/
```

### 5. Start New Add-on

```bash
# Via Home Assistant UI
Settings → Add-ons → Grocy (new) → Start

# Or via SSH
ha addons start a0d7b954_grocy
```

### 6. Verify Migration

```bash
# Check logs for errors
ha addons logs a0d7b954_grocy

# Look for:
# ✓ "s6-rc: info: service init-grocy successfully started"
# ✓ "s6-rc: info: service php-fpm successfully started"
# ✓ "s6-rc: info: service nginx successfully started"
```

**Access Grocy**:
- Home Assistant → Grocy add-on → "OPEN WEB UI"
- Login with your existing credentials
- Verify your data is present

---

## ✅ Verification Checklist

Quick checks after migration:

- [ ] Add-on started without errors
- [ ] Can access Grocy web interface
- [ ] Login works (your existing credentials)
- [ ] Stock item count matches old instance
- [ ] Product images display
- [ ] Shopping lists present
- [ ] Can create/edit items

---

## 🆘 Quick Troubleshooting

### "Database is locked" error
```bash
ha addons stop a0d7b954_grocy
cd /data/addons/a0d7b954_grocy
rm -f grocy.db-shm grocy.db-wal
ha addons start a0d7b954_grocy
```

### "Permission denied" error
```bash
cd /data/addons/a0d7b954_grocy
chown root:root grocy.db
chmod 644 grocy.db
ha addons restart a0d7b954_grocy
```

### Images not showing
```bash
cd /data/addons/a0d7b954_grocy
chmod -R 755 storage
ha addons restart a0d7b954_grocy
```

### Add-on won't start
```bash
# Check logs for specific error
ha addons logs a0d7b954_grocy

# Common fix: rebuild add-on
Settings → Add-ons → Grocy → ⋮ → Rebuild
```

---

## 🔄 Rollback (If Something Goes Wrong)

```bash
# Stop new add-on
ha addons stop a0d7b954_grocy

# Restore original database
cd /data/addons/a0d7b954_grocy
rm -rf grocy.db storage
cp /backup/grocy_YYYYMMDD/a0d7b954_grocy/grocy.db ./
cp -r /backup/grocy_YYYYMMDD/a0d7b954_grocy/storage ./

# Uninstall new add-on
Settings → Add-ons → Grocy → Uninstall

# Reinstall old version (use Home Assistant backup)
Settings → System → Backups → [Previous backup] → Restore
```

---

## 📋 Complete One-Liner Script

Copy and paste this entire block into SSH terminal:

```bash
#!/bin/bash
set -e
echo "=== Grocy Quick Migration ==="

# Stop old add-on
echo "Stopping old add-on..."
ha addons stop a0d7b954_grocy
sleep 5

# Create backup
BACKUP_DIR="/backup/grocy_$(date +%Y%m%d_%H%M%S)"
echo "Creating backup: $BACKUP_DIR"
mkdir -p $BACKUP_DIR
cp -r /data/addons/a0d7b954_grocy $BACKUP_DIR/

# Verify backup
if [ -f "$BACKUP_DIR/a0d7b954_grocy/grocy.db" ]; then
    echo "✓ Backup created successfully"
    DB_SIZE=$(du -h "$BACKUP_DIR/a0d7b954_grocy/grocy.db" | cut -f1)
    echo "  Database size: $DB_SIZE"
else
    echo "✗ Backup failed!"
    exit 1
fi

echo ""
echo "=== NEXT STEPS ==="
echo "1. Install new Grocy add-on (DON'T START IT)"
echo "2. Run this restore command:"
echo ""
echo "   cd /data/addons/a0d7b954_grocy && \\"
echo "   rm -rf grocy.db storage && \\"
echo "   cp $BACKUP_DIR/a0d7b954_grocy/grocy.db ./ && \\"
echo "   cp -r $BACKUP_DIR/a0d7b954_grocy/storage ./ && \\"
echo "   chown -R root:root . && \\"
echo "   chmod 644 grocy.db && \\"
echo "   chmod -R 755 storage && \\"
echo "   ha addons start a0d7b954_grocy"
echo ""
echo "Backup location: $BACKUP_DIR"
```

---

## 📞 Need Help?

**More Details**: See [DATABASE_MIGRATION.md](DATABASE_MIGRATION.md) for comprehensive guide

**Issues**: https://github.com/hassio-addons/addon-grocy/issues

**Community**: https://community.home-assistant.io/t/home-assistant-community-add-on-grocy/112422

---

**Time Required**: 10-15 minutes
**Difficulty**: Intermediate (requires SSH access)
**Risk Level**: Low (with backup)
