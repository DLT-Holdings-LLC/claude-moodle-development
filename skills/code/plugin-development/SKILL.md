---
description: Moodle plugin file structure, required files, registration patterns, capability definitions, external functions, and hard-won lessons from mod_skillsession development. Use this skill whenever building or modifying a Moodle plugin.
---

# Moodle Plugin Development

## Plugin File Structure

Activity modules use the `mod` plugin type. All files live under `/mod/pluginname/`.

```
mod_pluginname/
├── version.php              # Required — version, requires, maturity
├── lib.php                  # Required — add/update/delete instance callbacks
├── mod_form.php             # Required — course module edit form
├── view.php                 # Required — main activity page
├── index.php                # Required — list all instances in a course
├── lang/
│   └── en/
│       └── pluginname.php   # Required — all language strings
├── db/
│   ├── access.php           # Required — capability definitions
│   ├── install.xml          # Required ONLY if plugin has custom tables
│   ├── upgrade.php          # Required if install.xml exists
│   ├── services.php         # Optional — external function registration
│   └── messages.php         # Optional — message provider registration
├── classes/
│   └── external/            # External function classes go here
├── amd/
│   └── src/                 # JavaScript AMD modules
└── templates/               # Mustache templates
```

## version.php

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
 * Plugin version and other metadata.
 *
 * @package    mod_pluginname
 * @copyright  2026 Your Name <your@email.com>
 * @license    https://www.gnu.org/copyleft/gpl.html GNU GPL v3 or later
 */

defined('MOODLE_INTERNAL') || die();

$plugin->component = 'mod_pluginname';   // Must match directory and frankenstyle name.
$plugin->version   = 2026031002;         // YYYYMMDDXX format.
$plugin->requires  = 2024100700;         // Minimum Moodle version (4.5.0).
$plugin->maturity  = MATURITY_STABLE;
$plugin->release   = '1.0.2';
```

## Version Numbering Convention

Use `YYYYMMDDXX` format — e.g., `2026031002`. The last two digits are a sequence number for multiple releases on the same day. Moodle compares this value numerically to decide whether to run `upgrade.php`. **Never reuse a version number.** Always increment, even for minor changes.

**After any change to `db/services.php`, `db/access.php`, or `db/messages.php`, you MUST bump the version number and run the upgrade.** Moodle caches registered services; it will not pick up changes without a version bump.

```bash
# After bumping version.php:
php admin/cli/upgrade.php --non-interactive
# In DDEV:
ddev exec php admin/cli/upgrade.php --non-interactive
```

## Frankenstyle Naming

Plugin type prefix + underscore + plugin name. No hyphens, no uppercase.

- Activity module: `mod_pluginname`
- Local plugin: `local_pluginname`
- Block: `block_pluginname`
- Custom field: `customfield_pluginname`

This prefix governs **everything**: database table names, capability strings, language file names, class namespaces, and AMD module prefixes. Consistency is mandatory — a mismatch causes subtle failures that are hard to diagnose.

## lib.php — Required Callbacks

```php
<?php
/**
 * Library of functions for mod_pluginname.
 *
 * @package    mod_pluginname
 * @copyright  2026 Your Name <your@email.com>
 * @license    https://www.gnu.org/copyleft/gpl.html GNU GPL v3 or later
 */

defined('MOODLE_INTERNAL') || die();

/**
 * Add a new instance of mod_pluginname.
 *
 * @param stdClass $data Form data from mod_form.php.
 * @param mod_pluginname_mod_form|null $mform The form object.
 * @return int The new instance ID.
 */
function pluginname_add_instance(stdClass $data, ?mod_pluginname_mod_form $mform = null): int {
    global $DB;
    $data->timecreated = time();
    $data->timemodified = time();
    return $DB->insert_record('pluginname', $data);
}

/**
 * Update an existing instance of mod_pluginname.
 *
 * @param stdClass $data Form data.
 * @param mod_pluginname_mod_form|null $mform The form object.
 * @return bool True on success.
 */
function pluginname_update_instance(stdClass $data, ?mod_pluginname_mod_form $mform = null): bool {
    global $DB;
    $data->timemodified = time();
    $data->id = $data->instance;
    return $DB->update_record('pluginname', $data);
}

/**
 * Delete an instance of mod_pluginname.
 *
 * @param int $id The instance ID.
 * @return bool True on success.
 */
function pluginname_delete_instance(int $id): bool {
    global $DB;
    if (!$DB->get_record('pluginname', ['id' => $id])) {
        return false;
    }
    $DB->delete_records('pluginname', ['id' => $id]);
    return true;
}

/**
 * Declare supported features.
 *
 * @param string $feature The feature constant name.
 * @return mixed True/false/null or a string value.
 */
function pluginname_supports(string $feature): mixed {
    switch ($feature) {
        case FEATURE_MOD_INTRO:
            return true;
        case FEATURE_BACKUP_MOODLE2:
            return true;
        case FEATURE_MOD_PURPOSE:
            return MOD_PURPOSE_COLLABORATION;
        default:
            return null;
    }
}
```

Keep `lib.php` minimal. Heavy logic belongs in autoloaded classes under `classes/`.

## Capability Definitions (db/access.php)

```php
<?php
/**
 * Capability definitions for mod_pluginname.
 *
 * @package    mod_pluginname
 * @copyright  2026 Your Name <your@email.com>
 * @license    https://www.gnu.org/copyleft/gpl.html GNU GPL v3 or later
 */

defined('MOODLE_INTERNAL') || die();

$capabilities = [
    'mod/pluginname:view' => [
        'captype'      => 'read',
        'contextlevel' => CONTEXT_MODULE,
        'archetypes'   => [
            'guest'          => CAP_ALLOW,
            'student'        => CAP_ALLOW,
            'teacher'        => CAP_ALLOW,
            'editingteacher' => CAP_ALLOW,
            'manager'        => CAP_ALLOW,
        ],
    ],
    'mod/pluginname:addinstance' => [
        'riskbitmask'  => RISK_XSS,
        'captype'      => 'write',
        'contextlevel' => CONTEXT_COURSE,
        'archetypes'   => [
            'editingteacher' => CAP_ALLOW,
            'manager'        => CAP_ALLOW,
        ],
        'clonepermissionsfrom' => 'moodle/course:manageactivities',
    ],
];
```

Check capabilities with `require_capability()` (throws exception if denied) or `has_capability()` (returns bool). Never skip capability checks in `view.php` or any page accessible by URL.

## cmid vs Instance ID

`cmid` = the course module ID from `mdl_course_modules.id`. This is NOT the same as the activity's own record ID in `mdl_pluginname.id`.

`view.php` receives `?id=cmid`. Always resolve both:

```php
$id = required_param('id', PARAM_INT);  // Course module ID.

$cm = get_coursemodule_from_id('pluginname', $id, 0, false, MUST_EXIST);
$course = $DB->get_record('course', ['id' => $cm->course], '*', MUST_EXIST);
$pluginname = $DB->get_record('pluginname', ['id' => $cm->instance], '*', MUST_EXIST);

require_login($course, true, $cm);
$context = context_module::instance($cm->id);
require_capability('mod/pluginname:view', $context);
```

Confusing `$cm->id` (cmid) with `$cm->instance` (activity record ID) causes subtle bugs — the wrong record is fetched or updated silently.

## External Functions (classes/external/)

One class per function, one file per class. Located in `classes/external/`.

```php
<?php
/**
 * External function: get_slots.
 *
 * @package    mod_pluginname
 * @copyright  2026 Your Name <your@email.com>
 * @license    https://www.gnu.org/copyleft/gpl.html GNU GPL v3 or later
 */

namespace mod_pluginname\external;

use external_api;
use external_function_parameters;
use external_single_structure;
use external_multiple_structure;
use external_value;
use context_module;

/**
 * External function to retrieve slots for a pluginname instance.
 */
class get_slots extends external_api {

    /**
     * Describe input parameters.
     *
     * @return external_function_parameters
     */
    public static function execute_parameters(): external_function_parameters {
        return new external_function_parameters([
            'cmid' => new external_value(PARAM_INT, 'Course module ID'),
        ]);
    }

    /**
     * Execute the function.
     *
     * @param int $cmid Course module ID.
     * @return array List of slot objects.
     */
    public static function execute(int $cmid): array {
        $params = self::validate_parameters(self::execute_parameters(), ['cmid' => $cmid]);
        $context = context_module::instance($params['cmid']);
        self::validate_context($context);
        require_capability('mod/pluginname:view', $context);

        // Business logic here.
        return [];
    }

    /**
     * Describe return value.
     *
     * @return external_multiple_structure
     */
    public static function execute_returns(): external_multiple_structure {
        return new external_multiple_structure(
            new external_single_structure([
                'id'   => new external_value(PARAM_INT, 'Slot ID'),
                'name' => new external_value(PARAM_TEXT, 'Slot name'),
            ])
        );
    }
}
```

Note: External function classes in `classes/external/` do **not** need `defined('MOODLE_INTERNAL') || die()`. Phpcs will flag it as `MoodleInternalNotNeeded`.

## db/services.php — Registering External Functions

```php
<?php
/**
 * External function registrations for mod_pluginname.
 *
 * @package    mod_pluginname
 * @copyright  2026 Your Name <your@email.com>
 * @license    https://www.gnu.org/copyleft/gpl.html GNU GPL v3 or later
 */

defined('MOODLE_INTERNAL') || die();

$functions = [
    'mod_pluginname_get_slots' => [
        'classname'   => 'mod_pluginname\external\get_slots',
        'description' => 'Retrieve slots for a pluginname instance.',
        'type'        => 'read',
        'ajax'        => true,
    ],
];
```

**After adding or modifying `db/services.php`, you MUST bump `version.php` and run the upgrade.** Moodle will not re-read `services.php` without a version change. The upgrade script may report success while leaving the function unregistered. Verify by checking the `mdl_external_functions` table.

## db/messages.php — Message Provider Registration

```php
<?php
/**
 * Message provider definitions for mod_pluginname.
 *
 * @package    mod_pluginname
 * @copyright  2026 Your Name <your@email.com>
 * @license    https://www.gnu.org/copyleft/gpl.html GNU GPL v3 or later
 */

defined('MOODLE_INTERNAL') || die();

$messageproviders = [
    'booking_confirmation' => [
        'capability' => 'mod/pluginname:view',
        'defaults'   => [
            'popup' => MESSAGE_PERMITTED + MESSAGE_DEFAULT_ENABLED,
            'email'  => MESSAGE_PERMITTED + MESSAGE_DEFAULT_ENABLED,
        ],
    ],
];
```

**Critical:** As of Moodle 4.0+, the constant `MESSAGE_OUTPUT_EMAIL` no longer exists. Use the string `'email'` as the key in the `defaults` array (not the constant). Using `MESSAGE_OUTPUT_EMAIL` causes a fatal error during upgrade.

## Language Strings (lang/en/pluginname.php)

```php
<?php
/**
 * Language strings for mod_pluginname.
 *
 * @package    mod_pluginname
 * @copyright  2026 Your Name <your@email.com>
 * @license    https://www.gnu.org/copyleft/gpl.html GNU GPL v3 or later
 */

// Note: no defined('MOODLE_INTERNAL') in lang files — phpcs will flag it.

$string['modulename']       = 'Plugin Name';
$string['modulenameplural'] = 'Plugin Names';
$string['pluginname']       = 'Plugin Name';
$string['pluginadministration'] = 'Plugin Name administration';
$string['view']             = 'View';
```

Rules:
- Alphabetical order by key.
- No inline comments or section divider comments — phpcs flags these as `UnexpectedComment`.
- No `defined('MOODLE_INTERNAL') || die()`.
- Missing strings render as `[[key, pluginname]]` in the UI. Check every Mustache template `{{#str}}` call and every `get_string()` call.

## install.xml — When to Include It

Only include `db/install.xml` if your plugin creates custom database tables. An `install.xml` with an empty `<TABLES>` element causes a fatal `ddl_exception` ("XML database file errors found") during installation.

If you have no custom tables: **do not create `db/install.xml` at all.**

## Backup and Restore Files

Activity modules (`mod_`) **require** backup/restore classes:

```
backup/moodle2/
├── backup_pluginname_activity_task.class.php
├── backup_pluginname_stepslib.php
├── restore_pluginname_activity_task.class.php
└── restore_pluginname_stepslib.php
```

Other plugin types — `local_`, `block_`, `customfield_`, `tool_` — do **not** require backup/restore files. Including unnecessary backup files was flagged as a rejection reason by the Moodle Plugins Directory reviewer. Check the plugin type documentation before adding backup support.

## Delimiter-Safe Storage for Multi-Value Fields

If storing multiple values in a single database field with a delimiter (e.g., comma-separated IDs), values containing the delimiter character will silently corrupt on save and restore.

Options (choose one):
1. JSON-encode the array: `json_encode($values)` / `json_decode($stored, true)`.
2. Use a pipe `|` or newline delimiter if values are guaranteed delimiter-free.
3. Use a separate junction table (preferred for query flexibility).

This was flagged as a functional bug by the Moodle Plugins Directory reviewer.

## GitHub and Plugin Directory Submission

The Moodle Plugins Directory requires a publicly accessible bug tracker. If using GitHub:
- Ensure the repository is **public**.
- Ensure **Issues** are enabled in the repository settings.
- The reviewer will check this before approving the submission.
