# Development: Using Studio

This guide shows you how to set up a local Studio environment for interactively developing your site.

## In this article

- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Add `.studiorc` to Repository](#add-studiorc-to-repository)
- [Enter Studio](#enter-studio)
- [Load a Database Service](#load-a-database-service)
- [Load a Runtime Service](#load-a-runtime-service)
- [Load an HTTP Service](#load-an-http-service)
- [Studio Commands Reference](#studio-commands-reference)

## Introduction

[Emergence Studio](https://github.com/EmergencePlatform/studio/) uses [Chef Habitat Studio](https://www.habitat.sh/docs/developing-packages/#plan-builds) to provide an isolated, disposable, and interactive command-line environment for working on your site.

In this and other guides, we will focus on running Linux-like Studio environments contained by Docker, as that provides the most consistent developer experience across platforms. On Linux machines, Chef Habitat [also can provide a chroot-contained Studio type](https://www.habitat.sh/docs/developing-packages/#managing-the-studio-type-docker-linux-windows) that is faster, smaller, persistent by default, and exposes all network ports. Chroot studios offer lower overhead in CI and controlled environments, and will work just as well with Emergence Studio, but require more awareness of what else is in the environment to avoid conflicts.

## Prerequisites

- Complete the [prerequisites covered in Development: Getting Started](./getting_started.md#prerequisites)

## Add `.studiorc` to Repository

While entering a Chef Habitat Studio, any `.studiorc` script present in the root of the directory where the `hab studio enter` command is run will be sourced and executed automatically during the Studio's initialization, before the interactive command line is presented.

Use this `.studiorc` template as a starting point in the root of any emergence site project:

```bash
#!/bin/bash
hab pkg install emergence/studio
source "$(hab pkg path emergence/studio)/studio.sh"
```

This script installs and loads the Emergence Studio environment and commands. You may add any additional code here to extend the studio environment for your project.

## Enter Studio

```bash
# start from the root of your project repository
cd /path/to/my-repo/

# use your chosen/registered origin name
export HAB_ORIGIN="myorigin"

# pass options to Docker to open ports, persist mysql data, name container
export HAB_DOCKER_OPTS="
    -p 7080:80
    -p 7043:443
    -p 7036:3306
    -p 7099:19999
    -v mysite-mysql-data:/hab/svc/mysql/data
    --name mysite-studio
"

# launch and enter interactive Habitat studio, forcing Docker mode
hab studio enter -D
```

## Load a Database Service

Start a database service before any other. Your runtime service instance can have its `database` slot bound to an instance of `core/mysql`, `jarvus/mysql-remote`, or any other Chef Habitat service with compatible config exposure leading to a MySQL server connection.

Any approach must export `$DB_SERVICE` and `$DB_DATABASE` into your studio environment.

### Use `core/mysql` to run a local database

```bash
start-mysql
```

That's it! A local MySQL instance will be set up using [`core/mysql`](https://github.com/habitat-sh/core-plans/tree/master/mysql) with persistent data stored under `/hab/svc/mysql/data`. The example `HAB_DOCKER_OPTS` above includes mounting this path to a named volume and exposing the MySQL port to your host machine on `7036`.

Use `hab svc status` and `sup-log` to verify the database service is up and running without any errors. If you see any permission denied errors under `sup-log`, that likely indicates the volume mounted at `/hab/svc/mysql/data` did not initialize with permissions adequate for the MySQL service to write there, and can be fixed for the lifetime of that volume by running `chown hab:hab -R /hab/svc/mysql/data`

### Use `jarvus/mysql-remote` to connect a remote database

```bash
start-mysql-remote "mydatabasename"
```

This studio command will launch an editor to populate `/hab/user/mysql-remote/config/user.toml` with connection details and then load a [`jarvus/mysql-remote`](https://github.com/JarvusInnovations/habitat-plans/tree/master/mysql-remote) service instance.

## Load a Runtime Service

With a database service successfully loaded, you can load a runtime service. The runtime service provides the PHP worker pool and loads your site code into it.

```bash
start-runtime
```

This studio command will load an instance of the generic [`emergence/php-runtime`](https://github.com/EmergencePlatform/php-runtime) service that any site can be loaded into. The `$DB_SERVICE` environment variable exported when you ran one of the `start-mysql*` studio commands previously will be used to bind this runtime instance's `database` slot to your database instance.

The guide [Development: Build a Site-specific Service](development/site-specific-service.md) covers how to build and use your own runtime service package that can extend and bundle your site into its own deployable artifact.

Any approach must export `$EMERGENCE_RUNTIME` into your studio environment.

## Load an HTTP Service

Finally, with the database and runtime services successfully loaded, you can load an HTTP service to expose your application.

```bash
start-http
```

This studio command will load an instance of the generic [`emergence/nginx`](https://github.com/EmergencePlatform/habitat-plans/tree/master/nginx) service. The `$EMERGENCE_RUNTIME` environment variable exported when you ran the `start-runtime` studio command previously will be used to bind this nginx instance's `backend` slot to your runtime instance.

The example `HAB_DOCKER_OPTS` above includes exposing nginx's HTTP and HTTPS ports to your host machine on `7080` and `7043` respectively.

## Studio Commands Reference

### Runtime

- `update-site` — Projects your site tree with Hologit and loads it into the current runtime service instance
  - Affected by `$EMERGENCE_REPO`, `$EMERGENCE_HOLOBRANCH`, and `$EMERGENCE_FETCH` environment variables
- `watch-site` — Like `update-site`, but runs until terminated, watching for file changes and re-updating the site automatically
- `shell-runtime` — Opens an interactive [PsySH](https://psysh.org/) shell under the same environment used by the currently-loaded runtime instance
- `switch-site /path/to/repository/` — Switches what site repository will be used next time you run `update-site` (the default is whichever `.studiorc` was loaded from)
- `enable-xdebug 192.168.1.1` — Enables using a remote Xdebug debugger at an IP/hostname reachable from inside the container
- `enable-runtime-update` — When using a site-specific runtime service, enables using `update-site` as you can with the generic `emergence/php-runtime` service (instead of using the site build bundled with the service).

### Database

- `load-sql [-|file...|URL|site] [database]` — Load one or more .sql files into the current database instance
- `dump-sql [database] > file.sql` — Dump the current database to SQL
- `reset-mysql` — Drop and recreate empty the current database
- `promote-user <username> [account_level]` — Promote a user in the database to a higher account level after registration
- `shell-mysql` — Open an interactive MySQL client shell under the currently-loaded database instance
