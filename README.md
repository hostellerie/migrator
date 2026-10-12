# Migrator

Migrator is a Geeklog plugin for recovering content from legacy CMS database dumps into a **fresh Geeklog 2.2.2 installation**.

The project favors simple, auditable recovery over perfect reproduction of the source CMS.

## Target

- Geeklog **2.2.2 only**
- fresh destination installation
- MySQL-compatible database
- administration-only workflow

## Supported source CMS

1. Legacy Geeklog
2. glFusion
3. WordPress

## Migrator 1.0.0

Migrator 1.0.0 provides:

- Geeklog 2.2.2 autoinstall metadata
- `migrator.admin` permission and admin group
- isolated persistent storage under the site's `path_data`
- SQL dump upload
- explicit source CMS selection
- safe staging of `CREATE TABLE` and `INSERT INTO` statements
- staging table names under `migrator_src_*`
- blocking/skipping of unrelated SQL statements
- source-structure detection
- recoverable-content counts
- migration job tracking
- source/target ID mapping table
- migration log table
- legacy Geeklog dry run
- legacy Geeklog core migration writer
- glFusion core migration writer using the Geeklog-compatible core path
- glFusion Forum migration for categories, forums, posts, logs, moderators, user preferences, banned IPs and forum profile data
- glFusion MediaGallery migration for albums, media, album relations, queues, playback options, usage tracking, user preferences, watermarks, categories, sessions, sorting, ratings and EXIF settings
- MediaGallery `opacity` -> `wm_opacity` field conversion
- post-migration file manifest for MediaGallery media and converted Forum attachments
- ID preservation for users, topics, stories, comments and Static Pages when safe
- migration of legacy user profile fields into Geeklog 2.2.2 `user_attributes`
- automatic membership of imported users in the destination core groups
- migration of article/Static Page topic assignments
- preservation of legacy password hashes supported by Geeklog 2.2.2, with normal rehash-on-login behavior
- CSRF protection
- staged-source purge
- complete autouninstall metadata for plugin-owned tables, group and feature

Staging and analysis do **not** modify Geeklog destination content.

For Legacy Geeklog and glFusion core content, the administrator can then run an explicit **Dry run** followed by a confirmed **Migration** action. Migration writes are refused when the destination no longer matches the expected fresh-install baseline.

## Safety model

The uploaded SQL file is stored outside the public web root.

Source tables are recreated under isolated names. Migrator never executes source table names directly against Geeklog destination tables.

The staging importer accepts only the subset needed to recover source tables and rows. Other statements are skipped.

Destination migration is a separate, explicit action after analysis and dry-run.

## Development rules

Migrator follows the development and administration principles documented in:

- https://github.com/hostellerie/memorandum
- Geeklog 2.2.2 Plugin API conventions
- ACL checks on every administration entry point
- CSRF protection for state-changing actions
- Geeklog `DB_*` database abstraction
- `COM_createHTMLDocument()` page rendering
- templates for significant administration markup
- isolated persistent data storage
- repeatable installation and complete auto-uninstall cleanup

See [ROADMAP.md](ROADMAP.md) for the migration scope and milestones.


## glFusion plugin migration notes

### Forum

The current adapter migrates the compatible Forum data into the current Geeklog Forum tables.

The legacy glFusion `ff_attachments` records are converted to MediaGallery downloads. `ff_bookmarks` and `ff_rating_assoc` still have no direct target equivalents and are reported instead of being force-mapped.

### MediaGallery

The database schemas are highly compatible. Most MediaGallery tables migrate through their common columns while preserving IDs and relationships.

Media files are intentionally **not copied by Migrator**. After database migration, the administrator must copy the source MediaGallery files into MediaGallery's persistent storage. With MediaGallery 1.9.0 on a standard Geeklog 2.2.2 site, the default target is `public_html/images/mediagallery/` (derived from `$_CONF['path_images']`). Installations with a custom `path_images` / `images_url` pair must use that configured storage instead.


## WordPress migration

The WordPress adapter supports:

- users with deterministic UID remapping when WordPress IDs 1/2 conflict with Geeklog core users
- WordPress categories to Geeklog topics
- posts to Geeklog stories
- pages to Static Pages
- approved regular comments with parent/child relationships
- Geeklog comment tree rebuild after import
- author mapping through the migrated user map
- post/category topic assignments
- manual WordPress uploads copy to `public_html/images/wordpress/`
- URL rewriting for detected WordPress upload URLs inside migrated post/page content

Current limitations:

- WordPress password hashes are not copied; users must set a Geeklog password
- users without an email address require administrator intervention
- WordPress tags are reported but not imported yet
- pingbacks/trackbacks are not imported as comments
- attachment post records and featured-image metadata are not converted yet
- plugin-specific WordPress data is not migrated


## Package

The installable package for this release is:

`migrator_1.0.0_2.2.2.zip`

It is intended for Geeklog 2.2.2 only.


## Accepted database dump formats

Migrator accepts:

- `.sql`
- `.sql.gz`
- `.zip` containing exactly one `.sql` file

Compressed uploads are normalized to a private `.sql` file under `path_data/migrator/` before staging.

ZIP files are never extracted wholesale. Migrator reads the archive, rejects unsafe paths, requires exactly one SQL dump, and writes only that SQL entry to private storage.

The current uncompressed safety limit is 256 MiB.


### glFusion Forum attachments → MediaGallery

When the source contains `ff_attachments`, MediaGallery is a required destination dependency.

Migrator converts Forum attachments into a dedicated MediaGallery album named `Forum attachments (glFusion)`:

- normal Forum attachments are resolved from the glFusion Forum upload storage (default `public_html/forum/media/`);
- attachments with `repository_id > 0` are resolved through staged FileMgmt records;
- FileMgmt's default source store is `public_html/filemgmt_data/files/`, but installations with a customized or outside-webroot `FileStore` must use their actual configured source directory;
- each attachment receives a deterministic MediaGallery `media_id`;
- each migrated Forum post receives a MediaGallery `[download:...]` autotag pointing to the recovered file;
- attachment media are stored as generic downloadable MediaGallery items so no image/video derivatives are required during database migration;
- the dry run and migration report list the exact source and target file path for every attachment.

Migrator migrates the database records only. The administrator must copy the physical files to the exact MediaGallery persistent-storage paths shown in the report. On a standard Geeklog 2.2.2 / MediaGallery 1.9.0 installation, Forum attachments therefore land under `public_html/images/mediagallery/orig/<first-character>/...`.

## Legacy Geeklog Recovery direction (develop_1.1.0)

Migrator is the **SQL-based data recovery tool**, not a Core updater. Administrators stranded on old Geeklog or PHP versions can recover content from a SQL dump into a **fresh Geeklog 2.2.2 installation**, without running their legacy CMS on current PHP. Preserve the original SQL dump and media files.

**Planned improvements (not yet claimed supported):** detect historical Geeklog schema/version evidence and table prefixes; inventory legacy plugin tables; report missing or damaged data, available content, expected mapping, unsupported records, media dependencies, and URL/redirect needs. Historical versions must be advertised as supported only after fixture-based tests.

Recommended sequence: back up source SQL and files → provision clean Geeklog 2.2.2 → upload a copy into private, site-isolated staging → analyze → dry run → confirm import explicitly → copy referenced media → verify counts, relationships, login/password behavior, URLs, and missing files. No production source changes or direct Core upgrade are performed by Migrator.

**Responsibilities:** Migrator handles content import, adapters, ID mapping and reports. The native Geeklog installer/upgrade/rescue owns in-place Core upgrades, database upgrade checkpoints and recovery after failed upgrades. Monitor observes health and recommends actions. **No separate Updator plugin is planned.** See [maintenance architecture](https://github.com/hostellerie/memorandum/blob/docs/geeklog-222-multisite-upgrade-lessons/geeklog-installer-upgrade-rescue-architecture.md).

Security requirements: administrator-only access, ACL and CSRF checks for writes, bounded and validated SQL/archive inputs, private persistent staging, a fresh-destination baseline, non-mutating dry run, and explicit reporting rather than silent data loss. MySQL DDL does not provide universal rollback.
