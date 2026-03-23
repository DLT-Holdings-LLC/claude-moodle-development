---
description: Operational commands for Moodle development and production — CLI tools, cache management, debugging, upgrades, cron, and backup. Covers both DDEV local dev and AWS EC2 production environments.
---

# Moodle Operations

## Environment Overview

| Environment | Setup | Access |
|------------|-------|--------|
| Local dev | DDEV at `/Users/darrel/Sites/moodle-dev/` | `ddev exec php ...` |
| Production | AWS EC2, Ubuntu 22.04 | SSH, then `php ...` from Moodle root |

---

## Essential CLI Commands

All CLI commands run from the Moodle root directory (`/var/www/html/` on production, `/Users/darrel/Sites/moodle-dev/` locally).

### Cache Purging

**Run this after any code change.** Moodle aggressively caches templates, JS, language strings, and plugin definitions.

```bash
# Local (DDEV):
ddev exec php admin/cli/purge_caches.php

# Production:
php admin/cli/purge_caches.php
```

When to purge:
- After editing any PHP file (especially lib.php, classes/).
- After editing Mustache templates.
- After editing language strings.
- After editing AMD JavaScript (also rebuild JS with Grunt for production).
- After any version bump.

### Running Upgrades

Run after bumping `version.php` or modifying any `db/` file.

```bash
# Local:
ddev exec php admin/cli/upgrade.php --non-interactive

# Production:
php admin/cli/upgrade.php --non-interactive
```

The `--non-interactive` flag skips confirmation prompts. Without it, the CLI will pause waiting for input.

If the upgrade reports success but your external functions, capabilities, or messages are not registered, check that the version number was actually incremented before running the upgrade.

### Manual Cron Run

```bash
# Local:
ddev exec php admin/cli/cron.php

# Production:
php admin/cli/cron.php

# Run a specific scheduled task:
php admin/cli/scheduled_task.php --execute='\mod_pluginname\task\send_reminders'
```

### Other Useful CLI Tools

```bash
# Reset a user's password:
php admin/cli/reset_password.php --username=admin

# Check plugin status:
php admin/cli/plugin_manager.php --list

# Enable/disable a plugin:
php admin/cli/plugin_manager.php --enable=mod_pluginname

# Check for plugin updates:
php admin/tool/installaddon/cli/check.php

# Generate CSS from LESS (for themes):
php admin/cli/build_theme_css.php --themes=mytheme
```

---

## Debugging

### Development Debugging

Add to `config.php` for local development only:

```php
// Enable full error display.
$CFG->debug        = E_ALL;
$CFG->debugdisplay = 1;

// Disable JS and template caching for development.
$CFG->cachejs      = false;
$CFG->templatescache = false;
$CFG->langstringcache = false;

// Debug email sending.
$CFG->debugsmtp    = true;

// Debug performance (adds query count to footer).
$CFG->perfdebug    = 15;
$CFG->debugpageinfo = 1;
```

**NEVER enable `$CFG->debugdisplay = 1` on production.** It exposes stack traces, file paths, and database structure to anonymous users.

### Production Debugging

On production, write errors to a log file instead of displaying them:

```php
// In config.php — log errors without displaying them.
$CFG->debug        = E_ALL;
$CFG->debugdisplay = 0;
@ini_set('display_errors', '0');
@ini_set('log_errors', '1');
@ini_set('error_log', '/var/log/moodle/php_errors.log');
```

### Moodle Debug Mode via Admin UI

Site admin > Development > Debugging:
- Set `debug messages` to `DEVELOPER` for full output.
- Use only on non-production instances.

### Checking External Function Registration

```sql
SELECT * FROM mdl_external_functions WHERE name LIKE 'mod_pluginname%';
```

If your AJAX calls fail with `invalidrecord`, check this table first. If the function is missing, the services.php registration did not take effect — re-check the version bump and re-run the upgrade.

---

## Caching Architecture

### Cache Stores Available

| Store | Use Case |
|-------|---------|
| **Redis** | Preferred for application cache and session storage |
| **Memcached** | Available as alternative |
| **File system** | Default fallback, no additional config required |

Redis is preferred in the DSL stack. Configure in `config.php`:

```php
// Redis session handler.
$CFG->session_handler_class = '\core\session\redis';
$CFG->session_redis_host    = '127.0.0.1';
$CFG->session_redis_port    = 6379;
$CFG->session_redis_auth    = '';  // Password if set.
$CFG->session_redis_prefix  = 'mdl_sess_';
$CFG->session_redis_acquire_lock_timeout = 120;
$CFG->session_redis_lock_expire = 7200;
```

### Plugin-Level Caching with MUC

Use the Moodle Universal Cache (MUC) for plugin-specific data. Do not implement your own cache logic.

```php
// Define a cache in db/caches.php:
$definitions = [
    'slotdata' => [
        'mode'    => cache_store::MODE_APPLICATION,
        'simplekeys' => true,
        'ttl'     => 300,  // 5 minutes.
    ],
];

// Use the cache in PHP:
$cache = cache::make('mod_pluginname', 'slotdata');
$cacheKey = 'cmid_' . $cmid;

$slots = $cache->get($cacheKey);
if ($slots === false) {
    $slots = load_slots_from_db($cmid);
    $cache->set($cacheKey, $slots);
}
```

Purge plugin caches specifically:

```bash
php admin/cli/purge_caches.php
# Or from code:
cache_helper::purge_by_definition('mod_pluginname', 'slotdata');
```

---

## Performance

### OPcache

OPcache is active in the DSL stack. Configuration in `/etc/php/8.3/fpm/conf.d/10-opcache.ini` (or equivalent):

```ini
opcache.enable=1
opcache.memory_consumption=256
opcache.max_accelerated_files=20000
opcache.revalidate_freq=60
```

In development, set `opcache.revalidate_freq=0` or disable OPcache entirely so PHP picks up file changes immediately. In production, rely on the revalidation interval or clear manually after deployments.

### After Code Deployments to Production

```bash
# 1. Pull new code.
git pull origin main

# 2. Run upgrade if version.php changed.
php admin/cli/upgrade.php --non-interactive

# 3. Purge all Moodle caches.
php admin/cli/purge_caches.php

# 4. Reload PHP-FPM to clear OPcache (production).
sudo systemctl reload php8.3-fpm

# 5. Build AMD JavaScript if JS changed (requires Node/Grunt on server, or deploy built files).
# Option A: deploy amd/build/ from CI.
# Option B: grunt amd --root=mod/pluginname
```

---

## Backup and Restore

### Moodle's Automated Course Backup

Moodle's built-in backup covers course structure, activities, and grades. For activity modules with custom tables, ensure backup/restore classes are implemented (see plugin-development skill).

Automated backup schedule: Site admin > Courses > Automated backup setup.

### Database Backups

Moodle's course backup does NOT replace database backups. Run MySQL dumps separately:

```bash
# Full database dump:
mysqldump -u moodle -p moodle > /backups/moodle-$(date +%Y%m%d-%H%M%S).sql.gz | gzip

# DDEV local:
ddev export-db --file=/tmp/moodle-backup.sql.gz
```

### File Data Directory

`$CFG->dataroot` (typically `/var/moodledata/`) must be backed up alongside the database. It contains:
- Uploaded files (course files, user profile images, submitted assignments).
- File area cache.
- Session files (if not using Redis).
- Temp files (can be excluded from backup).

```bash
# Rsync dataroot to backup location (exclude temp and cache):
rsync -av --exclude='temp/' --exclude='cache/' /var/moodledata/ /backups/moodledata/
```

Both the database dump and the dataroot backup must be from the same point in time to restore successfully. Take them together or use a snapshot.

---

## DDEV Local Development Quick Reference

```bash
# Start the environment:
ddev start

# Open in browser:
ddev launch

# SSH into the container:
ddev ssh

# Run PHP CLI commands:
ddev exec php admin/cli/purge_caches.php
ddev exec php admin/cli/upgrade.php --non-interactive
ddev exec php admin/cli/cron.php

# View logs:
ddev logs

# Check database:
ddev mysql

# Stop the environment:
ddev stop
```

Moodle root inside the container is at `/var/www/html/`. Plugin development happens in `/Users/darrel/Sites/moodle-dev/` on macOS, which is mounted into the container.
