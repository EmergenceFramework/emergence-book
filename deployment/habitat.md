# Deploy a Site with Chef Habitat

## Initialize a Chef Habitat plan for the PHP application:

Create `habitat/plan.sh`

```bash
pkg_name=myapp
pkg_origin=myorigin
pkg_version="0.1.0"
pkg_maintainer="First Last <human@example.org>"
pkg_build_deps=(
  jarvus/hologit
  jarvus/toml-merge
)
pkg_deps=(
  emergence/php-runtime
)


pkg_binds=(
  [database]="port username password"
)

pkg_exports=(
  [port]=network.port
)


do_setup_environment() {
  set_runtime_env -f PHPRC "${pkg_svc_config_install_path}"
}

do_before() {
  # adjust PHP_EXTENSION_DIR after env is initially built
  set_runtime_env -f PHP_EXTENSION_DIR "${pkg_svc_config_install_path}/extensions-${PHP_ZEND_API_VERSION}"
}

do_build() {
  pushd "${PLAN_CONTEXT}/../" > /dev/null
  build_tree_hash="$(git holo project --fetch --working emergence-site)" # use working tree
  # build_tree_hash="$(git holo project --fetch --ref="v${pkg_version}" emergence-site)" # use version tag
  popd > /dev/null
}

do_install() {
  mkdir "${pkg_prefix}/site"
  git archive --format=tar "${build_tree_hash}" | (cd "${pkg_prefix}/site" && tar xf -)
}

do_build_config() {
  do_default_build_config

  build_line "Merging php-runtime config"
  cp -nrv "$(pkg_path_for emergence/php-runtime)"/{config_install,config,hooks} "${pkg_prefix}/"
  toml-merge \
    "$(pkg_path_for emergence/php-runtime)/default.toml" \
    "${PLAN_CONTEXT}/default.toml" \
    > "${pkg_prefix}/default.toml"
}

do_strip() {
  return 0
}

```

## Initialize default configuration for the PHP application:

Create `habitat/default.toml`

```toml
[sites.default]
database = "myapp"
```

## Initialize a Chef Habitat plan for running PHP application + nginx + database:

Create `habitat/composite/plan.sh`

```bash
composite_base_pkg_name=myapp
pkg_name="${composite_base_pkg_name}-composite"
pkg_origin=myorigin
pkg_maintainer="First Last <human@example.org>"
pkg_build_deps=(
  jarvus/toml-merge
)
pkg_deps=(
  "${pkg_origin}/${composite_base_pkg_name}"
  jarvus/habitat-compose
  jarvus/mysql-remote
  emergence/nginx
)

pkg_svc_user="root"
pkg_svc_run="habitat-compose ${pkg_svc_config_path}/services.json"


pkg_version() {
  echo "$(pkg_path_for ${pkg_origin}/${composite_base_pkg_name})" | cut -d / -f 6
}

# implement build workflow
do_before() {
  do_default_before
  update_pkg_version
}

do_build() {
  return 0
}

do_install() {
  return 0
}

do_build_config() {
  do_default_build_config

  build_line "Merging habitat-compose config"
  cp -nrv "$(pkg_path_for jarvus/habitat-compose)/config" "${pkg_prefix}/"
  toml-merge \
    "$(pkg_path_for jarvus/habitat-compose)/default.toml" \
    "${PLAN_CONTEXT}/default.toml" \
    > "${pkg_prefix}/default.toml"
}

do_strip() {
  return 0
}
```

Create `habitat/composite/default.toml`

```toml
[services.app]
    pkg_name = "myapp"
    [services.app.binds]
        database = "mysql"
    [services.app.config]
        default_timezone = "America/New_York"

[services.mysql]
    pkg_ident = "jarvus/mysql-remote" # or core/mysql for local

[services.nginx]
    pkg_ident = "emergence/nginx"
    [services.nginx.binds]
        backend = "app"
```

## Open a Habitat Studio:

```bash
HAB_DOCKER_OPTS="-p 7080:7080" hab studio enter -D
```
