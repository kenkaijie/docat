![docat](doc/assets/docat-teaser.png)

**Host your docs. Simple. Versioned. Fancy.**

[![build](https://github.com/docat-org/docat/workflows/docat%20ci/badge.svg)](https://github.com/docat-org/docat/actions)
[![Gitter](https://badges.gitter.im/docat-docs-hosting/community.svg)](https://gitter.im/docat-docs-hosting/community?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)

## Note for Modification

This repository is a fork of the original to add path prefix support. If did not mean to
come here, the original repository can be found at https://github.com/docat-org/docat.

The modifications here are only present for version `0.3.0`.

### Proxy Support Changes

The following locations require path prefix changes:

- (done) Python API needs to be aware it is running under a proxy, so the front-end
  requests can be properly performed. (use `root_path` option)
- The frontend JS must be capable of accepting a path prefix. (Vue config `publicPath`).

This is achieved by setting 2 variables when building:

`VUE_APP_BASE_URL`
  Used to set the base URL, is a direct replacement for the original HOST and PORT and
  is used to point to the base URL hosting this web server (and API). It does not
  currently support hosting the frontend and backend separately.
`VUE_APP_PATH_PREFIX`
  Used if this site is hosted on a non-root portion of the web server. Will update all
  references correctly.

The caveat here is that when building the container, the system will take a snapshot of
the variables when `yarn build` is called.

#### How to Configure

The main configuration that needs to be done is to create a custom docker image that is
based on this one. In the example below, the image has been built (without unset
`VUE_APP_BASE_URL` and `VUE_APP_PATH_PREFIX`) and is called `ken-docat`.

The key points are:
  - The image provides the source code to the website at the location `/app/web-src/` to
    allow for modification.
  - You will need to set the environment to use the path prefix you wish to set and
    rebuild the container.
  - The option `--root-path` must be provided to the `CMD` by overriding the existing
    parameter.

An example can be seen below, where you can use the build arguments `DOCAT_PATH_PREFIX`
and `DOCAT_BASE_URL` to apply changes needed for hosting behind a proxy.

```dockerfile
FROM ghcr.io/kenkaijie/docat as base

# Regenerated frontend with new prefix paths
FROM node:16-alpine3.15 as build-new-frontend
COPY --from=base /app/web-src/ ./

ARG DOCAT_PATH_PREFIX
ARG DOCAT_BASE_URL

ENV VUE_APP_PATH_PREFIX=$DOCAT_PATH_PREFIX
ENV VUE_APP_BASE_URL=$DOCAT_BASE_URL

RUN yarn install --frozen-lockfile
RUN yarn lint
RUN yarn run test:unit
RUN yarn build

# Copy new webpage to the container, overriding the existing system
FROM ghcr.io/kenkaijie/docat

ARG DOCAT_PATH_PREFIX
ENV VUE_APP_PATH_PREFIX=${DOCAT_PATH_PREFIX}
COPY --from=build-new-frontend /dist /var/www/html

CMD ["sh", "-c", "nginx && .venv/bin/python \
                           -m uvicorn \
                           --root-path ${VUE_APP_PATH_PREFIX} \
                           --host 0.0.0.0 \
                           --port 5000 \
                           docat.app:app"]
```

## Getting started

The simplest way is to build and run the docker container,
you can optionally use volumes to persist state:

```sh
# run container in background and persist data (docs, nginx configs and tokens database)
# use 'ghcr.io/docat-org/docat:unstable' to get the latest changes
mkdir -p docat-run/db && touch docat-run/db/db.json
docker run \
  --detach \
  --volume $PWD/docat-run/doc:/var/docat/doc/ \
  --volume $PWD/docat-run/db/db.json:/app/docat/db.json \
  --publish 8000:80 \
  ghcr.io/docat-org/docat
```

*Alternative:* Mount a dedicated directory to host `db.json` :

```sh
# run container in background and persist data (docs, nginx configs and tokens database)
# use 'ghcr.io/docat-org/docat:unstable' to get the latest changes
mkdir -p docat-run/db && touch docat-run/db/db.json
docker run \
  --detach \
  --volume $PWD/docat-run/doc:/var/docat/doc/ \
  --volume $PWD/docat-run/db:/var/docat/db/ \
  --env DOCAT_DB_PATH=/var/docat/db/db.json
  --publish 8000:80 \
  ghcr.io/docat-org/docat
```

Go to [localhost:8000](http://localhost:8000) to view your docat instance:

![docat screenshot](doc/assets/docat-screenshot.png)

### Local Development

For local development, first configure and start the backend (inside the `docat/` folder):

```sh
# create a folder for local development (uploading docs)
DEV_DOC_PATH="$(mktemp -d)"

# install dependencies
poetry install

# run the local development version
DOCAT_SERVE_FILES=1 DOCAT_DOC_PATH="$DEV_DOC_PATH" poetry run python -m docat
```

After this you need to start the frontend (inside the `web/` folder):

```sh
# install dependencies
yarn install --frozen-lockfile

# run the web app
yarn serve
```

For more advanced options, have a look at the
[backend](docat/README.md) and [web](web/README.md) docs.

### Push Documentation to docat

The preferred way to push documentation to a docat server is using the [docatl](https://github.com/docat-org/docatl)
command line application:

```sh
docatl push --host http://localhost:8000 /path/to/your/docs PROJECT VERSION
```

There are also docker images available for CI systems.

#### Using Standard UNIX Command Line Tools

If you have static html documentation or use something like
[mkdocs](https://www.mkdocs.org/), [sphinx](http://www.sphinx-doc.org/en/master/), ...
to generate your documentation, you can push it to docat:

```sh
# create a zip of your docs
zip -r docs.zip /path/to/your-docs
# upload them to the docat server (replace PROJECT/VERSION with your projectname and the version of the docs)
curl -X POST -F "file=@docs.zip" http://localhost:8000/api/PROJECT/VERSION
```

When you have multiple versions you may want to tag some version as **latest**:

```sh
# tag the version VERSION of project PROJECT as latest
curl -X PUT http://localhost:8000/api/PROJECT/VERSION/tags/latest
```

## Advanced `config.json`

It is possible to configure some things after the fact.

1. Create a `config.json` file
2. Mount it inside your docker container `--volume /path/to/config.json:/var/www/html/config.json`

Supported config options:

* headerHTML
