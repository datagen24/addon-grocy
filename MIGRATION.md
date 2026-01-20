# Grocy Add-on Migration Guide

This guide provides instructions for backing up your existing Grocy data and migrating to the updated add-on version.

## What's Changing

This update modernizes the Grocy add-on with:
- **Base Image**: `ghcr.io/hassio-addons/base-nodejs:0.2.5` (Alpine 3.19, ARCHIVED) → `ghcr.io/hassio-addons/base:19.0.0` (Alpine 3.22, Active)
- **nginx**: 1.24.0-r16 → 1.28.0-r3
- **PHP**: 8.2.26-r0 → **8.3.29-r0** (major version upgrade - Grocy supports 8.2 or 8.3)
- **Composer**: 2.7.7-r0 → 2.8.10-r0
- **Grocy Version**: v4.5.0 (no change - already latest)

**Note**: PHP 8.3 upgrade is required because Alpine 3.22's composer package depends on PHP 8.3. Grocy explicitly supports both PHP 8.2 and 8.3 (composer.json: "php": ">=8.2").

## Pre-Migration Checklist

- [ ] Read this entire guide before starting
- [ ] Ensure you have access to Home Assistant via SSH or Terminal
- [ ] Verify you have sufficient disk space for backup (typically 50-500MB depending on usage)
- [ ] Note your current Grocy configuration settings
- [ ] Stop any automated processes that write to Grocy

## Important: Database-Only Migration

**If your old add-on version lacks SQL export**, you need to copy the database file directly. See [DATABASE_MIGRATION.md](DATABASE_MIGRATION.md) for detailed step-by-step instructions on:
- Extracting database from old instance
- Injecting into new instance before first start
- Complete migration script
- Troubleshooting common issues

**Quick Summary**:
1. Stop old add-on
2. Copy `/data/addons/a0d7b954_grocy/grocy.db` to backup
3. Install new add-on (don't start)
4. Copy database to new add-on data directory
5. Start new add-on

---

## Backup Process

### Method 1: Home Assistant Backup (Recommended)

This is the safest method as it creates a complete snapshot of your add-on.

1. **Create Full Backup**:
   ```
   Settings → System → Backups → Create Backup
   Select: Include Grocy add-on
   ```

2. **Download Backup** (Optional but recommended):
   - Click on the backup you just created
   - Select "Download Backup"
   - Store in a safe location off your Home Assistant server

### Method 2: Manual Data Backup

If you prefer manual backups or need granular control:

1. **Access Home Assistant via SSH or Terminal Add-on**

2. **Create Backup Directory**:
   ```bash
   mkdir -p /backup/grocy-migration-$(date +%Y%m%d)
   cd /backup/grocy-migration-$(date +%Y%m%d)
   ```

3. **Stop Grocy Add-on**:
   ```
   Settings → Add-ons → Grocy → Stop
   ```

4. **Backup Grocy Data**:
   ```bash
   # Backup entire data directory
   cp -r /data/addons/a0d7b954_grocy /backup/grocy-migration-$(date +%Y%m%d)/

   # Verify backup completed
   ls -lh /backup/grocy-migration-$(date +%Y%m%d)/a0d7b954_grocy
   ```

5. **Export Database (Additional Safety)**:
   ```bash
   # Navigate to backup directory
   cd /backup/grocy-migration-$(date +%Y%m%d)/a0d7b954_grocy

   # Create SQL dump
   sqlite3 grocy.db .dump > grocy_backup_$(date +%Y%m%d).sql

   # Verify SQL dump
   wc -l grocy_backup_$(date +%Y%m%d).sql
   ```

6. **Backup Configuration**:
   ```bash
   # Save add-on configuration
   cd /config
   cp -r addons_config/a0d7b954_grocy /backup/grocy-migration-$(date +%Y%m%d)/config_backup
   ```

7. **Create Archive** (Optional):
   ```bash
   cd /backup
   tar -czf grocy-backup-$(date +%Y%m%d).tar.gz grocy-migration-$(date +%Y%m%d)/
   ```

### What Gets Backed Up

Your backup includes:
- **Database**: `grocy.db` (all your inventory, recipes, chores, etc.)
- **Uploaded Files**: `storage/` (product images, receipts, user files)
- **View Cache**: `viewcache/` (compiled templates - regenerates automatically)
- **Configuration**: Add-on settings and options

## Migration Process

### Scenario 1: In-Place Upgrade (Recommended)

This approach updates the existing add-on installation.

1. **Ensure Backup Completed** (see above)

2. **Stop Grocy Add-on**:
   ```
   Settings → Add-ons → Grocy → Stop
   ```

3. **Update Add-on**:
   ```
   Settings → Add-ons → Grocy → Update
   ```

4. **Review Configuration**:
   - Check Configuration tab for any new options
   - Verify existing settings are preserved
   - No changes needed unless you want to modify settings

5. **Start Add-on**:
   ```
   Settings → Add-ons → Grocy → Start
   ```

6. **Monitor Logs**:
   ```
   Settings → Add-ons → Grocy → Log tab
   Watch for successful startup messages
   ```

7. **Verify Data Integrity**:
   - Access Grocy web interface
   - Check that all data appears correctly:
     - Stock items and quantities
     - Shopping lists
     - Recipes
     - Chores and tasks
   - Test creating/editing items
   - Verify uploaded images display

### Scenario 2: Fresh Install Migration

If you need to reinstall or migrate to a different system:

1. **Install Updated Add-on**:
   ```
   Settings → Add-ons → Add-on Store → Grocy → Install
   ```

2. **Do NOT start the add-on yet**

3. **Stop Add-on** (if it auto-started):
   ```
   Settings → Add-ons → Grocy → Stop
   ```

4. **Restore Data via SSH**:
   ```bash
   # Remove default data directory
   rm -rf /data/addons/a0d7b954_grocy

   # Restore from backup
   cp -r /backup/grocy-migration-YYYYMMDD/a0d7b954_grocy /data/addons/

   # Verify permissions
   chown -R root:root /data/addons/a0d7b954_grocy
   chmod -R 755 /data/addons/a0d7b954_grocy
   ```

5. **Restore Configuration**:
   ```bash
   # Restore add-on configuration
   cp -r /backup/grocy-migration-YYYYMMDD/config_backup/* /config/addons_config/a0d7b954_grocy/
   ```

6. **Start Add-on and Verify** (see Scenario 1, steps 5-7)

## Rollback Procedure

If you encounter issues and need to rollback:

### Quick Rollback via Home Assistant Backup

1. **Stop Current Add-on**:
   ```
   Settings → Add-ons → Grocy → Stop
   ```

2. **Restore Backup**:
   ```
   Settings → System → Backups → [Your backup] → Restore
   Select: Restore Grocy add-on only (partial restore)
   ```

### Manual Rollback

1. **Stop Add-on**:
   ```
   Settings → Add-ons → Grocy → Stop
   ```

2. **Restore Data**:
   ```bash
   # Remove current data
   rm -rf /data/addons/a0d7b954_grocy

   # Restore backup
   cp -r /backup/grocy-migration-YYYYMMDD/a0d7b954_grocy /data/addons/
   ```

3. **Downgrade Add-on** (if needed):
   ```
   Settings → Add-ons → Grocy → ⋮ → Uninstall
   Install previous version from backup or repository
   ```

## Troubleshooting

### Add-on Won't Start

**Check Logs**:
```
Settings → Add-ons → Grocy → Log
```

**Common Issues**:

1. **Database Locked**:
   ```
   Error: database is locked
   ```
   **Solution**: Ensure no other processes are accessing the database
   ```bash
   # Check for running processes
   ps aux | grep grocy

   # If needed, restart Home Assistant
   Settings → System → Restart
   ```

2. **Permission Errors**:
   ```
   Error: Permission denied
   ```
   **Solution**: Fix file permissions
   ```bash
   chown -R root:root /data/addons/a0d7b954_grocy
   chmod -R 755 /data/addons/a0d7b954_grocy
   ```

3. **Missing Dependencies**:
   ```
   Error: Package not found
   ```
   **Solution**: Rebuild the add-on
   ```
   Settings → Add-ons → Grocy → Rebuild
   ```

### Data Not Appearing

**Check Data Directory**:
```bash
ls -lh /data/addons/a0d7b954_grocy/
# Should show: grocy.db, storage/, viewcache/
```

**Verify Database Integrity**:
```bash
cd /data/addons/a0d7b954_grocy
sqlite3 grocy.db "PRAGMA integrity_check;"
# Should output: ok
```

**Restore from SQL Dump** (if database corrupted):
```bash
cd /data/addons/a0d7b954_grocy
mv grocy.db grocy.db.broken
sqlite3 grocy.db < /backup/grocy-migration-YYYYMMDD/a0d7b954_grocy/grocy_backup_YYYYMMDD.sql
```

### Ingress Mode Not Working

**Check Configuration**:
```
Settings → Add-ons → Grocy → Configuration
Verify: grocy_ingress_user setting
```

**Test Direct Mode**:
```
Enable port 80 in Configuration
Access via: http://homeassistant.local:PORT
Default login: admin / admin
```

**Check Nginx Logs**:
```bash
docker logs addon_a0d7b954_grocy 2>&1 | grep nginx
```

### Images/Files Not Loading

**Check Storage Directory**:
```bash
ls -lh /data/addons/a0d7b954_grocy/storage/
# Should show uploaded files
```

**Fix Permissions**:
```bash
chmod -R 755 /data/addons/a0d7b954_grocy/storage
```

## Verification Checklist

After migration, verify:

- [ ] Add-on starts without errors
- [ ] Web interface accessible (Ingress or Direct mode)
- [ ] Login works with existing credentials
- [ ] Stock items display correctly
- [ ] Product images load
- [ ] Shopping lists accessible
- [ ] Recipes appear with correct data
- [ ] Chores and tasks visible
- [ ] Can create new items
- [ ] Can edit existing items
- [ ] Can delete items (test with dummy data)
- [ ] Barcode scanner works (if used)
- [ ] Mobile app connects (if used)

## Data Location Reference

For advanced users, here are the key data locations:

```
/data/addons/a0d7b954_grocy/          # Main data directory
├── grocy.db                          # SQLite database (all core data)
├── storage/                          # User uploads
│   ├── productpictures/              # Product images
│   ├── quantityunitconversions/      # Unit conversion files
│   ├── userentities/                 # Custom entity files
│   └── userfiles/                    # Misc user files
└── viewcache/                        # Compiled templates (regenerates)

/config/addons_config/a0d7b954_grocy/ # Add-on configuration
└── options.json                      # User settings
```

## Support

If you encounter issues not covered in this guide:

1. **Check GitHub Issues**: [addon-grocy/issues](https://github.com/hassio-addons/addon-grocy/issues)
2. **Home Assistant Community**: [Community Forum](https://community.home-assistant.io/t/home-assistant-community-add-on-grocy/112422)
3. **Discord**: [Home Assistant Community Add-ons Discord](https://discord.me/hassioaddons)

## Post-Migration Recommendations

After successful migration:

1. **Update Default Password**:
   ```
   Grocy → Manage users → admin → Change password
   ```

2. **Review Settings**:
   ```
   Grocy → Settings → Review all sections
   Check for new features in v4.5.0
   ```

3. **Test Backup/Restore**:
   - Create a test backup
   - Verify you can restore successfully
   - Document your backup procedure

4. **Schedule Regular Backups**:
   ```
   Settings → System → Backups → Configure automatic backups
   Recommendation: Daily backups, keep 7 days
   ```

5. **Monitor Logs** (first week):
   ```
   Check for warnings or errors in add-on logs
   Report any issues to GitHub
   ```

## Migration Timeline Estimate

- **Backup Creation**: 5-10 minutes
- **In-Place Upgrade**: 5-15 minutes
- **Fresh Install Migration**: 15-30 minutes
- **Verification**: 10-15 minutes

**Total Estimated Time**: 30-60 minutes for complete migration with verification

## Legal & Safety

**Disclaimer**: Always maintain backups before performing system updates. While this migration process has been tested, the authors assume no responsibility for data loss. The migration is performed at your own risk.

**Data Privacy**: Your Grocy data remains local to your Home Assistant installation. This migration does not transmit data externally.
