---
description: What differs in Moodle Workplace vs community Moodle, multi-tenancy considerations, and compatibility guidelines for plugins running on Moodle Workplace 5.1 (built on Moodle 4.5).
---

# Moodle Workplace Specifics

## What Moodle Workplace Is

Moodle Workplace is a commercial distribution of Moodle built and maintained by Moodle HQ. Workplace 5.1 is built on community Moodle 4.5. It includes all core Moodle APIs — everything in the community documentation applies.

Workplace adds proprietary features on top of the community core:

| Feature | Description |
|---------|-------------|
| **Multi-tenancy** | Multiple organisations sharing one Moodle instance with isolated data |
| **Program management** | Structured learning paths that can span multiple courses |
| **Dynamic rules** | Event-triggered automation (enrol, notify, complete, etc.) |
| **Content marketplace** | Integration with third-party content providers |
| **Organisation & job assignments** | HR-style reporting lines and role assignment |
| **Workplace reporting** | Enhanced reporting on completion, programs, and tenants |

---

## Core Compatibility Rule

**Custom plugins should use only community Moodle APIs.**

Do not call Workplace-specific classes, functions, or tables directly unless the plugin is explicitly built for Workplace-only deployment and will never be submitted to the Moodle Plugins Directory.

Workplace-specific tables and classes begin with `tool_wp_` or exist under `admin/tool/wp/`. These are internal, undocumented, and may change without notice between Workplace versions.

```php
// Wrong — Workplace-only class, will break on community Moodle.
$tenant = \tool_wp\tenant::get_current();

// Correct — use community APIs that work everywhere.
$context = context_system::instance();
```

If you need to detect whether Workplace is active and conditionally use its features:

```php
if (class_exists('\\tool_wp\\tenant')) {
    // Workplace is present — use its API.
}
```

---

## Multi-Tenancy Considerations

In a Workplace multi-tenant environment, multiple organisations share one database. Users and courses are isolated per tenant using category-based cohort and context structures.

**Risks for custom plugins:**

1. **Queries that return all users** — a naive `$DB->get_records('user')` returns users from all tenants. In Workplace, users in other tenants should be invisible to managers in a different tenant.

2. **Cohort and category assumptions** — if your plugin filters by category or cohort, verify it respects tenant context. Workplace uses specific category trees per tenant.

3. **System-level capabilities** — capabilities granted at system context may bleed across tenants. Test with managers from different tenants to verify isolation.

**Test with multiple tenants active** before considering a plugin complete for a Workplace deployment. Create at least two tenants with separate users and courses, and verify that:
- Users in Tenant A cannot see Tenant B's data.
- Reports and lists are correctly scoped.
- Enrolments and completions are tenant-local.

---

## Programs and Dynamic Rules

Workplace's program engine tracks completion across multiple courses. Custom activity modules work inside programs if they correctly implement the Moodle completion API:

```php
// In lib.php — declare completion support.
function pluginname_supports(string $feature): mixed {
    switch ($feature) {
        case FEATURE_COMPLETION_TRACKS_VIEWS:
            return true;
        case FEATURE_COMPLETION_HAS_RULES:
            return true;
        // ...
    }
}

// Trigger completion when appropriate.
$completion = new completion_info($course);
if ($completion->is_enabled($cm)) {
    $completion->update_state($cm, COMPLETION_COMPLETE, $userid);
}
```

Dynamic rules trigger on Moodle core events. Emit standard core events from your plugin to make it compatible with dynamic rules:

```php
// Emit a standard event so dynamic rules can react to it.
$event = \core\event\user_loggedin::create(['userid' => $userid, 'context' => $context]);
$event->trigger();
```

Custom events can also be used, but dynamic rule integration requires configuring them in Workplace's admin UI.

---

## Plugin Directory Submissions from Workplace Environments

Plugins submitted to the Moodle Plugins Directory must work on community Moodle, not just Workplace. The reviewer will test on a standard Moodle installation.

**Checklist before submitting:**
- [ ] Plugin uses only community Moodle APIs.
- [ ] No references to `tool_wp_` tables or `\tool_wp\` classes.
- [ ] Plugin installs and runs cleanly on community Moodle 4.5 without Workplace.
- [ ] `version.php` has `$plugin->requires` set to the minimum community Moodle version (not a Workplace version).
- [ ] GitHub repository is public with Issues enabled.

---

## Version Alignment

| Moodle Workplace | Based On |
|-----------------|----------|
| Workplace 5.1 | Moodle 4.5 |
| Workplace 4.5 | Moodle 4.4 |

When setting `$plugin->requires` in `version.php`, use the Moodle version code, not the Workplace version:

```php
// Requires Moodle 4.5.0 (the base for Workplace 5.1).
$plugin->requires = 2024100700;
```

---

## Workplace-Specific Table Awareness

You should be aware these tables exist (for query planning and debugging) but should not reference them directly in custom plugins:

| Table prefix | Meaning |
|-------------|---------|
| `tool_wp_*` | Workplace core features (tenants, programs, rules) |
| `tool_tenant_*` | Tenant management and user assignment |

If you see these tables in query logs or debug output while testing, you are looking at Workplace infrastructure — leave them alone.

---

## Support and Documentation

Moodle Workplace documentation is not publicly available in the same way as community docs. Resources:

- Community Moodle developer docs: https://moodledev.io/ (the authoritative reference)
- Workplace admin docs: available to Moodle Partners only through the Moodle Academy Partner Hub
- Internal Workplace APIs: not documented publicly — rely on code inspection if needed

When in doubt, use community APIs and test on both community Moodle and your Workplace instance.
