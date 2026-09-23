# pi-host

`pi-host` is the headless PI Agent Host used by PI-Desktop remote sessions. The
normal release artifact is a Linux tarball installed by the desktop over SSH.
The independent `.github/workflows/pi-host-release.yml` workflow builds the
Linux x64 bundle, packages and smoke-tests its container, publishes GHCR tags,
and uploads offline artifacts without running the Desktop installer matrix.

## Container artifacts

For package version `X.Y.Z`:

- Versioned registry image (dedicated Tag only): `ghcr.io/<repository-owner>/pi-host:X.Y.Z`
- Main alias: `ghcr.io/<repository-owner>/pi-host:main`
- Commit alias: `ghcr.io/<repository-owner>/pi-host:sha-<commit>`
- Dedicated Tag alias: `ghcr.io/<repository-owner>/pi-host:vX.Y.Z`
- Offline Actions artifact: `pi-host-X.Y.Z-linux-x64-docker.tar`
- Dedicated Tag release assets: the Docker tar, native bundle tarball, and SHA-256

A relevant push to `main`, or **Pi Host Docker Release** run manually from
`main`, refreshes `:main` and publishes `:sha-<commit>`. Pushing `pi-host-vX.Y.Z` or a numeric revision such as `pi-host-vX.Y.Z.1`
publishes matching `:X.Y.Z[.revision]` and `:vX.Y.Z[.revision]` images and
creates a pi-host-only GitHub Release. The runtime compatibility version inside
the bundle remains `X.Y.Z`.

The image uses Node 24 on Debian because `pi-host` is a Node application. Its
Rust `host-core` binary is built for `x86_64-unknown-linux-musl` and verified to
have no dynamic ELF interpreter or shared-library dependencies, so the native
binary does not require the target machine's glibc. The full image still cannot
use a literal `scratch` base because the Node runtime and Agent tools need a
userspace. The image includes CA certificates, Git, OpenSSH client, ripgrep,
and `tini`; project-specific tools still need to be installed by a derived
image or mounted from the host.

## Run on Linux

`pi-host` intentionally binds loopback only. Docker bridge networking changes
the peer address, so use Linux host networking to preserve that security
boundary:

```bash
docker pull ghcr.io/<repository-owner>/pi-host:X.Y.Z

docker run -d \
  --name pi-host \
  --restart unless-stopped \
  --network host \
  -v pi-host-data:/data \
  -v /absolute/path/to/projects:/workspace \
  ghcr.io/<repository-owner>/pi-host:X.Y.Z
```

The defaults are:

- RACP listener: `127.0.0.1:4123`
- Durable data: `/data`
- Browsable project root: `/workspace`

The port is not published with `-p`; reach it locally or through an SSH tunnel:

```bash
ssh -N -L 4123:127.0.0.1:4123 user@remote-host
```

Then connect to `ws://127.0.0.1:4123/v1/racp/ws` with a valid device token.

Bind-mounted project directories must be readable and writable by the
container's `node` user (UID/GID 1000 by default), or be mounted read-only when
the Host should only inspect them.
The current PI-Desktop SSH bootstrap continues to install the release tarball;
it does not automatically deploy this Docker image.

## Start with a pairing token

To print a one-time pairing token, run the container in the foreground once:

```bash
docker run --rm \
  --network host \
  -v pi-host-data:/data \
  -v /absolute/path/to/projects:/workspace \
  ghcr.io/<repository-owner>/pi-host:X.Y.Z \
  --host 127.0.0.1 \
  --port 4123 \
  --data-dir /data \
  --browse-root /workspace \
  --pair
```

Keep `/data` persistent. It contains the Host identity, paired-device records,
SQLite data, logs, provider configuration, and other Host-local state. Mount
only the project directories the Host is allowed to access under `/workspace`.

## Load the offline image

Download `pi-host-X.Y.Z-linux-x64-docker.tar` from the matching GitHub Release,
then run:

```bash
docker load -i pi-host-X.Y.Z-linux-x64-docker.tar
docker image ls | grep pi-host
```

Use the loaded version tag with the same `docker run` command above.

## Build a derived image

Install extra command-line tools without changing the release image:

```dockerfile
FROM ghcr.io/<repository-owner>/pi-host:X.Y.Z
USER root
RUN apt-get update \
  && apt-get install --no-install-recommends -y python3 make \
  && rm -rf /var/lib/apt/lists/*
USER node
```
