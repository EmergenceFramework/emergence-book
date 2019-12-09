# Deploy a Site with Chef Habitat

## Initialize a Chef Habitat plan for the PHP application:

Create `habitat/plan.sh`

```bash
pkg_name=myapp
pkg_origin=myorigin
pkg_maintainer="First Last <human@example.org>"
pkg_scaffolding=emergence/scaffolding-site

pkg_version() {
  scaffolding_detect_pkg_version
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
pkg_name="${composite_app_pkg_name}-composite"
pkg_origin=myorigin
pkg_maintainer="First Last <human@example.org>"
pkg_scaffolding=emergence/scaffolding-composite

# uncomment to use remote mysql instead of local core/mysql service:
# composite_mysql_pkg=jarvus/mysql-remote

pkg_version() {
  scaffolding_detect_pkg_version
}
```

Create `habitat/composite/default.toml`

```toml
[services.app.config]
default_timezone = "America/New_York"

# declare a basic username+password for core/mysql
[services.mysql.config]
app_username = "admin"
app_password = "admin"
bind = "0.0.0.0"
```

## Open a Habitat Studio:

```bash
HAB_DOCKER_OPTS="-p 7080:7080" hab studio enter -D
```
