# Installation Guide for Ubuntu 16.04

## Overview

### Quick install

If you are starting from a **fresh** Ubuntu 16.04 installation you can use this script to run all the following setup commandsall at once, and then skip down to **[Create a site](#create-a-site) or [Secure your installation](#secure)**.

```language-bash
user@hostname ~ $ wget http://emr.ge/dist/ubuntu/quickinstall-16.04.sh -O - | sudo sh
```

## Install service binaries

These commands will update your system and then install all the packages required for Emergence:

```language-bash
user@hostname ~ $ sudo apt-get update && sudo apt-get upgrade -y
user@hostname ~ $ sudo DEBIAN_FRONTEND=noninteractive apt-get install -y git python-software-properties python g++ make ruby-dev nodejs nginx php7.0-fpm php7.0-cli php-apcu php7.0-mysql php7.0-gd php7.0-json php7.0-curl php7.0-intl php7.0-mbstring php-imagick mysql-server mysql-client gettext imagemagick postfix ruby-compass
```

## Stop and disable default service instances

Emergence will be configuring and launching nginx, mysql, and php7.0-fpm for us, so we need to get the default instances set up by Ubuntu out of the way:

```language-bash
user@hostname ~ $ sudo service nginx stop && sudo update-rc.d -f nginx disable
user@hostname ~ $ sudo service php7.0-fpm stop && sudo update-rc.d -f php7.0-fpm disable
user@hostname ~ $ sudo service mysql stop && sudo update-rc.d -f mysql disable
```

## Configure AppArmor to allow Emergence to manage `mysqld`

AppArmor must be configured to allow MySQL to use Emergence files instead of the defaults:

```language-bash
user@hostname ~ $ echo -e "/emergence/services/etc/my.cnf r,\n/emergence/services/data/mysql/ r,\n/emergence/services/data/mysql/** rwk,\n/emergence/services/run/mysqld/mysqld.sock w,\n/emergence/services/run/mysqld/mysqld.pid rw,\n/emergence/services/run/mysqld/mysqld.sock.lock rw," | sudo tee -a /etc/apparmor.d/local/usr.sbin.mysqld
user@hostname ~ $ sudo service apparmor restart
```

## Increase shared memory limit

Ubuntu comes with a low limit of 32MB for shared memory. Emergence relies heavily on APC caching and needs kernel.shmmax increased to a more flexible amount. We'll use 128MB:

```language-bash
user@hostname ~ $ echo -e "apc.shm_size=128M" | sudo tee -a /etc/php5/mods-available/apcu.ini
```

## Install Emergence from GitHub

Clone Emergence into your home directory, then use `npm` to install the package and its dependencies:

```language-bash
user@hostname ~ $ sudo npm install -g git+https://github.com/JarvusInnovations/Emergence
```

## Start Emergence

`npm -g` installed the kernel's startup script to `/usr/bin/emergence-kernel`. You can now launch it manually, or install the init script:

```language-bash
user@hostname ~ $ sudo wget http://emr.ge/dist/ubuntu/emergence-kernel.service -O /etc/systemd/system/emergence-kernel.service
user@hostname ~ $ sudo service emergence-kernel start
user@hostname ~ $ sudo systemctl enable emergence-kernel
```

## Create a site

You can now open your web browser to take control of Emergence and create your first site: [http://127.0.0.1:9083](http://127.0.0.1:9083). If you're installing to a remote machine, replace 127.0.0.1 with your remote IP or hostname.

When prompted, log in with the username and password <kbd>admin</kbd> / <kbd>admin</kbd>.

## Secure your installation

Once you confirm that you are able to access the control panel, use the `htpasswd` tool provided in npm to delete the default admin account and create your own. Then restart the Emergence kernel to apply the changes:

```language-bash
user@hostname ~ $ sudo npm install -g htpasswd
user@hostname ~ $ sudo htpasswd -D /emergence/admins.htpasswd admin
user@hostname ~ $ sudo htpasswd -s /emergence/admins.htpasswd myusername
user@hostname ~ $ sudo service emergence-kernel restart
```

## (Optional) install Sencha CMD

Enter <kbd>/usr/local/bin</kbd> as the install path when prompted by Sencha's CMD installer:

```language-bash
user@hostname ~ $ sudo apt-get install openjdk-7-jre ruby1.9.3 unzip
user@hostname ~ $ wget http://cdn.sencha.com/cmd/3.1.2.342/SenchaCmd-3.1.2.342-linux-x64.run.zip
user@hostname ~ $ unzip SenchaCmd-*-linux-x64.run.zip
user@hostname ~ $ chmod +x SenchaCmd-*-linux-x64.run
user@hostname ~ $ sudo ./SenchaCmd-*-linux-x64.run
user@hostname ~ $ sudo mkdir /usr/local/bin/Sencha/Cmd/repo
user@hostname ~ $ sudo chown www-data:www-data -R /usr/local/bin/Sencha/Cmd/repo
```
