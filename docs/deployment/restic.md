# Set up Backups with Restic

## Install restic-snapshot command

Install the `emergence/restic-snapshot` command:

```bash
hab pkg install emergence/restic-snapshot
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
hab pkg exec emergence/restic-snapshot restic init
```

## Create backup script

1. Create `/etc/cron.daily/emergence-restic-backup`

    ```bash
    #!/bin/bash

    /bin/hab pkg exec emergence/restic-snapshot snapshot
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

## Verifying backups

1. Load environment:

    ```bash
    set -a; source /etc/restic.env; set +a
    ```

2. List snapshots:

    ```bash
    hab pkg exec emergence/restic-snapshot restic snapshots
    ```

3. Examine contents of an SQL dump:

    ```bash
    hab pkg exec emergence/restic-snapshot restic dump ced0825f /database.sql | grep '^CREATE'
    ```
