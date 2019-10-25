# Getting Started with Development

This guide shows you the minimal steps required run an emergence project locally, make changes to it, and see the result.

## In this article

- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Clone Project via Git](#clone-project-via-git)
- [Launch Studio via Habitat](#launch-studio-via-habitat)
- [Start Runtime and Build Project](#start-runtime-and-build-project)
- [Load Fixture Data (optional)](#load-fixture-data-optional)
- [Create User Account (optional)](#create-user-account-optional)
- [Running Tests](#running-tests)

## Introduction

## Prerequisites

Before you begin, you'll need Docker and Habitat set up on your workstation:

1. **Install Docker**

    On Mac and Windows workstations, Docker must be installed to use habitat. On Linux, Docker is optional.

    - [Download *Docker for Mac*](https://store.docker.com/editions/community/docker-ce-desktop-mac)
    - [Download *Docker for Windows*](https://store.docker.com/editions/community/docker-ce-desktop-windows)
    - [Install Docker on Ubuntu](https://docs.docker.com/install/linux/docker-ce/ubuntu/)

2. **Install Chef Habitat on your system**

    Chef Habitat is a tool for automating all the build and runtime workflows for applications, in a way that behaves consistently across time and environments. An application automated with Habitat can be run on any operating system, connected to other applications running locally or remotely, and deployed to either a container, virtual machine, or bare-metal system.

    Installing Habitat only adds one binary to your system, `hab`, and initializes the `/hab` tree.

    ```bash
    curl -s https://raw.githubusercontent.com/habitat-sh/habitat/master/components/hab/install.sh | sudo bash
    hab --version # should report 0.85.0 or newer
    ```

3. **Configure the `hab` client for your user**

    Setting up Habitat will interactively ask questions to initialize `~/.hab`.

    This command must be run once per user that will use `hab`:

    ```bash
    hab setup
    ```

## Clone Project via Git

In this example, the [slate-cbl](https://github.com/SlateFoundation/slate-cbl) project is cloned, but you might use any repository/branch containing an emergence project:

```bash
git clone --recursive -b develop git@github.com:SlateFoundation/slate-cbl.git
```

The `--recursive` option is used so that any submodule repositories are also cloned.

## Launch Studio via Habitat

1. Change into project's cloned directory

    ```bash
    cd ./slate-cbl
    ```

1. Launch Studio

    On any system, launch a studio with:

    ```bash
    HAB_DOCKER_OPTS="-p 7080:80 -p 3306:3306" \
        hab studio enter -D
    ```

    The `HAB_DOCKER_OPTS` environment variable here allows you to use any options supported by `docker run`, in this case forwarding ports for the web server and MySQL server from inside the container to your host machine.

    Review the notes printed to your terminal at the end of the studio startup process for a list of additional commands provided by your project's `.studiorc`

## Start Runtime and Build Site

1. Start environment services

    Use the studio command `start-all` to launch the http server (nginx), the application runtime (php-fpm), and a local mysql server:

    ```bash
    start-all
    ```

    At this point, you should be able to open [localhost:7080](http://localhost:7080) and see the error message `Page not found`.

1. Build environment

    To build the entire environment and load it, use the studio command `update-site`:

    ```bash
    update-site
    ```

    At this point, [localhost:7080](http://localhost:7080) should display the current build of the site

## Load Fixture Data (optional)

```bash
# clone fixture branch into git-ignored .data/ directory
git clone -b cbl/competencies https://github.com/SlateFoundation/slate-fixtures.git .data/fixtures

# load all .sql files from fixture
cat .data/fixtures/*.sql | load-sql -
```

## Create User Account (optional)

1. Enable user registration form (optional)

    If your project has registration disabled by default, you might want to enable it so you can register:

    ```bash
    # write class configuring enabling registration
    mkdir -p php-config/Emergence/People
    echo '<?php Emergence\People\RegistrationRequestHandler::$enableRegistration = true;' > php-config/Emergence/People/RegistrationRequestHandler.config.php

    # rebuild environment
    update-site
    ```

1. Promote registered user to developer (optional)

    After visiting [`/register`](http://localhost:7080/register) and creating a new user account, you can use the studio command `promote-user` to upgrade the user account you just registered to the highest access level:

    ```bash
    promote-user <myuser>
    ```

After editing code in the working tree, run the studio command `update-site` to rebuild and update the environment. A `watch-site` command is also available to automatically rebuild and update the environment as changes are made to the working tree.

## Running Tests

[Cypress](https://www.cypress.io/) is used to provide browser-level full-stack testing. The `package.json` file at the root of the repository specifies the dependencies for running the test suite and all the configuration/tests for Cypress are container in the `cypress/` tree at the root of the repository.

To get started, from a terminal **outside the studio** in the root of the repository:

```bash
# install development tooling locally
npm install

# launch cypress app
npm run cypress:open
```
