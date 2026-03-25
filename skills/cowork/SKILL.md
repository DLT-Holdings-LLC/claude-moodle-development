---
name: moodle-development
description: >
  Moodle plugin development expertise covering file structure, coding standards,
  database API, frontend patterns, and operational commands for Moodle Workplace 5.1 / Moodle 4.5+.
  Use this skill whenever building, scaffolding, or modifying a Moodle plugin, writing PHP for Moodle,
  working with AMD/JavaScript/Mustache templates, running CLI operations, debugging a plugin,
  or any time the user mentions mod_, local_, block_, db/services.php, install.xml,
  DDEV, purge_caches, or external functions. Also trigger for general Moodle questions
  about APIs, hooks, capabilities, or upgrade patterns. When the user asks to create,
  build, scaffold, or generate a plugin or any plugin component, use this skill and
  create the actual files in the workspace rather than showing code blocks in chat.
---

# Moodle Development — Cowork Skill

This skill encodes Moodle 4.5+ / Workplace 5.1 development expertise for use in Cowork.
It covers six domains; jump to the relevant section:

1. [Plugin File Structure & Registration](#1-plugin-file-structure--registration)
2. [PHP Coding Standards (phpcs rules)](#2-php-coding-standards)
3. [Database API (DML/DDL)](#3-database-api)
4. [Frontend Patterns (AMD/Mustache/AJAX)](#4-frontend-patterns)
5. [Moodle Workplace Specifics](#5-moodle-workplace-specifics)
6. [Operations & CLI](#6-operations--cli)

---

## Cowork File Creation Mode

This is the most important difference between this skill and the Chat version. **When Darrel asks to build, scaffold, create, or generate a plugin or any plugin component, create actual files — do not just show code blocks in the chat.**

### When to create files vs. explain

- **"Build me a plugin..."** / **"Scaffold a..."** / **"Create a mod_..."** → Create all required files in the workspace.
- **"Add an external function to..."** → Create or update the specific file(s) needed.
- **"What's wrong with my..."** / **"How do I..."** → Explain in chat; no file creation needed.
- **"Fix this..."** (with a file attached) → Edit the file and save the corrected version.

### File output structure

**Always** save files under the plugin's own subfolder inside the Cowork outputs folder, preserving Moodle's exact directory structure. This means every file — whether part of a full scaffold or a single new file being added — lives under `outputs/mod_pluginname/`:

```
outputs/
└── mod_pluginname/          ← plugin root, always present
    ├── version.php
    ├── lib.php
    ├── mod_form.php
    ├── view.php
    ├── index.php
    ├── lang/en/pluginname.php
    ├── db/
    │   ├── access.php
    │   ├── install.xml       (only if the plugin has custom tables)
    │   ├── upgrade.php       (only if install.xml exists)
    │   └── services.php      (only if registering external functions)
    ├── classes/external/
    │   └── function_name.php
    ├── amd/src/
    │   └── module.js
    └── templates/
        └── view.mustache
```

**Even when adding a single file** (e.g., a new external function), place it at its correct path within the plugin folder. For example, a new external function for `mod_booking` goes to `outputs/mod_booking/classes/external/get_open_slots.php` — never to the outputs root.

When asked to scaffold a full plugin, create **all required files** — not just version.php. Refer to the plugin type's required file list in Section 1 to know what's mandatory vs. optional. After creating the files, show the directory tree and link to the outputs folder.

### CLI commands

Cowork cannot execute DDEV or PHP CLI commands. When Darrel needs to run something, show the exact commands as a code block with a note to run them in his terminal.

```bash
# Always show both local (DDEV) and production variants:
ddev exec php admin/cli/purge_caches.php      # local
php admin/cli/purge_caches.php                 # production
```

---

## Environment Reference

| Item | Value |
|------|-------|
| Platform | Moodle Workplace 5.1 (core: Moodle 4.5) |
| PHP | 8.3 |
| Database | MySQL via MySQLi |
| Local dev | DDEV at `/Users/darrel/Sites/moodle-dev/` on macOS |
| Production | AWS EC2, Ubuntu 22.04 |
| Experience level | Expert (20 yrs Moodle admin, complex plugin dev) |

---

## 1. Plugin File Structure & Registration

### Activity Module File Layout

```
mod_pluginname/
├── version.php              # Required — version, requires, maturity
├── lib.php                  # Required — add/update/delete instance callbacks
├── mod_form.php             # Required — course module edit form
├── view.php                 # Required — main activity page
├── index.php                # Required — list all instances in a course
├── lang/en/pluginname.php   # Required — all language strings
├── db/
│   ├── access.php           # Required — capability definitions
│   ├── install.xml          # ONLY if plugin has custom DB tables
│   ├── upgrade.php          # Required if install.xml exists
│   ├── services.php         # Optional — external function registration
│   └── messages.php         # Optional — message provider registration
├── classes/external/        # External function classes
├── amd/src/                 # JavaScript AMD modules
└── templates/               # Mustache templates
```

### version.php

```php
<?php
// This file is part of Moodle - https://moodle.org/
// [GPL header — required on every PHP file]

/**
 * Plugin version and other metadata.
 *
 * @package    mod_pluginname
 * @copyright  2026 Your Name <your@email.com>
 * @license    https://www.gnu.org/copyleft/gpl.html GNU GPL v3 or later
 */

defined('MOODLE_INTERNAL') || die();

$plugin->component = 'mod_pluginname';  // Must match directory + frankenstyle name.
$plugin->version   = 2026031002;        // YYYYMMDDXX format.
$plugin->requires  = 2024100700;        // Moodle 4.5.0.
$plugin->maturity  = MATURITY_STABLE;
$plugin->release   = '1.0.2';
```

**Version numbering:** `YYYYMMDDXX` — increment the last two digits for multiple releases same day.
Never reuse a version number. **After ANY change to `db/services.php`, `db/access.php`, or `db/messages.php`, you MUST bump version and run upgrade.** Moodle caches registered services and will not pick up changes without a version bump.

### Frankenstyle Naming

Plugin type prefix + underscore + plugin name. No hyphens, no uppercase.

| Type | Example |
|------|---------|
| Activity module | `mod_pluginname` |
| Local plugin | `local_pluginname` |
| Block | `block_pluginname` |
| Custom field | `customfield_pluginname` |

This prefix governs **everything**: DB table names, capabilities, lang files, class namespaces, AMD module prefixes. A mismatch causes subtle failures.

### cmid vs Instance ID

**Critical distinction.** `cmid` = course module ID from `mdl_course_modules.id`. Activity instance ID = record in the plugin's own table.

`view.php` receives `?id=cmid`:

```php
$id = required_param('id', PARAM_INT);  // Course module ID.

$cm      = get_coursemodule_from_id('pluginname', $id, 0, false, MUST_EXIST);
$course  = $DB->get_record('course', ['id' => $cm->course], '*', MUST_EXIST);
$activity = $DB->get_record('pluginname', ['id' => $cm->instance], '*', MUST_EXIST);

require_login($course, true, $cm);
$context = context_module::instance($cm->id);
require_capability('mod/pluginname:view', $context);
```

Confusing `$cm->id` (cmid) with `$cm->instance` (activity record ID) causes silent wrong-record bugs.

### Capability Definitions (db/access.php)

```php
$capabilities = [
    'mod/pluginname:view' => [
        'captype'      => 'read',
        'contextlevel' => CONTEXT_MODULE,
        'archetypes'   => [
            'student'        => CAP_ALLOW,
            'teacher'        => CAP_ALLOW,
            'editingteacher' => CAP_ALLOW,
            'manager'        => CAP_ALLOW,
        ],
    ],
];
```

Check with `require_capability()` (throws if denied) or `has_capability()` (returns bool). Never skip in `view.php` or any URL-accessible page.

### External Functions (classes/external/)

One class per function, one file per class. **Do NOT add `defined('MOODLE_INTERNAL') || die()` to external function classes** — phpcs flags `MoodleInternalNotNeeded`.

```php
namespace mod_pluginname\external;

use external_api;
use external_function_parameters;
use external_value;

class book_slot extends external_api {

    public static function execute_parameters(): external_function_parameters {
        return new external_function_parameters([
            'cmid'   => new external_value(PARAM_INT, 'Course module ID'),
            'slotid' => new external_value(PARAM_INT, 'Slot ID'),
        ]);
    }

    public static function execute(int $cmid, int $slotId): array {
        $params = self::validate_parameters(self::execute_parameters(), ['cmid' => $cmid, 'slotid' => $slotId]);
        $context = context_module::instance($params['cmid']);
        self::validate_context($context);
        require_capability('mod/pluginname:view', $context);
        // Business logic here.
        return ['success' => true];
    }

    public static function execute_returns(): external_single_structure {
        return new external_single_structure([
            'success' => new external_value(PARAM_BOOL, 'Whether booking succeeded'),
        ]);
    }
}
```

### db/services.php — Register External Functions

```php
$functions = [
    'mod_pluginname_book_slot' => [
        'classname'   => 'mod_pluginname\external\book_slot',
        'description' => 'Book a slot in a pluginname instance.',
        'type'        => 'write',
        'ajax'        => true,
    ],
];
```

After adding/modifying `db/services.php`: **bump version + run upgrade**. Verify registration by checking `mdl_external_functions` table.

### db/messages.php

```php
$messageproviders = [
    'booking_confirmation' => [
        'defaults' => [
            'popup' => MESSAGE_PERMITTED + MESSAGE_DEFAULT_ENABLED,
            'email' => MESSAGE_PERMITTED + MESSAGE_DEFAULT_ENABLED,  // String 'email', NOT the removed constant MESSAGE_OUTPUT_EMAIL
        ],
    ],
];
```

**Critical:** `MESSAGE_OUTPUT_EMAIL` constant was removed in Moodle 4.0+. Use the string `'email'` as the key. Using the old constant causes a fatal error during upgrade.

### install.xml — When to Include It

Only include `db/install.xml` if your plugin creates custom database tables. **An `install.xml` with an empty `<TABLES>` element causes a fatal `ddl_exception` during installation.** If you have no custom tables: do not create `db/install.xml`.

### Backup/Restore Files

Activity modules (`mod_`) **require** backup/restore classes. Other plugin types (`local_`, `block_`, `customfield_`, `tool_`) do **not**. Including unnecessary backup files was flagged as a rejection reason by the Moodle Plugins Directory reviewer.

### Multi-Value Field Storage

If storing multiple values in a single DB field with a delimiter, values containing the delimiter will corrupt silently. Use JSON (`json_encode`/`json_decode`) or a separate junction table instead.

### Moodle Plugins Directory Submission

- Repository must be **public**.
- **Issues** must be enabled in GitHub repository settings.
- Reviewer checks these before approving.

---

## 2. PHP Coding Standards

These rules are enforced by `moodle-plugin-ci phpcs`. Apply proactively — do not generate code that needs a cleanup pass.

```bash
# Validate:
ddev exec php vendor/bin/phpcs --standard=moodle mod/pluginname

# Auto-fix (fixes what it can):
phpcbf --standard=moodle mod/pluginname
```

### Rule 1: $camelCase Variables Only

**Never** `$snake_case`. This is the most common AI violation.

| Wrong | Correct |
|-------|---------|
| `$slot_id` | `$slotId` |
| `$booking_manager` | `$bookingManager` |
| `$open_count` | `$openCount` |
| `$course_module` | `$cm` or `$courseModule` |

Exceptions: Core globals `$CFG`, `$DB`, `$SESSION`, `$USER`, `$PAGE`, `$OUTPUT` (uppercase). Single-letter loop vars `$i`, `$j` are OK.

### Rule 2: Inline Comment Formatting

Every `//` comment must start with a capital letter and end with a period.

```php
// Wrong:
// get the slot record
// check if booking exists

// Correct:
// Get the slot record.
// Check if booking exists.
```

phpcbf cannot auto-fix this — must fix manually.

### Rule 3: PHPDoc on All Methods

```php
/**
 * Count open bookings for a slot.
 *
 * @param int $slotId The slot ID.
 * @return int The number of open bookings.
 * @throws dml_exception If the query fails.
 */
public function countOpenBookings(int $slotId): int {
```

Required: one-line description + `@param` for every param + `@return` + `@throws` if applicable.

### Rule 4: PHPDoc on All Class Constants

```php
/** @var int Status code indicating a confirmed booking. */
const STATUS_BOOKED = 0;

/** @var int Status code indicating a cancelled booking. */
const STATUS_CANCELLED = 1;
```

### Rule 5: PHPDoc on All Class Properties

```php
/** @var int The course module ID. */
protected int $cmid;
```

### Rule 6: File-Level PHPDoc

Every PHP file must start with the GPL license header + docblock with `@package`, `@copyright`, `@license`.

```php
<?php
// This file is part of Moodle - https://moodle.org/
//
// Moodle is free software: you can redistribute it and/or modify
// it under the terms of the GNU General Public License as published by
// the Free Software Foundation, either version 3 of the License, or
// (at your option) any later version.
//
// Moodle is distributed in the hope that it will be useful,
// but WITHOUT ANY WARRANTY; without even the implied warranty of
// MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
// GNU General Public License for more details.
//
// You should have received a copy of the GNU General Public License
// along with Moodle.  If not, see <https://www.gnu.org/licenses/>.

/**
 * Brief description of this file.
 *
 * @package    mod_pluginname
 * @copyright  2026 Your Name <your@email.com>
 * @license    https://www.gnu.org/copyleft/gpl.html GNU GPL v3 or later
 */
```

### Rule 7: MOODLE_INTERNAL — Know When NOT to Add It

`defined('MOODLE_INTERNAL') || die();` is required in most files, but phpcs flags `MoodleInternalNotNeeded` in:

**DO NOT add it to:**
- `classes/external/` — external function classes
- `classes/event/` — event classes
- Privacy provider classes
- `lang/en/pluginname.php` — lang files
- Entry-point files (`view.php`, `index.php`, `ajax.php`) that call `require_login()` / `require_config()`

**DO add it to:** `lib.php`, `db/access.php`, `db/services.php`, `db/messages.php`, `db/upgrade.php`, `version.php`, any included-but-not-autoloaded file.

### Rule 8: Language File Conventions

```php
// Correct lang file:
$string['addslot']    = 'Add slot';       // Alphabetical order.
$string['booknow']    = 'Book now';
$string['modulename'] = 'Plugin Name';

// Wrong:
// Booking strings.          <-- No inline comments — phpcs UnexpectedComment
defined('MOODLE_INTERNAL') || die();  // <-- Not needed in lang files
$string['zebra'] = 'Zebra'; // Out of alphabetical order
```

Missing strings render as `[[key, pluginname]]`. Check every `{{#str}}` in templates and every `get_string()` call.

### Rule 9: Deprecated Functions

Never use: `sizeof()` (use `count()`), `print_error()` (use `throw new moodle_exception()`), `array()` (use `[]`).

### Rule 10: Namespace = Directory Structure

`classes/external/book_slot.php` → `namespace mod_pluginname\external; class book_slot`
`classes/booking_manager.php` → `namespace mod_pluginname; class booking_manager`

One class per file, PSR-4 autoloading.

### Rule 11: Type Hints and Return Types

```php
// Required in new code:
function pluginname_get_slot(int $id): ?stdClass {
```

Use `?Type` for nullable, `void` for no return.

---

## 3. Database API

### Never Use Raw SQL for DML

Always use `$DB` methods. Raw `mysqli_query()`, PDO, or `$DB->execute()` for data ops breaks cross-database compatibility.

### Reading Records

```php
// Single record:
$record = $DB->get_record('pluginname_slot', ['id' => $slotId], '*', MUST_EXIST);

// With WHERE clause:
$record = $DB->get_record_select(
    'pluginname_slot',
    'cmid = :cmid AND timecreated > :since',
    ['cmid' => $cmid, 'since' => time() - DAYSECS]
);

// With custom SQL:
$record = $DB->get_record_sql(
    'SELECT s.*, u.firstname FROM {pluginname_slot} s
       JOIN {user} u ON u.id = s.userid
      WHERE s.id = :id',
    ['id' => $slotId]
);
```

**Strictness:** `MUST_EXIST` throws on zero/multiple. `IGNORE_MISSING` (default) returns false. Avoid `IGNORE_MULTIPLE`.

```php
// Multiple records (indexed by id):
$slots = $DB->get_records('pluginname_slot', ['cmid' => $cmid], 'timecreated ASC');

// For JSON/templates, flatten: array_values($slots)

// With SQL:
$slots = $DB->get_records_sql(
    'SELECT s.id, s.name, COUNT(b.id) AS bookingcount
       FROM {pluginname_slot} s
  LEFT JOIN {pluginname_booking} b ON b.slotid = s.id
      WHERE s.cmid = :cmid
   GROUP BY s.id, s.name',
    ['cmid' => $cmid]
);
```

### Writing Records

```php
// Insert — returns new ID:
$slot = new stdClass();
$slot->cmid        = $cmid;
$slot->timecreated = time();
$slotId = $DB->insert_record('pluginname_slot', $slot);

// Update (requires id property):
$slot->id           = $slotId;
$slot->timemodified = time();
$DB->update_record('pluginname_slot', $slot);

// Delete:
$DB->delete_records('pluginname_slot', ['cmid' => $cmid]);
$DB->delete_records_select('pluginname_slot', 'cmid = :cmid AND status = :s', ['cmid' => $cmid, 's' => 1]);

// Update single field:
$DB->set_field('pluginname_slot', 'status', $newStatus, ['id' => $slotId]);
```

### Existence and Counting

```php
if ($DB->record_exists('pluginname_booking', ['slotid' => $slotId, 'userid' => $userid])) { ... }

$count = $DB->count_records('pluginname_booking', ['slotid' => $slotId]);
$count = $DB->count_records_select('pluginname_booking', 'slotid = :sid AND status != :s', ['sid' => $slotId, 's' => 2]);
```

### install.xml vs upgrade.php

`install.xml` = schema for fresh installs. `upgrade.php` = incremental changes for existing installs.

After modifying `install.xml`, **also** add the change to `upgrade.php` inside a version-gated block:

```php
function xmldb_pluginname_upgrade(int $oldversion): bool {
    global $DB;
    $dbman = $DB->get_manager();

    if ($oldversion < 2026031002) {
        // Add new column.
        $table = new xmldb_table('pluginname_slot');
        $field = new xmldb_field('notes', XMLDB_TYPE_TEXT, null, null, null, null, null, 'status');
        if (!$dbman->field_exists($table, $field)) {
            $dbman->add_field($table, $field);
        }
        upgrade_mod_savepoint(true, 2026031002, 'pluginname');
    }

    return true;
}
```

### Table Naming

Tables: `{plugintype}_{pluginname}_{tablename}`. Moodle adds `mdl_` prefix automatically. Use `{tablename}` in install.xml, `'plugintype_pluginname_tablename'` in PHP.

Use the **XMLDB editor** (Site admin > Development > XMLDB editor) rather than editing XML by hand.

---

## 4. Frontend Patterns

### AMD Module Structure

Source: `amd/src/`. Built: `amd/build/` (do not edit). Module naming: `mod_pluginname/modulename`.

```javascript
// amd/src/booking.js
import Ajax from 'core/ajax';
import Notification from 'core/notification';

export const init = (cmid) => {
    // Use delegated handlers — attach to persistent parent, not individual buttons.
    document.addEventListener('click', (e) => {
        const btn = e.target.closest('[data-action="book-slot"]');
        if (!btn) { return; }
        e.preventDefault();
        bookSlot(cmid, parseInt(btn.dataset.slotid, 10));
    });
};

const bookSlot = async (cmid, slotId) => {
    try {
        const result = await Ajax.call([{
            methodname: 'mod_pluginname_book_slot',
            args: {cmid, slotid: slotId},
        }])[0];
        // Handle result.
    } catch (err) {
        Notification.exception(err);
    }
};
```

**Include from Mustache template:**

```mustache
{{#js}}
require(['mod_pluginname/booking'], function(Booking) {
    Booking.init({{cmid}});
});
{{/js}}
```

**After JS changes:**

```bash
ddev exec php admin/cli/purge_caches.php
```

### Mustache Partials — Flat Context Required

When using `{{> mod_pluginname/partial}}`, the partial receives only the current scope. It cannot access parent-level keys.

**Solution:** Flatten context in `export_for_template()`:

```php
public function export_for_template(\renderer_base $output): array {
    return [
        'cmid'     => $this->cm->id,
        'slotid'   => $this->slot->id,
        'slotname' => $this->slot->name,  // Promoted from nested object.
        'slots'    => array_values(array_map(fn($s) => $this->flatten_slot($s), $this->slots)),
    ];
}
```

This was the `promote_slot_fields()` pattern used in `mod_skillsession`.

### AJAX via External Functions

Call registered external functions from JS using `core/ajax`:

```javascript
const result = await Ajax.call([{
    methodname: 'mod_pluginname_book_slot',
    args: {cmid, slotid: slotId},
}])[0];
```

If you get `invalidrecord` errors, the function is not registered — bump version + run upgrade, then check `mdl_external_functions` table.

### Form Change Checker — Suppress Duplicate Dialog

Moodle's form change checker intercepts submissions with a confirmation dialog. If your plugin has its own confirmation dialog (e.g., cancel booking), add `data-no-submit-warning="1"` to the form element to suppress Moodle's duplicate dialog.

### Delegated Event Handlers

When AJAX calls replace page content, use delegated handlers attached to a persistent parent element rather than re-attaching after each AJAX response. This prevents duplicate handler registration.

---

## 5. Moodle Workplace Specifics

- Workplace 5.1 is built on Moodle 4.5. All core Moodle APIs are available.
- Workplace adds: multi-tenancy, program management, dynamic rules, content marketplace, and reporting.
- **Use only core Moodle APIs** in custom plugins to maintain compatibility. Do not depend on Workplace-specific tables or classes unless the plugin is Workplace-only.
- **Tenant isolation:** DB queries in multi-tenant environments may need to respect tenant boundaries. Test with multiple tenants active.
- Plugins distributed on the Moodle Plugins Directory must work with community Moodle, not just Workplace.

---

## 6. Operations & CLI

### Environment

| | Local (DDEV) | Production |
|--|--|--|
| Path | `/Users/darrel/Sites/moodle-dev/` | `/var/www/html/` (or Moodle root) |
| Command prefix | `ddev exec php` | `php` |

### Essential Commands

```bash
# Purge caches — run after ANY code change:
ddev exec php admin/cli/purge_caches.php
php admin/cli/purge_caches.php  # production

# Run upgrade — run after version bump or db/ changes:
ddev exec php admin/cli/upgrade.php --non-interactive
php admin/cli/upgrade.php --non-interactive  # production

# Manual cron:
ddev exec php admin/cli/cron.php

# Specific scheduled task:
php admin/cli/scheduled_task.php --execute='\mod_pluginname\task\send_reminders'

# Plugin status:
php admin/cli/plugin_manager.php --list

# Reset password:
php admin/cli/reset_password.php --username=admin
```

### Cache Purge Triggers

Purge after: any PHP file edit, Mustache template edit, lang string edit, AMD JS edit, any version bump.

### Upgrade Troubleshooting

If upgrade reports success but external functions/capabilities/messages are not registered:
1. Confirm version number was actually incremented.
2. Check `mdl_external_functions` table for the function.
3. Re-bump version and re-run upgrade.

### Debug Configuration (config.php)

```php
// Development only — NEVER on production:
$CFG->debug        = E_ALL;
$CFG->debugdisplay = 1;
$CFG->cachejs      = false;   // Read AMD from amd/src/ directly.
$CFG->debugsmtp    = true;    // Debug email issues.
```

### Caching Stack

Redis and Memcached available. Redis preferred for application cache and session storage. Use MUC (Moodle Universal Cache) for plugin-level caching — do not implement custom cache logic.

### DDEV-Specific

```bash
# SSH into container:
ddev ssh

# Run any PHP CLI command:
ddev exec php admin/cli/...

# Access MySQL:
ddev mysql

# View logs:
ddev logs
```

---

## Quick Diagnostic Checklist

When a plugin is not behaving as expected, run through this list:

- [ ] Purge caches after last change?
- [ ] Version bumped after `db/services.php`, `db/access.php`, or `db/messages.php` change?
- [ ] Upgrade run after version bump?
- [ ] External function registered? Check `mdl_external_functions`.
- [ ] `cmid` vs instance ID confusion? (`$cm->id` ≠ `$cm->instance`)
- [ ] Missing lang string? Renders as `[[key, pluginname]]`.
- [ ] `MESSAGE_OUTPUT_EMAIL` constant used instead of string `'email'`?
- [ ] `install.xml` present but empty `<TABLES>`? Remove the file.
- [ ] `defined('MOODLE_INTERNAL')` in an external function class? Remove it.
- [ ] Variable names using `$snake_case`? Should be `$camelCase`.
