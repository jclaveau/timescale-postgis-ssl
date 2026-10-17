# SSL-enabled Postgres DB image + TimescaleDB and PostGIS

This repository contains the logic to build SSL-enabled Postgres images with TimescaleDB and PostGIS extensions already installed.

When you deploy the TimescaleDB + PostGIS DB service from the Railway-published template, the image that is used is built from this repository.

[![Deploy on Railway](https://railway.app/button.svg)](https://railway.app/template/timescaledb-postgis)

### Why though?

The [timescale/timescaledb-ha](https://hub.docker.com/r/timescale/timescaledb-ha) image in Docker hub does not come with SSL baked in.

Since this could pose a problem for applications or services attempting to connect, we decided to roll our own image with SSL enabled.

### How does it work?

The Dockerfiles contained in this repository start with the `timescale/timescaledb-ha` image as base.  Then the `init-ssl.sh` script is copied into the `docker-entrypoint-initdb.d/` directory to be executed upon initialization.

#### Wrapper script

You will see in the Dockerfiles that the `ENTRYPOINT` is replaced with a `wrapper.sh` script.  The only logic the wrapper executes is to `unset` the `PGHOST` environment variable prior to executing the default entrypoint.  This logic is specific to Railway and allows us to use the `PGHOST` environment variable to enable the Database View.

If this logic conflicts with your use-case in any way, please feel free to spin up your own version of the image without the wrapper script.

#### Cert expiry
By default, the cert expiry is set to 820 days.  You can control this by configuring the `SSL_CERT_DAYS` environment variable as needed.

### A note about running the images locally

The images built from this repo and workflow do not contain support for ARM.  Check out the Dockerfiles that build them and feel free to copy the logic to build your own image with the appropriate base containing support for ARM.
### The alpine line, for CI

`Dockerfile.alpine-pg18-ts2.29` builds a second, unrelated package:
`ghcr.io/jclaveau/timescaledb-alpine:pg18-ts2.29`. The name says alpine because
the tag only ever says which PostgreSQL and which TimescaleDB an image carries,
and nothing about which libc built it.

It exists to make a CI job's image pull cheap: 129MB compressed against the
649MB of the `timescaledb-ha` line, which is ~7s instead of ~31s per job.

It starts from `timescale/timescaledb` (alpine, musl) and **compiles**
`timescaledb_toolkit` and `pg_textsearch` from source, because the `.so` files
the HA image ships link `libc.so.6` and cannot be copied onto musl. So it offers
the same extension set a database created from a production dump needs, built
from different binaries.

What it does not have, because nothing in CI asks for it: PostGIS, the SSL
certificate init, the `wrapper.sh` `PGHOST` unset, Patroni and pgBackRest. It
also keeps only the running TimescaleDB version's libraries and upgrade scripts,
so it can initialise an empty data directory and nothing else — which is all a
CI database ever is. **Do not point a persistent volume at it.**
