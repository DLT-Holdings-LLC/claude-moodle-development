---
description: Moodle DML (Data Manipulation) and DDL (Data Definition) API patterns, install.xml structure, upgrade.php, and table naming conventions. Use this skill for all database work in Moodle plugins.
---

# Moodle Database API

## The Golden Rule: Never Use Raw SQL for DML

Always use the Moodle DML API (`$DB` methods). Do not use `mysqli_query()`, PDO, or raw `$DB->execute()` for data operations. Raw SQL bypasses the database abstraction layer and breaks cross-database compatibility (Moodle supports MySQL, PostgreSQL, MS-SQL, and Oracle).

The `$DB` global is available in any file that includes Moodle's `config.php` (directly or via `require_login()`).

---

## Core DML Methods

### Reading Single Records

```php
// Returns stdClass or false/exception depending on $strictness.
$record = $DB->get_record('pluginname_slot', ['id' => $slotId], '*', MUST_EXIST);
$record = $DB->get_record('pluginname_slot', ['cmid' => $cmid, 'status' => 1]);

// With custom WHERE clause:
$record = $DB->get_record_select(
    'pluginname_slot',
    'cmid = :cmid AND timecreated > :since',
    ['cmid' => $cmid, 'since' => time() - DAYSECS]
);

// With full custom SQL:
$record = $DB->get_record_sql(
    'SELECT s.*, u.firstname FROM {pluginname_slot} s
      JOIN {user} u ON u.id = s.userid
     WHERE s.id = :id',
    ['id' => $slotId]
);
```

**Strictness modes:**
- `MUST_EXIST` — throws `dml_exception` if zero or multiple records found. Use when you know the record must exist.
- `IGNORE_MISSING` (default) — returns `false` if not found. Shows debug warning if multiple records exist.
- `IGNORE_MULTIPLE` — returns the first record silently. Avoid this.

### Reading Multiple Records

```php
// Returns array of stdClass objects indexed by id field.
$slots = $DB->get_records('pluginname_slot', ['cmid' => $cmid], 'timecreated ASC');

// With field selection and limits:
$slots = $DB->get_records('pluginname_slot', ['cmid' => $cmid], 'timecreated DESC', 'id, name, status', 0, 10);

// With custom SQL:
$sql = 'SELECT s.id, s.name, COUNT(b.id) AS bookingcount
          FROM {pluginname_slot} s
     LEFT JOIN {pluginname_booking} b ON b.slotid = s.id AND b.status = :status
         WHERE s.cmid = :cmid
      GROUP BY s.id, s.name
      ORDER BY s.timecreated ASC';

$slots = $DB->get_records_sql($sql, ['cmid' => $cmid, 'status' => 0]);
```

**Note:** `get_records()` returns an array indexed by the `id` field of the first column. If you need a plain indexed array for JSON/templates, wrap with `array_values()`.

### Writing Records

```php
// Insert — returns the new record's ID.
$slot = new stdClass();
$slot->cmid        = $cmid;
$slot->name        = 'Morning session';
$slot->timecreated = time();
$slotId = $DB->insert_record('pluginname_slot', $slot);

// Update — requires the record to have an id property.
$slot->id          = $slotId;
$slot->name        = 'Afternoon session';
$slot->timemodified = time();
$DB->update_record('pluginname_slot', $slot);

// Delete by conditions (AND logic):
$DB->delete_records('pluginname_slot', ['cmid' => $cmid]);

// Delete with WHERE clause:
$DB->delete_records_select('pluginname_slot', 'cmid = :cmid AND status = :status', [
    'cmid'   => $cmid,
    'status' => $statusCancelled,
]);

// Update a single field:
$DB->set_field('pluginname_slot', 'status', $newStatus, ['id' => $slotId]);
```

### Checking Existence and Counting

```php
// Returns bool — use instead of count() when you only need to know if it exists.
if ($DB->record_exists('pluginname_booking', ['slotid' => $slotId, 'userid' => $userid])) {
    throw new moodle_exception('alreadybooked', 'mod_pluginname');
}

// Returns int.
$count = $DB->count_records('pluginname_booking', ['slotid' => $slotId, 'status' => 0]);

// With custom WHERE:
$count = $DB->count_records_select('pluginname_slot', 'cmid = :cmid AND timecreated > :since', [
    'cmid'  => $cmid,
    'since' => time() - WEEKSECS,
]);
```

---

## Parameter Binding

Always use placeholders. Never concatenate user-supplied values into SQL strings (SQL injection risk).

**Named placeholders (preferred — clearer and reusable):**

```php
$DB->get_records_sql(
    'SELECT * FROM {pluginname_slot} WHERE cmid = :cmid AND status = :status',
    ['cmid' => $cmid, 'status' => $status]
);
```

Named params must be **unique** even if the same value is used twice:

```php
// Wrong — duplicate named param:
'WHERE timecreated > :ts OR timemodified > :ts'

// Correct — use distinct names:
'WHERE timecreated > :ts1 OR timemodified > :ts2',
['ts1' => $since, 'ts2' => $since]
```

**Question mark placeholders:**

```php
$DB->get_records_sql(
    'SELECT * FROM {pluginname_slot} WHERE cmid = ? AND status = ?',
    [$cmid, $status]
);
```

**IN clauses — use get_in_or_equal():**

```php
[$insql, $inparams] = $DB->get_in_or_equal($slotIds, SQL_PARAMS_NAMED, 'slotid');
$slots = $DB->get_records_sql(
    "SELECT * FROM {pluginname_slot} WHERE id {$insql}",
    $inparams
);
```

---

## Large Result Sets — Use Recordsets

For memory-intensive operations, use a recordset instead of loading all records into an array.

```php
$rs = $DB->get_recordset('pluginname_slot', ['cmid' => $cmid]);
foreach ($rs as $slot) {
    // Process one slot at a time.
    process_slot($slot);
}
$rs->close(); // Always close the recordset.
```

Always call `$rs->close()` — use a try/finally if there's any risk of an exception:

```php
$rs = $DB->get_recordset_sql($sql, $params);
try {
    foreach ($rs as $record) {
        process($record);
    }
} finally {
    $rs->close();
}
```

---

## Transactions

Use delegated transactions when multiple writes must succeed or fail together.

```php
try {
    $transaction = $DB->start_delegated_transaction();

    $slotId = $DB->insert_record('pluginname_slot', $slot);
    $DB->insert_record('pluginname_booking', ['slotid' => $slotId, 'userid' => $userid]);

    $transaction->allow_commit();
} catch (Exception $e) {
    $transaction->rollback($e);  // Re-throws the exception.
}
```

Moodle uses "delegated" transactions — nested calls increment a counter rather than nesting real transactions. Only the outermost transaction commits or rolls back.

---

## Table Naming Conventions

**Format:** `plugintype_pluginname_tablename`

Examples:
- `mod_pluginname_slot` → `mdl_mod_pluginname_slot` (after Moodle adds its prefix)
- `mod_pluginname_booking` → `mdl_mod_pluginname_booking`

**In PHP DML calls:** omit the global prefix (`mdl_`). Use the bare table name without prefix:

```php
$DB->get_record('mod_pluginname_slot', ['id' => $id]);
```

**In custom SQL:** wrap table names in curly braces — Moodle adds the prefix automatically:

```php
$DB->get_records_sql('SELECT * FROM {mod_pluginname_slot} WHERE cmid = :cmid', ['cmid' => $cmid]);
```

---

## install.xml — Schema Definition

`db/install.xml` defines the initial database schema for fresh installations. **Edit only with the built-in XMLDB editor** (Site admin > Development > XMLDB editor) — do not hand-edit XML.

Minimal example:

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<XMLDB PATH="mod/pluginname/db" VERSION="20260310" COMMENT="XMLDB file for mod/pluginname"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="../../../lib/xmldb/xmldb.xsd">
  <TABLES>
    <TABLE NAME="pluginname_slot" COMMENT="Slots for mod_pluginname">
      <FIELDS>
        <FIELD NAME="id" TYPE="int" LENGTH="10" NOTNULL="true" SEQUENCE="true"/>
        <FIELD NAME="cmid" TYPE="int" LENGTH="10" NOTNULL="true" DEFAULT="0"/>
        <FIELD NAME="name" TYPE="char" LENGTH="255" NOTNULL="true" DEFAULT=""/>
        <FIELD NAME="timecreated" TYPE="int" LENGTH="10" NOTNULL="true" DEFAULT="0"/>
        <FIELD NAME="timemodified" TYPE="int" LENGTH="10" NOTNULL="true" DEFAULT="0"/>
      </FIELDS>
      <KEYS>
        <KEY NAME="primary" TYPE="primary" FIELDS="id"/>
      </KEYS>
      <INDEXES>
        <INDEX NAME="cmid" UNIQUE="false" FIELDS="cmid"/>
      </INDEXES>
    </TABLE>
  </TABLES>
</XMLDB>
```

**Do not include `db/install.xml` if the plugin has no custom tables.** An empty `<TABLES>` block causes a fatal `ddl_exception` during installation.

---

## upgrade.php — Incremental Schema Changes

`db/upgrade.php` handles schema changes for existing installations. Fresh installs use `install.xml` only and skip `upgrade.php`.

**After modifying `install.xml`, you MUST add the corresponding change to `upgrade.php`** inside a version-gated block. Existing sites will not get the schema change otherwise.

```php
<?php
/**
 * Upgrade script for mod_pluginname.
 *
 * @package    mod_pluginname
 * @copyright  2026 Your Name <your@email.com>
 * @license    https://www.gnu.org/copyleft/gpl.html GNU GPL v3 or later
 */

defined('MOODLE_INTERNAL') || die();

/**
 * Upgrade the mod_pluginname database.
 *
 * @param int $oldversion The version being upgraded from.
 * @return bool True on success.
 */
function xmldb_pluginname_upgrade(int $oldversion): bool {
    global $DB;
    $dbman = $DB->get_manager();

    if ($oldversion < 2026031002) {
        // Add status field to pluginname_slot table.
        $table = new xmldb_table('pluginname_slot');
        $field = new xmldb_field('status', XMLDB_TYPE_INTEGER, '4', null, XMLDB_NOTNULL, null, '0', 'timemodified');

        if (!$dbman->field_exists($table, $field)) {
            $dbman->add_field($table, $field);
        }

        upgrade_mod_savepoint(true, 2026031002, 'pluginname');
    }

    return true;
}
```

Each upgrade block must:
1. Be guarded by `if ($oldversion < YYYYMMDDXX)`.
2. Check whether changes already exist before applying them (`field_exists()`, `table_exists()`, etc.).
3. End with `upgrade_mod_savepoint(true, YYYYMMDDXX, 'pluginname')`.

Use the XMLDB editor's "Generate PHP code" feature to produce the DDL calls automatically.

---

## Cross-Database Compatibility Helpers

When writing SQL that must work across MySQL, PostgreSQL, and other databases:

```php
// LIKE pattern (database-safe):
$likesql = $DB->sql_like('name', ':name', false);  // false = case-insensitive.
$records = $DB->get_records_sql(
    "SELECT * FROM {pluginname_slot} WHERE {$likesql}",
    ['name' => '%' . $DB->sql_like_escape($search) . '%']
);

// String concatenation:
$sql = 'SELECT ' . $DB->sql_concat('firstname', "' '", 'lastname') . ' AS fullname FROM {user}';

// TEXT column comparison (required for PostgreSQL):
$sql = 'SELECT * FROM {pluginname_slot} WHERE ' . $DB->sql_compare_text('description') . ' = :desc';
```

Avoid MySQL-specific syntax. Use the helper functions above for any SQL that will run in production.
