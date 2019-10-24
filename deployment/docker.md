# Deploy a Site with Docker

There are two methods, both based on [Chef Habitat plans](./habitat.md) for deploying an emergence site with Docker:

- Build Habitat packages for the app and composite and then have Habitat export a Docker container from the composite package
- Write a Dockerfile that handles building both Habitat packages internally and make deliberate use of Docker layers

The former lets you stay out of the weeds of Docker container builds and provides versatile Habitat packages you can deploy in a variety of ways. The later lets you stay out of the weeds of Habitat and provides a hyper-optimized set of container image layers.

## Building from Habitat plans

```bash
export HAB_BLDR_CHANNEL="emergence-studio-0.6"
export HAB_ORIGIN="myorigin"

hab pkg build .
hab pkg build --reuse habitat/composite
env $(cat results/last_build.env | xargs) bash -c 'hab pkg export docker results/$pkg_artifact'
```

## Building from a Dockerfile

This Dockerfile can be used as-is for any emergence project repository with `habitat/plan.sh` and `habitat/composite/plan.sh` in place per the [Deploy a Site with Chef Habitat](./habitat.md) guide. Replace every occurance of the placeholder `myorigin` with any origin name you'd like to prefix your built packages with. In this scenario it need not be externally registered with any bldr server.

```Dockerfile
FROM jarvus/habitat-compose:latest as habitat
ARG HAB_LICENSE=no-accept
ENV HAB_LICENSE=$HAB_LICENSE
ENV STUDIO_TYPE=Dockerfile
ENV HAB_ORIGIN=myorigin
RUN hab origin key generate
# pre-layer all external runtime plan deps (pulling from pre-release channel)
COPY habitat/plan.sh /habitat/plan.sh
RUN hab pkg install \
    --channel=emergence-studio-0.6 \
    $({ cat '/habitat/plan.sh' && echo 'echo "${pkg_deps[@]/$pkg_origin\/*/}"'; } | hab pkg exec core/bash bash) \
    && hab pkg exec core/coreutils rm -rf /hab/{artifacts,src}/
# pre-layer all external runtime composite deps
COPY habitat/composite/plan.sh /habitat/composite/plan.sh
RUN hab pkg install \
    $({ cat '/habitat/composite/plan.sh' && echo 'echo "${pkg_deps[@]/$pkg_origin\/*/}"'; } | hab pkg exec core/bash bash) \
    && hab pkg exec core/coreutils rm -rf /hab/{artifacts,src}/


FROM habitat as projector
# pre-layer all build-time plan deps
RUN hab pkg install \
    core/hab-plan-build \
    $({ cat '/habitat/plan.sh' && echo 'echo "${pkg_build_deps[@]/$pkg_origin\/*/}"'; } | hab pkg exec core/bash bash) \
    && hab pkg exec core/coreutils rm -rf /hab/{artifacts,src}/
# pre-layer all build-time composite deps
RUN hab pkg install \
    $({ cat '/habitat/composite/plan.sh' && echo 'echo "${pkg_build_deps[@]/$pkg_origin\/*/}"'; } | hab pkg exec core/bash bash) \
    && hab pkg exec core/coreutils rm -rf /hab/{artifacts,src}/
# build application
COPY . /src
RUN hab pkg exec core/hab-plan-build hab-plan-build /src
RUN hab pkg exec core/hab-plan-build hab-plan-build /src/habitat/composite


FROM habitat as runtime
# install .hart artifact from builder stage
COPY --from=projector /hab/cache/artifacts/$HAB_ORIGIN-* /hab/cache/artifacts/
RUN hab pkg install /hab/cache/artifacts/$HAB_ORIGIN-* \
    && hab pkg exec core/coreutils rm -rf /hab/{artifacts,src}/


# configure persistent volumes
RUN hab pkg exec core/coreutils mkdir -p /hab/svc/{mysql,myapp}/data \
    && hab pkg exec core/coreutils chown hab:hab -R /hab/svc/{mysql,myapp}/data

VOLUME ["/hab/svc/mysql/data", "/hab/svc/myapp/data"]


# configure entrypoint
ENTRYPOINT ["hab", "sup", "run"]
CMD ["myorigin/myapp-composite"]
```

With this file in place, you can run the following command to build a container without Habitat installed:

```bash
docker build . \
    --build-arg HAB_LICENSE=accept-no-persist \
    -t myorigin/myapp
```
