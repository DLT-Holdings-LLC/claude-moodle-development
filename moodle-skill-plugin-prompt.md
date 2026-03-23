# Prompt for Claude Code: Build the Moodle Development Skill Plugin

Paste this entire prompt into Claude Code from your terminal.

---

I need you to build a custom Claude skill plugin called `moodle-development` for use in Claude Code and Cowork. This plugin encodes Moodle development expertise so that Claude follows correct patterns without needing to search documentation or guess at conventions.

## My environment

- Moodle Workplace 5.1 (built on Moodle 4.5)
- PHP 8.3, MySQL via MySQLi
- Local dev: DDEV at /Users/darrel/Sites/moodle-dev/ on macOS
- Production: AWS EC2, Ubuntu 22.04
- I have 20 years of Moodle admin experience. I use Claude for complex plugin development.

## What to build

Create the following plugin structure:

```
moodle-development/
    .claude-plugin/plugin.json
    skills/
        plugin-development/SKILL.md
        coding-standards/SKILL.md
        frontend-patterns/SKILL.md
        database-api/SKILL.md
        workplace-specifics/SKILL.md
        operations/SKILL.md
```

## plugin.json manifest

```json
{
  "name": "moodle-development",
  "description": "Moodle plugin development expertise, API patterns, and operational best practices for Moodle Workplace 5.1 / Moodle 4.5+",
  "version": "1.0.0",
  "author": {
    "name": "Direct Support Learning"
  }
}
```

## Skill file requirements

Each SKILL.md file should have a YAML frontmatter block with a `description` field, then detailed markdown content. The content should be authoritative, specific, and opinionated — not vague guidelines. State the correct pattern, explain why, and note common mistakes.

For each skill, fetch the relevant pages from https://moodledev.io/ to get current API documentation, then distill the critical patterns into the skill file. Supplement with the specific lessons listed below from our actual development work.

### skills/plugin-development/SKILL.md

Fetch and distill from moodledev.io: Plugin files overview, version.php, lib.php, db/install.xml, db/upgrade.php, db/services.php, db/messages.php, lang strings, capabilities.

Must include these specific patterns learned from our mod_skillsession build:

- **MESSAGE_OUTPUT_EMAIL removal**: As of Moodle 4.0+, the constant `MESSAGE_OUTPUT_EMAIL` no longer exists. In `db/messages.php`, use the string `'email'` instead. This will cause a fatal error during upgrade if not corrected.
- **services.php requires version bump**: After adding or modifying `db/services.php`, you MUST increment the version number in `version.php` and run `php admin/cli/upgrade.php`. Moodle will NOT re-read services.php without a version change. The upgrade will report success but the external function will not be registered.
- **External function class pattern**: External functions go in `classes/external/` with one class per function. The class must implement `execute_parameters()`, `execute()`, and `execute_returns()`. Register in `db/services.php` with `'ajax' => true` for AJAX-callable functions.
- **Language strings**: Every string used in templates or PHP must be defined in `lang/en/{pluginname}.php`. Missing strings render as `[[key, pluginname]]` in the UI. Always check templates for `{{#str}}` calls and ensure every key exists in the lang file.
- **Version numbering convention**: Use YYYYMMDDXX format (e.g., 2026031002). Bump the last two digits for sequential changes on the same day. Moodle compares this numerically to decide whether to run upgrades.
- **Frankenstyle naming**: Plugin type prefix + underscore + plugin name. Activity modules use `mod_pluginname`. All database tables, capabilities, language files, and class namespaces must follow this convention.
- **Capability definitions**: Define in `db/access.php`. Every capability must have a context level and default archetypes. Use `mod/pluginname:capabilityname` format. Check capabilities with `require_capability()` or `has_capability()` — never skip capability checks.
- **The cmid vs instance ID distinction**: `cmid` is the course module ID (from `mdl_course_modules.id`). The activity instance ID is from the plugin's own table. `view.php` typically receives `?id=cmid` and must call `$cm = get_coursemodule_from_id()` to resolve both. Confusing these causes subtle bugs.
- **Do not include empty install.xml**: If the plugin has no custom database tables, do NOT include `db/install.xml`. An install.xml with an empty tables list causes a fatal `ddl_exception` ("XML database file errors found") during installation. This was flagged as a rejection reason by the Moodle Plugins Directory reviewer.
- **Only include backup/restore files when the plugin type requires them**: Activity modules (`mod_`) require backup/restore classes. But many other plugin types — `customfield_`, `local_`, `tool_`, `block_`, etc. — do NOT require backup/restore files. Including unnecessary backup files was flagged as a rejection reason by the Moodle Plugins Directory reviewer. Check the Moodle plugin type documentation before adding backup support.
- **Delimiter-safe storage for multi-value fields**: If a plugin stores multiple values in a single database field using a delimiter (e.g., comma-separated), values containing the delimiter character will break on save and restore. Either escape the delimiter in values, use a different delimiter (e.g., pipe `|` or newline), use JSON encoding, or use a separate junction table for true multi-value storage. This was flagged as a functional bug by the Moodle Plugins Directory reviewer.
- **GitHub Issues must be public for plugin directory submission**: The Moodle Plugins Directory requires a publicly accessible bug tracker. If using GitHub, ensure Issues are enabled in the repository settings before submitting.

### skills/coding-standards/SKILL.md

Fetch and distill from moodledev.io: Moodle coding style guide, PHP coding conventions. This skill ensures that generated code passes `moodle-plugin-ci phpcs` without requiring a cleanup session afterward. All patterns below are enforced by the Moodle phpcs ruleset and were identified from a 32-file cleanup session on mod_skillsession.

Must include:

- **Variable naming — $camelCase only**: All PHP variables must use `$camelCase`. Never use `$snake_case`. This is one of the most common violations when AI generates Moodle code. Examples: `$slotId` not `$slot_id`, `$bookingManager` not `$booking_manager`, `$openCount` not `$open_count`. Check every variable in every file.
- **Inline comment formatting**: Every inline comment must start with a capital letter and end with a period (or other sentence-ending punctuation). `// Get the slot record.` not `// get the slot record`. `// Check if booking exists.` not `// check if booking exists`. This applies to every `//` comment in the codebase.
- **PHPDoc blocks required on all class constants**: Every class constant must have a PHPDoc block above it: `/** @var int Status code for booked. */` before `const STATUS_BOOKED = 0;`. Missing docblocks on constants are flagged by phpcs.
- **PHPDoc blocks required on all methods**: Every method (public, protected, private) must have a complete PHPDoc block with `@param`, `@return`, and `@throws` tags as applicable. This is not optional.
- **defined('MOODLE_INTERNAL') || die() — know when to use it**: This guard is required in most PHP files that are included by Moodle. However, it is NOT needed (and phpcs will flag it as `MoodleInternalNotNeeded`) in: external API classes in `classes/external/`, event classes in `classes/event/`, privacy provider classes, lang files, and files that define their own entry point (like `view.php`, `index.php`). Do not add it reflexively to every file.
- **Language file conventions**: `lang/en/{pluginname}.php` must have all `$string` entries sorted alphabetically by key. Do NOT include inline comments or section divider comments inside lang files — phpcs flags these as `UnexpectedComment`. Do NOT add `defined('MOODLE_INTERNAL') || die()` to lang files.
- **Line length**: Maximum 180 characters per line. Break longer lines logically. This is rarely an issue but phpcs enforces it.
- **Forbidden/deprecated functions**: Do not use deprecated PHP or Moodle functions. Common violations include `sizeof()` (use `count()`), `print_error()` (use `throw new moodle_exception()`), and legacy Moodle output functions. If unsure whether a function is deprecated, check the Moodle deprecation documentation.
- **File-level PHPDoc**: Every PHP file must start with a standard Moodle file-level docblock including `@package`, `@copyright`, and `@license` tags.
- **One class per file**: Each PHP class should be in its own file, following PSR-4 autoloading conventions (namespace matches directory structure under `classes/`).
- **Run phpcs before committing**: Always validate with `moodle-plugin-ci phpcs mod/pluginname` (or the equivalent in your environment) before considering code complete. If phpcbf (auto-fixer) is available, run it first: `phpcbf --standard=moodle mod/pluginname`.

### skills/frontend-patterns/SKILL.md

Fetch and distill from moodledev.io: JavaScript/AMD modules, Mustache templates, Web services for AJAX.

Must include:

- **AMD module location**: Files go in `amd/src/`. Moodle compiles to `amd/build/` but in development mode reads from `src/`. Always purge caches after JS changes: `php admin/cli/purge_caches.php`.
- **Mustache partials require flat context**: When using `{{> mod_pluginname/partial}}`, the partial receives only the current context scope. If your main template passes a nested object, the partial cannot access parent-level keys. Solution: flatten the context in `export_for_template()` using a helper method that promotes nested fields to the top level.
- **AJAX via external functions**: Call registered external functions from JavaScript using `core/ajax` module's `call()` method. The function must be registered in `db/services.php` with `'ajax' => true`. If you get `invalidrecord` errors, the function is not registered — check the `mdl_external_functions` table.
- **Form change checker**: Moodle's form change checker intercepts form submissions with a confirmation dialog. If your plugin has its own confirmation (e.g., a cancel booking dialog), add `data-no-submit-warning="1"` to the form element to suppress Moodle's duplicate dialog.
- **RequireJS debugging**: If an AMD module doesn't load, `purge_caches.php` first. If still broken, check the module exists at the expected path. Moodle's RequireJS handler will silently return core JS bundles instead of your module if the path is wrong — it does not return a 404.
- **Delegated event handlers**: When AJAX calls replace page content, use delegated event handlers attached to a persistent parent element rather than re-attaching handlers after each AJAX response. This prevents duplicate handler registration.

### skills/database-api/SKILL.md

Fetch and distill from moodledev.io: Data manipulation API (DML), Data definition API (DDL), install.xml, upgrade.php.

Must include:

- **Never use raw SQL in plugins**. Always use the Moodle DML API: `$DB->get_record()`, `$DB->get_records()`, `$DB->insert_record()`, `$DB->update_record()`, `$DB->delete_records()`, `$DB->count_records()`, `$DB->record_exists()`.
- **Parameter binding**: Always use placeholders. Named: `['fieldname' => $value]`. The second argument to most $DB methods is a WHERE clause string with `:named` placeholders, and the third is the params array.
- **install.xml vs upgrade.php**: `install.xml` defines the schema for fresh installs. `upgrade.php` handles incremental changes for existing installs. After modifying `install.xml`, you must ALSO add the corresponding change to `upgrade.php` inside a version-gated block, or existing installations will not get the schema change.
- **Table naming**: Tables are named `{plugintype}_{pluginname}_{tablename}`. In install.xml and $DB calls, omit the global prefix (e.g., `mdl_`) — Moodle adds it automatically. Use `{tablename}` in install.xml, reference as `'plugintype_pluginname_tablename'` in PHP.
- **XMLDB editor**: Use the built-in XMLDB editor (Site admin > Development > XMLDB editor) to modify install.xml rather than editing XML by hand. It generates correct XML and can produce upgrade.php code.

### skills/workplace-specifics/SKILL.md

This skill covers what differs in Moodle Workplace from community Moodle. Note that Moodle Workplace documentation is not publicly available in the same way as community Moodle docs. Include what is known:

- Moodle Workplace is built on top of community Moodle (currently Moodle 4.5 for Workplace 5.1) and includes all core APIs.
- Workplace adds multi-tenancy, program management, dynamic rules, content marketplace, and reporting features.
- Custom plugins should use only core Moodle APIs to maintain compatibility. Do not depend on Workplace-specific database tables or classes unless the plugin is exclusively for Workplace deployments.
- Tenant isolation: be aware that database queries in a multi-tenant environment may need to respect tenant boundaries. Test custom plugins with multiple tenants active.
- Workplace licensing is separate from community Moodle. Plugins distributed on the Moodle Plugins Directory must work with community Moodle, not just Workplace.

### skills/operations/SKILL.md

Cover operational patterns for the DSL production environment:

- **CLI commands**: `php admin/cli/purge_caches.php` (after any code change), `php admin/cli/upgrade.php --non-interactive` (after version bumps), `php admin/cli/cron.php` (manual cron run). In DDEV local dev, prefix with `ddev exec`.
- **Caching**: Redis and Memcached are both available. Redis is preferred for application cache and session storage. Configure in config.php under `$CFG->cachedir` and cache store definitions.
- **Debugging**: Set `$CFG->debug = E_ALL` and `$CFG->debugdisplay = 1` in config.php for development. NEVER enable debug display on production. Use `$CFG->debugsmtp = true` to debug email issues.
- **Performance**: Enable opcode caching (OPcache is already active in the DSL stack). Use MUC (Moodle Universal Cache) for plugin-specific caching rather than implementing custom cache logic.
- **Backup**: Moodle's automated backup covers course content. Database backups should be handled separately via MySQL dumps. File data directory (`$CFG->dataroot`) must be backed up alongside the database.

## Instructions

1. Fetch the relevant pages from moodledev.io for each skill area to get current, accurate API documentation. For the coding-standards skill, also reference the Moodle coding style guide specifically.
2. Create all files in the structure above (6 skill files plus the plugin.json manifest).
3. Each skill file should be thorough but focused — aim for 150-300 lines per skill file. The coding-standards skill may be longer given the number of rules. Include code examples where they clarify the pattern.
4. For the coding-standards skill, include both correct and incorrect examples for the most common violations ($camelCase, inline comments, PHPDoc blocks, MOODLE_INTERNAL guard).
5. After creating all files, show me the plugin.json and a summary of what each skill covers so I can review before testing.
