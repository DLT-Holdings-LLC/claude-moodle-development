---
description: Moodle PHP coding standards enforced by moodle-plugin-ci phpcs. All rules below were identified from a 32-file cleanup session on mod_skillsession. Follow these exactly to avoid a cleanup session after generation.
---

# Moodle Coding Standards

These rules are enforced by the Moodle phpcs ruleset. Generated code that violates them will fail CI. Apply all rules proactively — do not wait for phpcs to report violations.

## Validate Before Considering Code Complete

```bash
# Run phpcs:
moodle-plugin-ci phpcs mod/pluginname

# Auto-fix what can be auto-fixed first:
phpcbf --standard=moodle mod/pluginname

# In DDEV:
ddev exec php vendor/bin/phpcs --standard=moodle mod/pluginname
```

---

## Rule 1: Variable Naming — $camelCase Only

All PHP variables must use `$camelCase`. Never use `$snake_case`. This is the single most common violation when AI generates Moodle code.

| Wrong | Correct |
|-------|---------|
| `$slot_id` | `$slotId` |
| `$booking_manager` | `$bookingManager` |
| `$open_count` | `$openCount` |
| `$course_module` | `$cm` or `$courseModule` |
| `$user_data` | `$userData` |
| `$time_created` | `$timeCreated` |
| `$is_valid` | `$isValid` |

Exception: Moodle core globals use uppercase: `$CFG`, `$DB`, `$SESSION`, `$USER`, `$PAGE`, `$OUTPUT`. These are the only uppercase variables.

Single-letter loop variables (`$i`, `$j`, `$k`) are acceptable only in small loops.

---

## Rule 2: Inline Comment Formatting

Every inline comment must:
1. Start with a capital letter.
2. End with a period (or other sentence-ending punctuation: `?`, `!`).
3. Follow `// ` with a space after the slashes.

```php
// Wrong — lowercase start, no period
// get the slot record
// check if booking exists
// loop through results

// Correct
// Get the slot record.
// Check if booking exists.
// Loop through results.
```

This applies to every `//` comment in the codebase without exception. phpcbf cannot auto-fix this — you must fix it manually.

---

## Rule 3: PHPDoc on All Methods

Every method (public, protected, private) requires a complete PHPDoc block with:
- One-line description ending with a period.
- `@param` tag for every parameter (type + variable name + description).
- `@return` tag (use `void` if nothing is returned).
- `@throws` tag if the method can throw an exception.

```php
// Wrong — no PHPDoc
public function getSlot(int $slotId): stdClass {
    return $this->slot;
}

// Correct
/**
 * Retrieve a slot record by ID.
 *
 * @param int $slotId The slot ID.
 * @return stdClass The slot database record.
 * @throws dml_exception If the record does not exist.
 */
public function getSlot(int $slotId): stdClass {
    return $this->slot;
}
```

---

## Rule 4: PHPDoc on All Class Constants

Every class constant requires a PHPDoc block above it.

```php
// Wrong — no docblock on constant
const STATUS_BOOKED = 0;
const STATUS_CANCELLED = 1;

// Correct
/** @var int Status code indicating a confirmed booking. */
const STATUS_BOOKED = 0;

/** @var int Status code indicating a cancelled booking. */
const STATUS_CANCELLED = 1;
```

---

## Rule 5: PHPDoc on All Class Properties

Every class property requires an `@var` tag.

```php
// Wrong
protected $slotId;
private $bookingManager;

// Correct
/** @var int The slot ID. */
protected int $slotId;

/** @var booking_manager The booking manager instance. */
private booking_manager $bookingManager;
```

---

## Rule 6: File-Level PHPDoc Block

Every PHP file must begin with the GPL license header comment, followed by a docblock with `@package`, `@copyright`, and `@license`.

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

---

## Rule 7: MOODLE_INTERNAL — Know When to Use It

`defined('MOODLE_INTERNAL') || die();` is required in most PHP files. But phpcs will flag it as `MoodleInternalNotNeeded` in files that are auto-loaded or define their own entry point.

**DO NOT add it to:**
- External function classes in `classes/external/`
- Event classes in `classes/event/`
- Privacy provider classes
- Lang files (`lang/en/pluginname.php`)
- Entry-point files that call `require_login()` or `require_config()`: `view.php`, `index.php`, `ajax.php`

**DO add it to:**
- `lib.php`
- `db/access.php`, `db/services.php`, `db/messages.php`, `db/upgrade.php`
- `version.php`
- Any other file that is included by Moodle but is not an autoloaded class

```php
// Correct in lib.php, version.php, db/*.php:
defined('MOODLE_INTERNAL') || die();

// Wrong in classes/external/get_slots.php — phpcs flags MoodleInternalNotNeeded
defined('MOODLE_INTERNAL') || die(); // Remove this.
```

---

## Rule 8: Language File Conventions

```php
// Wrong — inline comment, section divider, wrong order
$string['zebra'] = 'Zebra';
// Booking strings.
$string['booknow'] = 'Book now';
$string['addslot'] = 'Add slot';

// Correct — alphabetical, no comments
$string['addslot']  = 'Add slot';
$string['booknow']  = 'Book now';
$string['zebra']    = 'Zebra';
```

Rules:
- All `$string` entries sorted **alphabetically** by key.
- **No inline comments** of any kind — phpcs flags as `UnexpectedComment`.
- **No** `defined('MOODLE_INTERNAL') || die()` in lang files.
- Values must be single-quoted strings; no concatenation, heredoc, or newdoc.

---

## Rule 9: Line Length

Maximum **180 characters** per line. Break longer lines logically.

```php
// Too long — break it up.
$records = $DB->get_records_sql('SELECT id, name, timecreated FROM {pluginname_slot} WHERE cmid = :cmid AND status = :status ORDER BY timecreated ASC', ['cmid' => $cmid, 'status' => $status]);

// Correct — split SQL and params.
$sql = 'SELECT id, name, timecreated
          FROM {pluginname_slot}
         WHERE cmid = :cmid
           AND status = :status
         ORDER BY timecreated ASC';
$records = $DB->get_records_sql($sql, ['cmid' => $cmid, 'status' => $status]);
```

---

## Rule 10: Forbidden and Deprecated Functions

| Forbidden | Replacement |
|-----------|-------------|
| `sizeof()` | `count()` |
| `print_error()` | `throw new \moodle_exception('errorkey', 'mod_pluginname')` |
| `array()` | `[]` |
| `eval()` | Never use |
| Backticks for shell execution | `exec()` or shell_exec if needed |
| `preg_replace()` with `/e` modifier | Use `preg_replace_callback()` |

Legacy Moodle output functions (`print_heading()`, `print_simple_box()`, etc.) are deprecated. Use `$OUTPUT->heading()`, renderers, and Mustache templates instead.

---

## Rule 11: One Class Per File

Each PHP class, interface, or trait belongs in its own file. The file name must match the class name, and the directory structure must match the namespace.

```
classes/
├── external/
│   ├── get_slots.php       → namespace mod_pluginname\external; class get_slots
│   └── create_booking.php  → namespace mod_pluginname\external; class create_booking
├── booking_manager.php     → namespace mod_pluginname; class booking_manager
└── slot.php                → namespace mod_pluginname; class slot
```

---

## Rule 12: Array Syntax

Always use short array syntax `[]`, never `array()`.

```php
// Wrong
$data = array('id' => 1, 'name' => 'Test');

// Correct
$data = ['id' => 1, 'name' => 'Test'];
```

---

## Rule 13: Type Hints and Return Types

New code must include type hints and return type declarations.

```php
// Wrong — no type hints
function pluginname_get_slot($id) {
    return $slot;
}

// Correct
function pluginname_get_slot(int $id): ?stdClass {
    return $slot;
}
```

Use `?Type` for nullable types. Use `void` for methods that return nothing.

---

## Complete Example: Correctly Styled Class

```php
<?php
// This file is part of Moodle - https://moodle.org/
// [GPL header omitted for brevity]

/**
 * Booking manager for mod_pluginname.
 *
 * @package    mod_pluginname
 * @copyright  2026 Your Name <your@email.com>
 * @license    https://www.gnu.org/copyleft/gpl.html GNU GPL v3 or later
 */

namespace mod_pluginname;

defined('MOODLE_INTERNAL') || die();

/**
 * Manages booking operations for a pluginname instance.
 */
class booking_manager {

    /** @var int Status indicating a confirmed booking. */
    const STATUS_BOOKED = 0;

    /** @var int Status indicating a cancelled booking. */
    const STATUS_CANCELLED = 1;

    /** @var int The course module ID. */
    protected int $cmid;

    /**
     * Construct the booking manager.
     *
     * @param int $cmid The course module ID.
     */
    public function __construct(int $cmid) {
        $this->cmid = $cmid;
    }

    /**
     * Count open bookings for a slot.
     *
     * @param int $slotId The slot ID.
     * @return int The number of open bookings.
     */
    public function countOpenBookings(int $slotId): int {
        global $DB;

        // Count confirmed bookings only.
        return $DB->count_records('pluginname_booking', [
            'slotid' => $slotId,
            'status' => self::STATUS_BOOKED,
        ]);
    }
}
```
