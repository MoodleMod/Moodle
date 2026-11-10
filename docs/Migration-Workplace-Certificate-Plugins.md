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
versions (the 5.0.x releases pinned below) only
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
rollback relies on. `set -o pipefail` only makes the pipeline's exit
status reflect a `mysqldump` failure — the explicit check below is
what discards the failed archive and stops the procedure (a truncated
dump still gzips cleanly, so it must never be left behind):

```bash
sudo -u www-data php /moodle/html/moodle/admin/cli/maintenance.php --enable
set -o pipefail
if ! mysqldump -h $MOODLE_DATABASE_DNS -u $MOODLE_DATABASE_ADMIN_USERNAME \
    -p"$MOODLE_DATABASE_ADMIN_PASSWORD" --single-transaction moodle \
    | gzip > /moodle/db-backup-pre-plugins.sql.gz; then
  rm -f /moodle/db-backup-pre-plugins.sql.gz
  echo "BACKUP FAILED — do not continue"
  exit 1
fi
ls -l /moodle/db-backup-pre-plugins.sql.gz   # confirm a plausible size
```

## Download and install the plugin code

Download **pinned releases** from the official GitHub repositories
(the Marketplace download URLs are not reliably fetchable with
`curl`). Do not download a moving branch such as `MOODLE_500_STABLE` —
its content changes over time, so you could install different code
from what was verified. Pick the release **tag** for the 5.0.x version
you validated against the migrated database (see the version check
above and each repository's *Releases* page), and record it:

```bash
# Substitute the release tags you verified, e.g. the 5.0.x releases
TOOL_CERT_TAG="<tool_certificate-release-tag>"
MOD_CERT_TAG="<mod_coursecertificate-release-tag>"

# Reset the staging directory — leftovers from a previous run would make
# the mv commands below nest the new releases inside the old directories
rm -rf /tmp/moodle-plugins
mkdir -p /tmp/moodle-plugins && cd /tmp/moodle-plugins
curl -fLo tool_certificate.zip \
  "https://github.com/moodleworkplace/moodle-tool_certificate/archive/refs/tags/${TOOL_CERT_TAG}.zip"
curl -fLo mod_coursecertificate.zip \
  "https://github.com/moodleworkplace/moodle-mod_coursecertificate/archive/refs/tags/${MOD_CERT_TAG}.zip"
file *.zip   # both must report "Zip archive data"

unzip -q tool_certificate.zip && unzip -q mod_coursecertificate.zip
mv moodle-tool_certificate-*    certificate
mv moodle-mod_coursecertificate-* coursecertificate

# Record the exact plugin versions being installed
grep -h '\$plugin->\(version\|release\)' certificate/version.php coursecertificate/version.php
```

Install the code into the shared tree. Plugins go into their own
subdirectories under `public/` — never unzip anything over the
root. Use `cp` + `rm` rather than `mv` because `/tmp` (local disk) and
`/moodle` (Azure Files) are different filesystems:

```bash
sudo mkdir -p /moodle/html/moodle/public/admin/tool/certificate \
              /moodle/html/moodle/public/mod/coursecertificate
sudo cp -a /tmp/moodle-plugins/certificate/.       /moodle/html/moodle/public/admin/tool/certificate/
sudo cp -a /tmp/moodle-plugins/coursecertificate/. /moodle/html/moodle/public/mod/coursecertificate/
sudo chown -R www-data:www-data /moodle/html/moodle/public/admin/tool/certificate \
                                /moodle/html/moodle/public/mod/coursecertificate
# Remove only this guide's dedicated staging directory — never /tmp/*.zip,
# which would delete unrelated archives in the shared /tmp
sudo rm -rf /tmp/moodle-plugins

# Sanity check: version.php must be at the top level of each plugin directory
ls /moodle/html/moodle/public/admin/tool/certificate/version.php \
   /moodle/html/moodle/public/mod/coursecertificate/version.php
```

## Upgrade, propagate and verify

Repeat Steps 5 and 7–9 of the [migration
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

**IMPORTANT NOTE**

QR verification links embedded in certificates that were **already
downloaded or printed** before the migration are hard-coded to the
*source* hostname — the database URL replacement in Step 6 of the
migration guide cannot update them. If the public hostname changed,
keep the source hostname reachable over HTTPS with redirects to the
new site that preserve the verification path and query string
(`/admin/tool/certificate/index.php?code=...`), or those certificates
become unverifiable once the source is retired. The simplest option is
retaining the original public URL (Step 6, first option).

## Rolling back

Perform the rollback as **one ordered procedure under maintenance
mode**. Restoring the database while the newer plugin code is still
installed (or removing code against a database that recorded the newer
versions) leaves a code/schema mismatch: Moodle immediately requests
another upgrade, or detects a plugin-version hash mismatch and
suspends cron.

```bash
# 1 — close the site first
sudo -u www-data php /moodle/html/moodle/admin/cli/maintenance.php --enable

# 2 — restore the pre-plugin database (reset-and-import, as in Step 3
#     of the migration guide)
mysql -h $MOODLE_DATABASE_DNS -u $MOODLE_DATABASE_ADMIN_USERNAME \
  -p"$MOODLE_DATABASE_ADMIN_PASSWORD" -e \
  "DROP DATABASE moodle; CREATE DATABASE moodle DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
gunzip < /moodle/db-backup-pre-plugins.sql.gz \
  | mysql -h $MOODLE_DATABASE_DNS -u $MOODLE_DATABASE_ADMIN_USERNAME \
      -p"$MOODLE_DATABASE_ADMIN_PASSWORD" moodle

# 3 — remove the plugin code so it matches the restored database
#     (the migrated certificate data stays in the database and filedir)
rm -rf /moodle/html/moodle/public/admin/tool/certificate \
       /moodle/html/moodle/public/mod/coursecertificate

# 4 — record the new plugin set and rebuild/purge caches; without
#     upgrade.php Moodle detects a plugin-version hash mismatch and
#     suspends cron
rm -f /var/www/html/moodle/core_component.php
sudo -u www-data php /moodle/html/moodle/admin/cli/alternative_component_cache.php --rebuild
sudo -u www-data php /moodle/html/moodle/admin/cli/upgrade.php --non-interactive
sudo -u www-data php /moodle/html/moodle/admin/cli/purge_caches.php
```

Only then repeat Steps 8 and 9 of the migration guide: propagate to
the web frontends (classmap rebuild and service restarts happen only
there, never on the controller) and, after **every** frontend is
synchronized, disable maintenance mode.
