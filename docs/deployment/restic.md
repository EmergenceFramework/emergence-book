# Restic Backups

## Install Restic CLI

```bash
hab pkg install --binlink core/restic
```

## Provision bucket

Create a bucket and obtain access credentials at a low-cost cloud object storage host:

=== "Linode"
    1. [Add an Object Storage bucket](https://cloud.linode.com/object-storage/buckets)
    2. [Create an Access key](https://cloud.linode.com/object-storage/access-keys)

=== "Backblaze"
    1. [Provision B2 bucket](https://secure.backblaze.com/b2_buckets.htm)
    2. [Provision B2 app key](https://secure.backblaze.com/app_keys.htm)

## Build restic environment

Create a secure file to store needed environment variables for the `restic` client to read and write to the encrypted repository bucket:

=== "Linode"
    ```bash
    #!/bin/bash

    RESTIC_REPOSITORY="s3:us-east-1.linodeobjects.com/myhost-restic"
    RESTIC_PASSWORD=""

    AWS_ACCESS_KEY_ID="" # Access Key
    AWS_SECRET_ACCESS_KEY="" # Secret Key
    ```

=== "Backblaze"
    ```bash
    #!/bin/bash

    RESTIC_REPOSITORY="b2:myhost-restic"
    RESTIC_PASSWORD=""

    B2_ACCOUNT_ID="" # keyID
    B2_ACCOUNT_KEY="" # applicationKey
    ```

1. Create `/etc/restic.env` from above template
2. Tailor `RESTIC_REPOSITORY` to created bucket
3. Generate `RESTIC_PASSWORD` and save to credential vault
4. Fill storage credentials
5. Secure configuration:

    ```bash
    sudo chmod 660 /etc/restic.env
    ```

## Initialize repository

Load the environment into your current shell to run Restic's one-time `init` command to set up the encrypted repository structure within the bucket:

```bash
set -a; source /etc/restic.env; set +a
restic init
```

## Create backup script

1. Create `/etc/cron.daily/emergence-restic-backup`

    ```bash
    #!/bin/bash

    set -a
    HOME=/root
    source /etc/restic.env
    set +a

    # snapshot host config to _
    >&2 echo -e "\n==> snapshot host"

    /bin/restic backup /emergence \
        --host=_ \
        --exclude='/emergence/services/**' \
        --exclude='/emergence/sites/**'

    # snapshot each site to its own host
    for site_path in /emergence/sites/*; do
    site_name=$(basename ${site_path})

    >&2 echo -e "\n==> snapshot site: ${site_name} @ ${site_path}"

    /bin/restic backup "${site_path}/" \
        --host="${site_name}" \
        --exclude='*.log' \
        --exclude='/emergence/sites/*/logs/**' \
        --exclude='/emergence/sites/**/media/*x*/**'
    done

    # setup mysql
    mysql_args="-u root -p$(/bin/jq -r .services.plugins.sql.managerPassword /emergence/config.json) -S /emergence/services/run/mysqld/mysqld.sock"

    mysql_query() {
        >&2 echo -e "\n==> mysql_query: ${1}"

        /usr/bin/mysql $mysql_args -srNe "${1}"
    }

    mysql_dump() {
        >&2 echo -e "\n==> mysql_dump: ${1} ${2}"

        /usr/bin/mysqldump ${mysql_args} \
            --force \
            --single-transaction \
            --quick \
            --compact \
            --extended-insert \
            --order-by-primary \
            --ignore-table="${1}.sessions" \
            "${1}" "${2}"
    }

    # dump each database+table
    databases=$(mysql_query 'SELECT schema_name FROM information_schema.schemata WHERE schema_name NOT IN ("information_schema", "mysql", "performance_schema")')

    for db_name in $databases; do

    if [ -d "/emergence/sites/${db_name}" ]; then
        cd "/emergence/sites/${db_name}"
    else
        cd "/tmp"
    fi

    mysql_dump "${db_name}" "${table_name}" \
        | /bin/restic backup \
            --host="${db_name}" \
            --stdin \
            --stdin-filename="database.sql"

    # restic de-dupe not as effective
    #    | /bin/gzip --rsyncable \
    done

    # thin out snapshots
    >&2 echo -e "\n==> restic forget"
    /bin/restic forget \
        --keep-last=1 \
        --keep-within=3d \
        --keep-daily=10 \
        --keep-weekly=10 \
        --keep-monthly=1200
    ```

2. Max executable:

    ```bash
    chmod +x /etc/cron.daily/emergence-restic-backup
    ```

## Run manually

To verify the configuration and create an initial snapshot:

```bash
/etc/cron.daily/emergence-restic-backup
```
