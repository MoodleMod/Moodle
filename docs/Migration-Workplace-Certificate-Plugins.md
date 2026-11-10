# Migrating the Workplace Certificate Plugins

This document follows [Migrating an Existing Moodle 4.5 Site to
Moodle 5.3 on Azure](./Migration-4.5-to-5.3.md). The database import
and moodledata copy performed there move the certificate **data**, but
the certificate **plugin code** is not installed on the target
deployment, so templates, issued certificates and the course
activities will be invisible until the plugins are installed. The
plugins involved are maintained by Moodle HQ and are LMS-compatible:

| Plugin | Frankenstyle | Install path (5.x `public/` layout) |
|---|---|---|
| Workplace certificate manager | `tool_certificate` | `/moodle/html/moodle/public/admin/tool/certificate` |
| Workplace course certificate | `mod_coursecertificate` | `/moodle/html/moodle/public/mod/coursecertificate` |

`mod_coursecertificate` depends on `tool_certificate` — always
install/upgrade `tool_certificate` first and keep both plugins on the
same release train.

All commands run **on the controller VM** as root unless noted, and
use the database admin credentials from the deployment outputs
(`MOODLE_DATABASE_DNS`, `MOODLE_DATABASE_ADMIN_USERNAME`,
`MOODLE_DATABASE_ADMIN_PASSWORD` — see
[Get-Install-Data](./Get-Install-Data.md)).

## Check the plugin versions the migrated database expects

The code you install must be the same version or **newer** than the
version recorded in the migrated database, otherwise Moodle refuses to
start the plugins ("downgrade detected"). Substitute your
`$CFG->prefix` in the table name if it is not `mdl_` (see the prefix
reconciliation step of the migration guide):

```bash
mysql -h $MOODLE_DATABASE_DNS -u $MOODLE_DATABASE_ADMIN_USERNAME \
  -p"$MOODLE_DATABASE_ADMIN_PASSWORD" moodle -e \
  "SELECT plugin, value AS db_version FROM mdl_config_plugins
   WHERE plugin IN ('tool_certificate','mod_coursecertificate') AND name='version';"
```

Check the [tool_certificate](https://marketplace.moodle.com/plugins/tool_certificate/versions)
and [mod_coursecertificate](https://marketplace.moodle.com/plugins/mod_coursecertificate/versions)
version pages for a release that supports your target Moodle version.
If the newest release only declares support for an older Moodle
version, installation still proceeds as long as the plugin's
`$plugin->requires` is satisfied (undeclared support produces only a
warning) — but smoke-test thoroughly before declaring success.

**IMPORTANT NOTE**

In this example the plugin releases that replace the 4.5-compatible
versions (the `MOODLE_500_STABLE` 5.0.x releases used below) only
declare support up to **Moodle 5.2**. The Workplace developers (Moodle
HQ) make **no compatibility claim for Moodle 5.3**. They were found to
work on a Moodle 5.3 target during this migration, but they have
**not been exhaustively tested** on 5.3 — treat them as working at
your own risk, run the verification checklist below immediately after
installing, and check the Marketplace version pages above for a
release that officially declares 5.3 support before executing.

## Enable maintenance mode and take a pre-plugin backup

Save this backup under a **different filename** than the
pre-migration backup — reusing
`/moodle/db-backup-pre-migration.sql.gz` would overwrite the only
copy of the original target database that the migration guide's
rollback relies on:

```bash
sudo -u www-data php /moodle/html/moodle/admin/cli/maintenance.php --enable
mysqldump -h $MOODLE_DATABASE_DNS -u $MOODLE_DATABASE_ADMIN_USERNAME \
  -p"$MOODLE_DATABASE_ADMIN_PASSWORD" --single-transaction moodle \
  | gzip > /moodle/db-backup-pre-plugins.sql.gz
```

## Download and install the plugin code

Download pinned versions from the official GitHub repositories (the
Marketplace download URLs are not reliably fetchable with `curl`):

```bash
cd /tmp && rm -f *.zip
curl -fLo tool_certificate.zip \
  "https://github.com/moodleworkplace/moodle-tool_certificate/archive/refs/heads/MOODLE_500_STABLE.zip"
curl -fLo mod_coursecertificate.zip \
  "https://github.com/moodleworkplace/moodle-mod_coursecertificate/archive/refs/heads/MOODLE_500_STABLE.zip"
file *.zip   # both must report "Zip archive data"

unzip -q tool_certificate.zip && unzip -q mod_coursecertificate.zip
mv moodle-tool_certificate-MOODLE_500_STABLE certificate
mv moodle-mod_coursecertificate-MOODLE_500_STABLE coursecertificate
```

Install the code into the shared tree. Plugins go into their own
subdirectories under `public/` — never unzip anything over the
root. Use `cp` + `rm` rather than `mv` because `/tmp` (local disk) and
`/moodle` (Azure Files) are different filesystems:

```bash
sudo mkdir -p /moodle/html/moodle/public/admin/tool/certificate \
              /moodle/html/moodle/public/mod/coursecertificate
sudo cp -a /tmp/certificate/.       /moodle/html/moodle/public/admin/tool/certificate/
sudo cp -a /tmp/coursecertificate/. /moodle/html/moodle/public/mod/coursecertificate/
sudo chown -R www-data:www-data /moodle/html/moodle/public/admin/tool/certificate \
                                /moodle/html/moodle/public/mod/coursecertificate
sudo rm -rf /tmp/certificate /tmp/coursecertificate /tmp/*.zip

# Sanity check: version.php must be at the top level of each plugin directory
ls /moodle/html/moodle/public/admin/tool/certificate/version.php \
   /moodle/html/moodle/public/mod/coursecertificate/version.php
```

## Upgrade, propagate and verify

Repeat Steps 5–8 of the [migration
guide](./Migration-4.5-to-5.3.md) (upgrade, controller cache purge,
propagation to the web frontends, then disable maintenance mode).
Because the migrated database already holds the source plugin
versions, Moodle runs the plugins' own upgrade steps against the
existing tables — the migrated templates and issued certificates are
preserved, not recreated.

Finally verify:

1. *Site administration → Plugins → Plugins overview*: both plugins
   listed, no "missing from disk"
2. *Site administration → Certificates → Manage certificate
   templates*: migrated templates render with their images
3. Open a course with a Course certificate activity and download a PDF
   as a test student
4. The verification (QR) link on a pre-migration certificate validates
   on the public URL
5. Cron runs error-free:
   `sudo -u www-data php /moodle/html/moodle/admin/cli/cron.php`

## Rolling back

To restore the database to its pre-plugin state, use the
reset-and-import strategy from Step 3 of the migration guide with
`/moodle/db-backup-pre-plugins.sql.gz`.

To remove the certificate plugin code without touching the migrated
data (the data stays in the database and filedir):

```bash
sudo -u www-data php /moodle/html/moodle/admin/cli/maintenance.php --enable
rm -rf /moodle/html/moodle/public/admin/tool/certificate \
       /moodle/html/moodle/public/mod/coursecertificate

# Record the new plugin set — without this Moodle detects a
# plugin-version hash mismatch and suspends cron
rm -f /var/www/html/moodle/core_component.php
sudo -u www-data php /moodle/html/moodle/admin/cli/alternative_component_cache.php --rebuild
sudo -u www-data php /moodle/html/moodle/admin/cli/upgrade.php --non-interactive

sudo -u www-data php /moodle/html/moodle/admin/cli/purge_caches.php
```

Then repeat Steps 7 and 8 of the migration guide (propagate to the
web frontends — classmap rebuild and service restarts happen only
there, never on the controller — and disable maintenance mode).
