# Migrate a Site to a Holobranch

## Branch name

For this guide, the branch name `emergence/vfs-site/v1` will be used as the target for projection, but any valid git branch name could be used.

This example follows a convention of using the `emergence/vfs-site/` prefix for holobranch projections intended to be loaded into VFS-powered Emergence sites. So a branch named `emergence/vfs-site/v1` might indicate VFS-compatible site projections from a v1.x release stream, while `emergence/vfs-site/develop` might be used to indicate such from a `develop` branch.

## Initialize projected branch with a VFS snapshot (optional)

When a new holobranch is projected to a branch for the first time, an empty initialization commit will be generated automatically. Optionally, you can manually initialize the branch you'll project to with a snapshot of the existing VFS composite first.

### Full history via git gateway

This relies on the `.git` gateway merged into [skeleton-v1 v1.11.0](https://github.com/JarvusInnovations/emergence-skeleton/releases/tag/v1.11.0), so the target site may need to be updated via traditional VFS means first to acquire the latest code in skeleton-v1/skeleton-v2. By initializing your projected branch this way, you'll be able to capture and review the effective set of tree changes that happen in the switchover.

An issue with the git gateway currently prevents adding it as a new remote in an existing repo, but cloning into a fresh repo works:

```bash
cd ..
git clone http://my-site.example.org/.git
```

Then from that local clone, you can fetch the composite `master` branch as the start of your projection target:

```
cd ./my-repo
git fetch ../my-site.example.org master:emergence/vfs-site/v1
```

### Remote snapshot via WebDAV

If the git gateway is unavailable on the target site, or a history is not needed, a complete snapshot can be captured remotely over the standard `/develop` interface using the `@emergence/source-http-legacy` NPM package:

```bash
# install command globally
npm install -g @emergence/source-http-legacy

# build snapshot tree of entire VFS
emergence-source-http-legacy pull my-site.example.org

# using tree hash output at end of pull command
TREE_HASH=4b825dc642cb6eb9a060e54bf8d69288fbee4904
git commit-tree -m "snapshot mysite.example.org" "${TREE_HASH}"

# using commit hash output at end of commit-tree command
COMMIT_HASH=4b825dc642cb6eb9a060e54bf8d69288fbee4904
git update-ref refs/heads/emergence/vfs-site/v1
```

## Update git mappings to include all trees

Create or update a `php-config/Git.config.d/*` file mapping the VFS to projected git branch:

```php
<?php

Git::$repositories['my-repo'] = [
    'remote' => 'git@github.com:my-org/my-repo.git',
    'originBranch' => 'emergence/vfs-site/v1',
    'workingBranch' => 'emergence/vfs-site/v1',
    'trees' => [
        'api-docs',
        'console-commands',
        'cypress',
        'data-exporters',
        'dwoo-plugins',
        'event-handlers',
        'html-templates',
        'php-classes',
        'php-config',
        'php-migrations',
        'phpunit-tests',
        // 'sencha-workspace',
        'site-root',
        'site-tasks',
        'webapp-builds',
        // 'webapp-plugin-builds',
    ]
];
```

## Project holobranch and push to server

```bash
git holo project --fetch='*' emergence-vfs-site --commit-to=emergence/vfs-site/v1
git push origin emergence/vfs-site/v1
```

## Reconfigure site

You could edit the `site.json` file directly and then restart the kernel to reconfigure the site, or use an HTTP client like HTTPie to make the changes via the kernel API:

```bash
# install HTTPie command
sudo hab pkg install core/python
sudo hab pkg exec core/python pip install httpie httpie-unixsocket
sudo hab pkg binlink core/python http

# verify connection to kernel API and site selection
export SITE_HANDLE="my-site"
sudo http \
    GET http+unix://%2Femergence%2Fkernel.sock/sites/${SITE_HANDLE}

# null out parent_hostname and parent_key
sudo http \
    PATCH http+unix://%2Femergence%2Fkernel.sock/sites/${SITE_HANDLE} \
    parent_hostname:=null \
    parent_key:=null
```

## Purge parent cache

Open MySQL shell for the site:

```bash
emergence-mysql-shell ${SITE_HANDLE}
```

Then run the following queries:

```sql
DELETE FROM _e_files WHERE CollectionID IN (SELECT ID FROM _e_file_collections WHERE Site = "Remote");

DELETE FROM _e_file_collections WHERE Site = "Remote";
```

## Import `vfs-site` branch

Switch site repo to new vfs-site branch:

```bash
export SOURCE_NAME="mysourcename"
sudo emergence-git-shell ${SITE_HANDLE} ${SOURCE_NAME}
git fetch --all
git checkout emergence/vfs-site/v1
pwd
exit
```

Change into that directory and open a PHP shell:

```bash
cd "/emergence/sites/${SITE_HANDLE}/site-data/git/${SOURCE_NAME}"
emergence-shell ${SITE_HANDLE}
```

Then run the following PHP code:

```php
$collections = array_diff(glob('*'), ['sencha-workspace']);
foreach ($collections as $collection) {
    echo "Importing $collection\n\t";
    echo http_build_query(Emergence_FS::importTree($collection, $collection));
    echo "\n\n";
}

echo NestingBehavior::repairTable(SiteCollection::class, 'PosLeft', 'PosRight')." collections renested\n\n";
```

Then stop/start PHP to clear all caches and run repair on the VFS, and then stop/start PHP again for good measure