# Development: Build a Site-specific Service

This guide shows you how to develop a [Chef Habitat package](https://www.habitat.sh/docs/developing-packages/) containing a complete build of one site that is ready to [run as a service](https://www.habitat.sh/docs/using-habitat/) under the Chef Habitat Supervisor.

## In this article

- [Introduction](#introduction)
- [Initialize `habitat/` tree](#initialize-habitat-tree)
- [Build runtime package](#build-runtime-package)
- [Enter Studio](#enter-studio)

## Introduction

After you complete this, you will be able to build a version of the site's source tree into a `.hart` Chef Habitat package build artifact file that is ready to be installed into any environment with the Chef Habitat Supervisor running, uploaded to a [Chef Habitat Builder](https://www.habitat.sh/docs/using-builder/) server, or [exported into a Docker container image or other formats](https://www.habitat.sh/docs/developing-packages/#pkg-exports). The `.hart` file will contain a complete manifest of exact versions for all dependencies

This approach also enables you to bundle any additional environmental dependencies, build steps, commands, configuration, or lifecycle hooks into your site's runtime environment via Chef Habitat's facilities.

## Initialize `habitat/` Tree

- Before you begin, complete the steps covered in [Deployment: Deploy a Site with Chef Habitat](../deployment/habitat.md) to initialize the `habitat/` configuration tree defining the Chef Habitat package(s) for your site.

## Build Runtime Package

- Use the standard `build` command provided by Chef Habitat Studio as you would to build any other Chef Habitat package:

    ```bash
    cd "${EMERGENCE_REPO}"
    build
    ```

## Enter Studio

A Chef Habitat Studio will provide an isolated, disposable, and interactive command-line environment for working on your package.

When building your own Chef Habitat packages, be sure to set `HAB_ORIGIN` to the company/team/personal name you want to prefix your builds with. If you plan to publish your packages to a Chef Habitat Builder server, it should be an origin name you've registered on that server. Otherwise the name is entirely for your own local organization and can be set to any string you could use for a directory name.

- Follow all steps in the [Using Studio](studio.md) guide, but at the *Load a Runtime Service* step, pass the package identifier for your site-specific runtime:

    ```bash
    start-runtime myorigin/mypackage
    ```

## Update Site Without Rebuilding Runtime

The most accurate way to test your site-specific service artifact is to use the `build` command to entirely rebuild the package each time. Once an instance of it is running though, you may update it directly from your site sources just like you would with the generic `emergence/php-runtime` service.

- Enable live-updating of site-specific service:

    ```bash
    enable-runtime-update
    ```

- Use `update-site` (or `watch-site`) like normal:

    ```bash
    update-site
    ```
