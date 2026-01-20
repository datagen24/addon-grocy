# Database Migration: Old Instance → New Instance

This guide covers migrating your Grocy database from an old add-on version (without SQL export) to the modernized version.

## Scenario

You have:
- ✅ Old Grocy add-on running with your data
- ✅ New modernized add-on built (not yet started)
- ❌ No SQL export feature in old version

**Goal**: Extract database from old instance, inject into new instance before first start.

---

## Method 1: Direct Database Copy (Recommended)

### Step 1: Stop Old Add-on and Backup

```bash
# Via Home Assistant UI
Settings → Add-ons → Grocy (old) → Stop

# Or via SSH
ha addons stop a0d7b954_grocy
```

**Why stop?**: Prevents database corruption from active writes during copy.

### Step 2: Locate Database File

```bash
# SSH into Home Assistant
ssh root@homeassistant.local  # or your HA IP

# Find the database
cd /mnt/data/supervisor/addons/data/a0d7b954_grocy
ls -lh grocy.db

# Expected output:
# -rw-r--r-- 1 root root 2.5M Jan 20 15:00 grocy.db
```

**Database Location**: `/mnt/data/supervisor/addons/data/a0d7b954_grocy/grocy.db`

### Step 3: Create Backup Copy

```bash
# Create timestamped backup
cp grocy.db grocy_backup_$(date +%Y%m%d_%H%M%S).db

# Verify backup
ls -lh grocy_backup_*.db

# Optional: Copy to safe location
cp grocy.db /backup/grocy_migration_$(date +%Y%m%d).db
```

### Step 4: Copy Database to New Add-on Data Directory

**Option A: If new add-on not yet installed**

```bash
# Install new add-on first (don't start it yet)
# Settings → Add-ons → Add-on Store → Grocy (new) → Install
# IMPORTANT: Do NOT start the add-on

# Wait for installation to complete, then:
cd /mnt/data/supervisor/addons/data/a0d7b954_grocy

# The new add-on will create default structure on first start
# We'll copy our database before that happens
```

**Option B: If new add-on already ran once (and created default DB)**

```bash
# Stop new add-on
ha addons stop a0d7b954_grocy

# Navigate to data directory
cd /mnt/data/supervisor/addons/data/a0d7b954_grocy

# Backup default database (if exists)
if [ -f grocy.db ]; then
    mv grocy.db grocy_default_backup.db
fi

# Copy your database from backup
cp /backup/grocy_migration_YYYYMMDD.db grocy.db

# Or copy directly from old backup
cp grocy_backup_YYYYMMDD_HHMMSS.db grocy.db
```

### Step 5: Copy Storage Directory (Images/Files)

```bash
# Still in /mnt/data/supervisor/addons/data/a0d7b954_grocy/

# Backup existing storage (if new add-on created one)
if [ -d storage ]; then
    mv storage storage_default_backup
fi

# Copy storage from backup
cp -r /backup/grocy_migration_YYYYMMDD/a0d7b954_grocy/storage ./

# Verify files copied
ls -lh storage/productpictures/
```

### Step 6: Set Correct Permissions

```bash
# Ensure correct ownership
chown -R root:root /mnt/data/supervisor/addons/data/a0d7b954_grocy

# Set correct permissions
chmod 755 /mnt/data/supervisor/addons/data/a0d7b954_grocy
chmod 644 /mnt/data/supervisor/addons/data/a0d7b954_grocy/grocy.db
chmod -R 755 /mnt/data/supervisor/addons/data/a0d7b954_grocy/storage
```

### Step 7: Start New Add-on

```bash
# Via Home Assistant UI
Settings → Add-ons → Grocy (new) → Start

# Or via SSH
ha addons start a0d7b954_grocy
```

### Step 8: Verify Migration

**Check Logs**:
```bash
ha addons logs a0d7b954_grocy
```

**Access Grocy**:
- Open Home Assistant → Grocy add-on
- Login with existing credentials
- Verify data:
  - Stock items present
  - Product images display
  - Shopping lists intact
  - Recipes available

---

## Method 2: File-Based Transfer (If SSH Not Available)

### Step 1: Use Home Assistant File Editor Add-on

```bash
# Install File Editor add-on if not already installed
Settings → Add-ons → Add-on Store → File Editor → Install

# Enable "Show in sidebar"
```

### Step 2: Stop Old Grocy Add-on

```bash
Settings → Add-ons → Grocy (old) → Stop
```

### Step 3: Download Database via Terminal Add-on

```bash
# Install Terminal & SSH add-on if not already installed
Settings → Add-ons → Add-on Store → Terminal & SSH → Install

# Open Terminal and run:
cd /mnt/data/supervisor/addons/data/a0d7b954_grocy
cp grocy.db /config/www/grocy_backup_$(date +%Y%m%d).db
cp -r storage /config/www/grocy_storage_backup
```

### Step 4: Download Files via Web Browser

Access: `http://homeassistant.local:8123/local/grocy_backup_YYYYMMDD.db`

This downloads the database to your computer.

### Step 5: Upload to New Instance

```bash
# After installing new add-on (don't start yet)
# Upload database back to /config/www/

# Then via Terminal:
cd /mnt/data/supervisor/addons/data/a0d7b954_grocy
cp /config/www/grocy_backup_YYYYMMDD.db grocy.db
cp -r /config/www/grocy_storage_backup storage

# Set permissions
chown -R root:root /mnt/data/supervisor/addons/data/a0d7b954_grocy
chmod 644 grocy.db
```

---

## Method 3: Samba/SMB Share (If Enabled)

### Step 1: Enable Samba Share

```bash
# Install Samba share add-on if not already installed
Settings → Add-ons → Add-on Store → Samba share → Install → Start
```

### Step 2: Access from Computer

**Windows**:
```
\\homeassistant\addons\a0d7b954_grocy\
```

**Mac**:
```
smb://homeassistant/addons/a0d7b954_grocy/
```

**Linux**:
```bash
smbclient //homeassistant/addons -U username
cd a0d7b954_grocy
get grocy.db
mget storage/*
```

### Step 3: Stop Old Add-on

```bash
Settings → Add-ons → Grocy (old) → Stop
```

### Step 4: Copy Files via Network

Copy `grocy.db` and `storage/` directory to your computer for safekeeping.

### Step 5: Install New Add-on

```bash
Settings → Add-ons → Add-on Store → Grocy (new) → Install
# Do NOT start yet
```

### Step 6: Copy Database to New Instance

Via Samba share, copy your backed up files:
- `grocy.db` → `\\homeassistant\addons\a0d7b954_grocy\grocy.db`
- `storage\` → `\\homeassistant\addons\a0d7b954_grocy\storage\`

---

## Method 4: Database Dump via SQLite3 (If Available)

If you can access the old container:

```bash
# Access old container
docker exec -it addon_a0d7b954_grocy /bin/bash

# Install sqlite3 (if not present)
apk add sqlite

# Dump database
cd /data/grocy
sqlite3 grocy.db .dump > /config/grocy_dump_$(date +%Y%m%d).sql

# Exit container
exit

# Now the SQL dump is in /config/ and accessible
```

**Restore in new instance**:
```bash
# Access new container
docker exec -it addon_a0d7b954_grocy /bin/bash

# Restore from dump
cd /data/grocy
sqlite3 grocy.db < /config/grocy_dump_YYYYMMDD.sql
```

---

## Verification Checklist

After migration, verify:

### Database Integrity
```bash
# Check database is readable
docker exec -it addon_a0d7b954_grocy sqlite3 /data/grocy/grocy.db "PRAGMA integrity_check;"
# Expected: ok

# Check table count
docker exec -it addon_a0d7b954_grocy sqlite3 /data/grocy/grocy.db "SELECT COUNT(*) FROM sqlite_master WHERE type='table';"
# Expected: ~40-50 tables
```

### Data Verification
- [ ] Login works with existing credentials
- [ ] Stock items count matches old instance
- [ ] Product images display correctly
- [ ] Shopping lists present
- [ ] Recipes intact
- [ ] Chores and tasks visible
- [ ] User accounts present (if multiple users)
- [ ] Barcode associations preserved

### Functional Testing
- [ ] Can create new stock item
- [ ] Can consume/purchase items
- [ ] Can add to shopping list
- [ ] Can edit recipes
- [ ] Can upload images
- [ ] Barcode scanner works (if used)
- [ ] Reports generate correctly

---

## Troubleshooting

### Database Locked Error

```bash
Error: database is locked
```

**Solution**:
```bash
# Ensure old add-on is stopped
ha addons stop a0d7b954_grocy

# Check for stale lock files
cd /mnt/data/supervisor/addons/data/a0d7b954_grocy
ls -la grocy.db*
rm grocy.db-shm grocy.db-wal  # Remove journal files if present

# Restart new add-on
ha addons restart a0d7b954_grocy
```

### Permission Denied Error

```bash
Error: unable to open database file
```

**Solution**:
```bash
cd /mnt/data/supervisor/addons/data/a0d7b954_grocy
chown root:root grocy.db
chmod 644 grocy.db
```

### Database Corrupted

```bash
Error: database disk image is malformed
```

**Solution**:
```bash
# Export and reimport database
sqlite3 grocy.db ".dump" | sqlite3 grocy_fixed.db
mv grocy.db grocy_broken.db
mv grocy_fixed.db grocy.db
```

### Missing Images

**Symptom**: Products have no images, placeholders shown

**Solution**:
```bash
# Verify storage directory copied
cd /mnt/data/supervisor/addons/data/a0d7b954_grocy
ls -la storage/productpictures/

# If missing, copy from backup
cp -r /backup/grocy_migration_YYYYMMDD/a0d7b954_grocy/storage ./

# Fix permissions
chmod -R 755 storage
```

### Database Version Mismatch

**Symptom**: Grocy shows "Database version mismatch" error

**Solution**: This shouldn't happen (same Grocy v4.5.0), but if it does:
```bash
# Grocy will auto-migrate on first start
# Check logs for migration progress
ha addons logs a0d7b954_grocy
```

---

## Complete Migration Script

Here's a complete script for the SSH method:

```bash
#!/bin/bash
# Grocy Database Migration Script
# Run this on Home Assistant host via SSH

set -e  # Exit on error

echo "=== Grocy Database Migration ==="
echo "This script will migrate your database from old to new add-on"
echo ""

# Configuration
OLD_ADDON_ID="a0d7b954_grocy"
BACKUP_DIR="/backup/grocy_migration_$(date +%Y%m%d_%H%M%S)"
DATA_DIR="/data/addons/$OLD_ADDON_ID"

# Step 1: Stop old add-on
echo "1. Stopping old Grocy add-on..."
ha addons stop $OLD_ADDON_ID
sleep 5

# Step 2: Create backup directory
echo "2. Creating backup directory..."
mkdir -p $BACKUP_DIR

# Step 3: Backup entire data directory
echo "3. Backing up database and files..."
cp -r $DATA_DIR/* $BACKUP_DIR/
echo "   Backup created at: $BACKUP_DIR"

# Step 4: Verify backup
echo "4. Verifying backup..."
if [ -f "$BACKUP_DIR/grocy.db" ]; then
    DB_SIZE=$(du -h "$BACKUP_DIR/grocy.db" | cut -f1)
    echo "   ✓ Database backed up: $DB_SIZE"
else
    echo "   ✗ ERROR: Database backup failed!"
    exit 1
fi

if [ -d "$BACKUP_DIR/storage" ]; then
    STORAGE_COUNT=$(find "$BACKUP_DIR/storage" -type f | wc -l)
    echo "   ✓ Storage backed up: $STORAGE_COUNT files"
else
    echo "   ⚠ Warning: No storage directory found"
fi

# Step 5: Instructions for new add-on
echo ""
echo "=== Backup Complete ==="
echo ""
echo "Next steps:"
echo "1. Uninstall old Grocy add-on (Settings → Add-ons → Grocy → Uninstall)"
echo "2. Install new Grocy add-on (DON'T START IT YET)"
echo "3. Run the restore script:"
echo ""
echo "   ha addons stop $OLD_ADDON_ID"
echo "   rm -rf $DATA_DIR/*"
echo "   cp -r $BACKUP_DIR/* $DATA_DIR/"
echo "   chown -R root:root $DATA_DIR"
echo "   chmod 644 $DATA_DIR/grocy.db"
echo "   chmod -R 755 $DATA_DIR/storage"
echo "   ha addons start $OLD_ADDON_ID"
echo ""
echo "Backup location: $BACKUP_DIR"
```

Save this as `/config/scripts/grocy_migrate.sh` and run:
```bash
chmod +x /config/scripts/grocy_migrate.sh
/config/scripts/grocy_migrate.sh
```

---

## Recovery Plan

If migration fails and you need to rollback:

```bash
# Stop new add-on
ha addons stop a0d7b954_grocy

# Restore from backup
cd /mnt/data/supervisor/addons/data/a0d7b954_grocy
rm -rf *
cp -r /backup/grocy_migration_YYYYMMDD/* ./

# Uninstall new add-on
ha addons uninstall a0d7b954_grocy

# Reinstall old add-on version
# (Use Home Assistant backup to restore old version)
```

---

## Summary: Recommended Approach

**For most users, the best approach is**:

1. **Stop old add-on** (Settings → Add-ons → Grocy → Stop)
2. **SSH into Home Assistant**
3. **Create full backup**: `cp -r /mnt/data/supervisor/addons/data/a0d7b954_grocy /backup/grocy_YYYYMMDD`
4. **Install new add-on** (don't start)
5. **Copy database**: `cp /backup/grocy_YYYYMMDD/grocy.db /mnt/data/supervisor/addons/data/a0d7b954_grocy/`
6. **Copy storage**: `cp -r /backup/grocy_YYYYMMDD/storage /mnt/data/supervisor/addons/data/a0d7b954_grocy/`
7. **Fix permissions**: `chown -R root:root /mnt/data/supervisor/addons/data/a0d7b954_grocy && chmod 644 /mnt/data/supervisor/addons/data/a0d7b954_grocy/grocy.db`
8. **Start new add-on**
9. **Verify data**

**Time estimate**: 10-20 minutes

**Risk level**: Low (with backup)

---

## Additional Resources

- **Home Assistant Backup**: Settings → System → Backups
- **Add-on Logs**: Settings → Add-ons → Grocy → Log
- **SQLite Documentation**: https://sqlite.org/cli.html
- **Grocy Issues**: https://github.com/grocy/grocy/issues
