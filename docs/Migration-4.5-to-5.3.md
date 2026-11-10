# Migrating an Existing Moodle 4.5 Site to Moodle 5.3 on Azure

This document provides an overview of how to migrate an existing
Moodle 4.5 site (database + moodledata) into a Moodle cluster deployed
on Azure running Moodle 5.3, and how to bring the migrated content up
to the deployed code version. It is based on a verified migration of
this exact scenario.

Only the **database and moodledata files** are migrated by this
process — in a single server-side AzCopy pass that leaves every cache
behind. The Moodle code stays as deployed by the template. Add-on
plugin code is not migrated automatically — see [Migrating the
Workplace certificate plugins](./Migration-Workplace-Certificate-Plugins.md)
for installing add-on plugin code on the target after this migration.

## Prerequisites

In order to configure our deployment and tools we'll set up some
[environment variables](./Environment-Variables.md) to ensure consistency.

In order to migrate a site it is clearly necessary to first [deploy a
scalable Moodle cluster on Azure](./Deploy.md).

**IMPORTANT NOTE**

This guide assumes the target is a **fresh deployment of all
components in a new resource group** — a newly deployed controller VM,
web frontends, database server, Redis cache and file share that have
not served production traffic and hold no data worth keeping. Do not
run this migration against a resource group with an existing,
in-service Moodle installation: Step 2 overlays moodledata and Step 3
replaces the entire target database, destroying whatever was there
before.

For convenience and readability this document also assumes that
essential [deployment details for your cluster have been assigned to
environment variables](./Get-Install-Data.md).

In addition you will need read access to the **source** site's Azure
Files share, prepared as follows:

* `moodledata/` — the source site's moodledata directory
* `source-db.sql.gz` — a `mysqldump` of the source Moodle 4.5
  database, placed in the share root next to `moodledata/`:

  ```bash
  mysqldump -h <source-mysql-host> -u <source-dbadmin> -p'<source-dbadminpass>' \
    --single-transaction moodle | gzip > source-db.sql.gz
  ```

If the source site does not live on Azure Files, upload `moodledata/`
and the dump to a temporary share first (`azcopy copy` from the source
server) so that the migration pass in Step 2 remains a fast
server-side copy.

Except for the AzCopy pass (Step 2, runs anywhere with `azcopy` and
the Azure CLI), all commands below run **on the controller VM** (for
example via Bastion or `az vm run-command invoke -g $MOODLE_RG_NAME -n
<controller-vm>`), as root unless noted. The application database
credentials for the target cluster are in
`/moodle/html/moodle/config.php` (`dbhost`, `dbuser`, `dbpass`);
the database **admin** credentials are in the deployment outputs
(`MOODLE_DATABASE_DNS`, `MOODLE_DATABASE_ADMIN_USERNAME`,
`MOODLE_DATABASE_ADMIN_PASSWORD` — see
[Get-Install-Data](./Get-Install-Data.md)).

## Important rules before you begin

1. **Never unzip a source Moodle code tree over
   `/moodle/html/moodle`.** The Moodle 5.x layout is:
   * `/moodle/html/moodle/` — root: `config.php`, `admin/cli/` (CLI
     tools), build files
   * `/moodle/html/moodle/public/` — the web root (`$CFG->dirroot`),
     served by the web server

   Only the database and moodledata files migrate. The code stays as
   deployed.
2. **Never copy the source `config.php`** — the target one has the
   correct database, Redis, Azure Files and proxy settings. Merge
   individual custom settings by hand if needed.
3. **Never copy source caches**: skip `muc/`, `cache/`, `localcache/`,
   `sessions/`, `temp/` and `trashdir/` from the source
   moodledata. A copied `muc/config.php` will point Moodle at the
   *source* Redis and break every request.
4. The source Moodle major version must be upgrade-compatible with 5.3
   (4.2 or later is recommended; this guide assumes 4.5). Upgrade the
   source first if it is older.
5. **Keep maintenance mode enabled until the web frontends are
   synchronized** (Step 7); it is only disabled afterwards (Step 8).
   Disabling it earlier lets requests reach an upgraded database
   through frontends that still run stale code and caches.

## Step 1 — Enable maintenance mode and take a safety backup

Take the backup with the database **admin** account — default dumps
contain `LOCK TABLES` statements that the application user from
`config.php` is not granted:

```bash
sudo -u www-data php /moodle/html/moodle/admin/cli/maintenance.php --enable
mysqldump -h $MOODLE_DATABASE_DNS -u $MOODLE_DATABASE_ADMIN_USERNAME \
  -p"$MOODLE_DATABASE_ADMIN_PASSWORD" --single-transaction moodle \
  | gzip > /moodle/db-backup-pre-migration.sql.gz
```

Maintenance mode stays enabled through Step 7.

## Step 2 — Copy moodledata and the database dump in one AzCopy pass

A single server-side `azcopy` pass moves the source moodledata **and**
the database dump to the target share, and leaves every cache and
session directory behind. Run it from any machine with `azcopy` and
the Azure CLI:

```bash
# SAS tokens for the source and target shares (valid 8 hours)
exp=$(date -u -d "+8 hours" '+%Y-%m-%dT%H:%MZ')
srcKey=$(az storage account keys list -g <source-rg> -n <source-account> --query "[0].value" -o tsv)
tgtKey=$(az storage account keys list -g $MOODLE_RG_NAME -n <target-account> --query "[0].value" -o tsv)
srcSas=$(az storage share generate-sas --account-name <source-account> --account-key "$srcKey" \
  -n moodle --permissions rl --expiry $exp -o tsv)
tgtSas=$(az storage share generate-sas --account-name <target-account> --account-key "$tgtKey" \
  -n moodle --permissions rwdlc --expiry $exp -o tsv)

# One pass: moodledata + database dump; code tree and caches are never copied
azcopy copy \
  "https://<source-account>.file.core.windows.net/moodle?$srcSas" \
  "https://<target-account>.file.core.windows.net/moodle?$tgtSas" \
  --recursive --overwrite=ifSourceNewer --preserve-smb-info=false \
  --include-path "moodledata;source-db.sql.gz" \
  --exclude-path "moodledata/muc;moodledata/cache;moodledata/localcache;moodledata/sessions;moodledata/temp;moodledata/trashdir;moodledata/lock"
```

* `--include-path` restricts the copy to `moodledata/` and the dump —
  the source **code tree is never copied** (rule 1)
* `--exclude-path` leaves every cache behind (rule 3); in particular
  the deployed `/moodle/moodledata/muc/config.php`, which points at
  this environment's Redis cache, is never overwritten
* The copy is server-side; for large shares run it inside `tmux` so it
  survives disconnects, and resume interruptions with
  `azcopy jobs resume <job-id> --source-sas "$srcSas" --destination-sas "$tgtSas"`

Then fix ownership **on the controller VM**:

```bash
chown -R www-data:www-data /moodle/moodledata
```

## Step 3 — Reset and import the database

Importing the dump over the freshly deployed 5.3 schema would leave
target-only tables behind, which can make the upgrade fail with
"table already exists" errors. Drop and recreate the database first,
and import as the database **admin** user (`LOCK TABLES`, see Step 1):

```bash
mysql -h $MOODLE_DATABASE_DNS -u $MOODLE_DATABASE_ADMIN_USERNAME \
  -p"$MOODLE_DATABASE_ADMIN_PASSWORD" -e \
  "DROP DATABASE moodle; CREATE DATABASE moodle DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
gunzip < /moodle/source-db.sql.gz \
  | mysql -h $MOODLE_DATABASE_DNS -u $MOODLE_DATABASE_ADMIN_USERNAME \
      -p"$MOODLE_DATABASE_ADMIN_PASSWORD" moodle
```

## Step 4 — Reconcile the table prefix

If the source database uses a table prefix other than the target's
`$CFG->prefix` (default `mdl_`), the upgrade would read the wrong
tables. Compare them:

```bash
grep 'CFG->prefix' /moodle/html/moodle/config.php
zcat /moodle/source-db.sql.gz | grep -o -m1 'CREATE TABLE `[a-z0-9_]*config`'
```

If they differ, edit `/moodle/html/moodle/config.php` and set
`$CFG->prefix` to the **source** prefix before continuing.

## Step 5 — Run the upgrade (4.5 → 5.3)

```bash
cd /tmp
sudo -u www-data php /moodle/html/moodle/admin/cli/upgrade.php --non-interactive
```

If it aborts with `Outdated component cache file`, rebuild the
component cache first and then rerun the upgrade:

```bash
rm -f /var/www/html/moodle/core_component.php
sudo -u www-data php /moodle/html/moodle/admin/cli/alternative_component_cache.php --rebuild
```

## Step 6 — Purge caches on the controller

```bash
sudo -u www-data php /moodle/html/moodle/admin/cli/purge_caches.php
rm -rf /tmp/localcachedir
```

**IMPORTANT NOTE**

Do **not** restart web services on the controller. The deployment
deliberately stops nginx, PHP-FPM and Varnish there
(`scripts/install_moodle.sh`); restarting them would reopen a web
listener on the controller's public IP that bypasses the frontend
load balancer. Maintenance mode also stays **enabled** — the web
frontends have not been synchronized yet.

## Step 7 — Propagate code and caches to the web frontends

As described in [the management documentation](./Manage.md#updating-moodle-codesettings),
the web frontends keep a local copy of the Moodle code. After a
migration the local copies and their component caches must be
refreshed.

On the **controller** VM:

```bash
/usr/local/bin/update_last_modified_time.moodle_on_azure.sh
```

On **each** web frontend instance (for example `az vmss run-command
invoke ... --instance-id <id>`):

```bash
/usr/local/bin/sync_moodle_html_local_copy_if_modified.sh
rm -f /var/www/html/moodle/core_component.php
sudo -u www-data php /var/www/html/moodle/admin/cli/alternative_component_cache.php --rebuild
rm -rf /tmp/localcachedir
systemctl restart php8.3-fpm nginx varnish
```

**IMPORTANT NOTE**

Rebuild the component cache (classmap) from each node's **local**
tree, not the shared one — shared-tree paths in the classmap cause
`Cannot declare class ...` fatal errors on web requests.

## Step 8 — Disable maintenance mode

Only after **every** web frontend has been synchronized, disable
maintenance mode from the controller:

```bash
sudo -u www-data php /moodle/html/moodle/admin/cli/maintenance.php --disable
```

## Step 9 — Verify the migration

Derive the site hostname from `MOODLE_SITE_URL`
(set in [Get-Install-Data](./Get-Install-Data.md)):

```bash
MOODLE_SITE_HOSTNAME="${MOODLE_SITE_URL#https://}"
MOODLE_SITE_HOSTNAME="${MOODLE_SITE_HOSTNAME#http://}"
MOODLE_SITE_HOSTNAME="${MOODLE_SITE_HOSTNAME%%/*}"
```

```bash
# On a web frontend — expect 200 and the login page title
curl -s "http://localhost:81/login/index.php" \
  -H 'X-Forwarded-Proto: https' -H "Host: $MOODLE_SITE_HOSTNAME" \
  -o /tmp/v.html -w '%{http_code}\n'
grep -io '<title>[^<]*' /tmp/v.html

# Public URL — expect 200 and no maintenance text
curl -skL "https://$MOODLE_SITE_HOSTNAME/" -o /tmp/pub.html -w '%{http_code}\n'
```

Results:

```expected_similarity=0.4
200
<title>Log in to the site
200
```

Then log in as an administrator and check *Site administration →
Notifications* and the plugins overview page.

If the source site used add-on plugins (for example the Workplace
certificate plugins), install their code on the target next — see
[Migrating the Workplace certificate plugins](./Migration-Workplace-Certificate-Plugins.md).

## Rolling back

The Step 1 backup allows a full rollback of the database. Use the
same reset-and-import strategy as Step 3 (drop/recreate removes
migration-only tables; the admin user is needed for `LOCK TABLES`),
and keep maintenance mode enabled until the frontends are refreshed:

```bash
sudo -u www-data php /moodle/html/moodle/admin/cli/maintenance.php --enable
mysql -h $MOODLE_DATABASE_DNS -u $MOODLE_DATABASE_ADMIN_USERNAME \
  -p"$MOODLE_DATABASE_ADMIN_PASSWORD" -e \
  "DROP DATABASE moodle; CREATE DATABASE moodle DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
gunzip < /moodle/db-backup-pre-migration.sql.gz \
  | mysql -h $MOODLE_DATABASE_DNS -u $MOODLE_DATABASE_ADMIN_USERNAME \
      -p"$MOODLE_DATABASE_ADMIN_PASSWORD" moodle
sudo -u www-data php /moodle/html/moodle/admin/cli/purge_caches.php
```

If `$CFG->prefix` was changed in Step 4, restore the original value.
Then repeat Step 7 (propagate to the web frontends — restarts happen
only there, never on the controller) and Step 8 (disable maintenance
mode).
