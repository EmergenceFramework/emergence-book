# Docker



Emergence's multi-site runtime environment is available as a Docker image: [jarvus/emergence](https://hub.docker.com/r/jarvus/emergence/)

Going forward, this will be the recommended way to run an emergence host system. The `/emergence` tree within the container should be preserved via a volume or bind mount if you want to persist the sites set up within the container.

This guide will walk you through the complete process from a fresh machine, optionally migrating sites and data from an existing machine along the way.

### Setup guide

1. **Provision Ubuntu 18.04 and install current Docker version**

   Install Docker directly from Docker's official repository, bypassing the out-of-date versions distributed by Ubuntu's default sources:

   ```bash
    sudo apt update
    sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"
    sudo apt update
    sudo apt install -y docker-ce
   ```

2. **\(Optional\) Migrate data from an existing host**
   * To migrate data from an existing host, launch a temporary container from the `jarvus/emergence image` that just gives you a bash shell instead of running the kernel so you can set up the `/emergence` volume first, bind-mounting your current user's `~/.ssh` directory to gain ssh access to the existing host. This approach ensures that file ownership within the volume aligns with the correct users/groups defined in the container.

     ```bash
       docker run \
           --rm \
           -it \
           -v /emergence:/emergence \
           -v ~/.ssh:/root/.ssh \
           jarvus/emergence \
           bash
     ```

     Then, from inside this shell:

     1. Ensure you can connect to the existing machine's root user using ssh:

        ```bash
         ssh root@emergence.example.org
         # accept the new host key and then exit the shell after it works
        ```

     2. Use `rsync` to clone the existing `/emergence` tree from another machine to the new container volume, excluding run state, logs, and backups:

        ```bash
         rsync \
             --archive \
             --verbose \
             --progress \
             --exclude 'sql-backups' \
             --exclude '*.log' \
             --exclude '*.pid' \
             --exclude '*.sock' \
             root@emergence.example.org:/emergence/ /emergence/
        ```

        _For a clean migration, this should ideally be done while host services on the source machine are shut down. Cloning a running server for test purposes should be fine, but be aware of any additional load placed placed on the source server during the clone._

     3. Repair cloned table data offline with `myisamchk`:

        ```bash
         find /emergence/services/data/mysql/ -iname '*.MYI' -exec bash -c 'file="{}"; myisamchk -r "${file::-4}"' \;
        ```

     4. Back up the copied `emergence-kernel` config:

        ```bash
         cp -a /emergence/config.json /emergence/config.json.bak
        ```

     5. Update host-specific configuration properties to match what is appropriate for the container runtime:

        ```bash
         cat /emergence/config.json.bak | underscore process '
           data.user = "www-data";
           data.group = "www-data";
           data.server.host = "0.0.0.0";
           data.services.plugins.web.execPath = "/usr/sbin/nginx";
           data.services.plugins.web.bindHost = "0.0.0.0";
           data.services.plugins.sql.execPath = "/usr/sbin/mysqld";
           data.services.plugins.sql.bindHost = "0.0.0.0";
           data.services.plugins.php.execPath = "/usr/sbin/php-fpm5.6";
           data;
         ' > /emergence/config.json
        ```

     6. If you need to add or change administrative users from what was cloned from the existing host, install the [htpasswd](https://www.npmjs.com/package/htpasswd) command from NPM and use it now to make changes to `/emergence/admins.htpasswd`:

        ```bash
         npm install -g htpasswd

         # add user myuser:
         htpasswd -s /emergence/admins.htpasswd myuser
        ```

     7. Exit the shell to shut down and remove the temporary container:

        ```bash
         exit
        ```
3. **Create emergence multi-site host container**

   Run the container with all ports exposed and the `/emergence` tree bind-mounted to the same path on your Docker host machine:

   ```bash
    docker run -d \
        --name emergence \
        -v /emergence:/emergence \
        -p 80:80 \
        -p 3306:3306 \
        -p 9083:9083 \
        jarvus/emergence
   ```

   The administrative web UI should now be accessible at [http://127.0.0.1:9083](http://127.0.0.1:9083) via any user defined in `/emergence/admins.htpasswd`. Verify that you can login and that all services show as **online**.

4. **\(Optional\) Upgrade cloned MySQL data**

   If you cloned an existing machine earlier in this guide, you should now use the `mysql_upgrade` script to ensure the cloned MySQL data is in sync with the container's possibly newer MySQL version:

   ```bash
    docker exec -it emergence bash -c '
        mysql_upgrade \
            -u root \
            -p$(cat /emergence/config.json | underscore extract --outfmt text services.plugins.sql.managerPassword) \
            -S /emergence/services/run/mysqld/mysqld.sock
    '
   ```

### Useful commands

#### Open a MySQL shell

```bash
docker exec -it emergence emergence-mysql-shell
```

This is a wrapper for the normal `mysql` command-line client that handles the connection details for you, so you can use all its normal capabilities:

```bash
echo "SELECT * FROM people;" | docker exec -i emergence emergence-mysql-shell mydatabase
```

