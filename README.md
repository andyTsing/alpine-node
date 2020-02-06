Minimal Node.js Docker Images
-----------------------------

Versions v13.8.0, v12.15.0, v10.19.0, v8.17.0, v6.17.1, v4.9.1, v0.12.18 and v0.10.48 –
built on [Alpine Linux](https://alpinelinux.org/).

All versions use the one [mhart/alpine-node](https://hub.docker.com/r/mhart/alpine-node/) repository,
but each version aligns with the following tags (ie, `mhart/alpine-node:<tag>`). The sizes are for the
*unpacked* images as reported by Docker – compressed sizes are about 1/3 of these:

- Full install built with npm and yarn:
  - `latest`, `13`, `13.8`, `13.8.0` – 110 MB (npm 6.13.6, yarn 1.21.1)
  - `12`, `12.15`, `12.15.0` – 79.5 MB (npm 6.13.7, yarn 1.21.1)
  - `10`, `10.19`, `10.19.0` – 74.9 MB (npm 6.13.7, yarn 1.21.1)
  - `8`, `8.17`, `8.17.0` – 67.6 MB (npm 6.13.6, yarn 1.21.1)
- Full install build with npm:
  - `6`, `6.17`, `6.17.1` – 49 MB (npm 3.10.10)
  - `4`, `4.9`, `4.9.1` – 35.2 MB (npm 2.15.12)
  - `0.12`, `0.12.18` – 32.4 MB (npm 2.15.12)
  - `0.10`, `0.10.48` – 27.8 MB (npm 2.15.12)
- Slim install with no npm or yarn:
  - `slim`, `slim-13`, `slim-13.8`, `slim-13.8.0` – 75 MB
  - `slim-12`, `slim-12.15`, `slim-12.15.0` – 45.3 MB
  - `slim-10`, `slim-10.19`, `slim-10.19.0` – 41.9 MB
  - `slim-8`, `slim-8.17`, `slim-8.17.0` – 37.2 MB
  - `slim-6`, `slim-6.17`, `slim-6.17.1` – 32.5 MB

Examples
--------

```console
$ docker run --rm mhart/alpine-node:12 node --version
v12.15.0

$ docker run --rm mhart/alpine-node:10 node --version
v10.19.0

$ docker run --rm mhart/alpine-node:12 npm --version
6.13.6

$ docker run --rm mhart/alpine-node:12 yarn --version
1.21.1

$ docker run --rm mhart/alpine-node:8 node --version
v8.17.0

$ docker run --rm mhart/alpine-node:slim-10 node --version
v10.19.0

$ docker run --rm mhart/alpine-node:slim-8 node --version
v8.17.0
```

Example Dockerfile for your own Node.js project
-----------------------------------------------

If you're doing your `npm install`/`npm ci` or `yarn install` from your
`Dockerfile`, then you'll probably want to add `node_modules` to your
`.dockerignore` file first, so that it doesn't get sent to the docker daemon.

For the smallest builds, use a multi-stage build – where you install your
modules using the full install image, but then create your app using the slim
image – this can reduce the size of your final image by ~35MB or so.

```Dockerfile
# This stage installs our modules
FROM mhart/alpine-node:12
WORKDIR /app
COPY package.json package-lock.json ./

# If you have native dependencies, you'll need extra tools
# RUN apk add --no-cache make gcc g++ python

RUN npm ci --prod

# Then we copy over the modules from above onto a `slim` image
FROM mhart/alpine-node:slim-12

# If possible, run your container using `docker run --init`
# Otherwise, you can use `tini`:
# RUN apk add --no-cache tini
# ENTRYPOINT ["/sbin/tini", "--"]

WORKDIR /app
COPY --from=0 /app .
COPY . .
CMD ["node", "index.js"]
```

If you can't do multi-stage builds, then you can just do everything on a "full
install" image:

```Dockerfile
FROM mhart/alpine-node:12

# If possible, run your container using `docker run --init`
# Otherwise, you can use `tini`:
# RUN apk add --no-cache tini
# ENTRYPOINT ["/sbin/tini", "--"]

WORKDIR /app
COPY . .

# If you have native dependencies, you'll need extra tools
# RUN apk add --no-cache make gcc g++ python

RUN npm ci --prod

CMD ["node", "index.js"]
```

Caveats
-------

As Alpine Linux uses musl, you may run into some issues with environments
expecting glibc-like behavior – especially if you try to use binaries compiled
with glibc. You should recompile these binaries to use musl (compiling on
Alpine is probably the easiest way to do this).

If you get an error similar to `error loading shared library
ld-linux-x86-64.so.2`, it may be that you have dependencies relying on libc –
you can try to fix this by adding `RUN apk add --no-cache libc6-compat` or
`RUN ln -s /lib/libc.musl-x86_64.so.1 /lib/ld-linux-x86-64.so.2` to your
`Dockerfile`.

Inspired by:

- https://github.com/alpinelinux/aports/blob/454db196/main/nodejs/APKBUILD
- https://github.com/alpinelinux/aports/blob/454db196/main/libuv/APKBUILD
- https://hub.docker.com/r/ficusio/nodejs-base/~/dockerfile/
