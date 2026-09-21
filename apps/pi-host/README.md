# pi-host

`pi-host` is the headless PI Agent Host used by PI-Desktop remote sessions. The
normal release artifact is a Linux tarball installed by the desktop over SSH.
Tag releases also publish a Linux x64 container image assembled from the same
bundle.

## Container artifacts

For release `vX.Y.Z`:

- Registry image: `ghcr.io/<repository-owner>/pi-host:X.Y.Z`
- Registry tag alias: `ghcr.io/<repository-owner>/pi-host:vX.Y.Z`
- Offline GitHub Release asset: `pi-host-X.Y.Z-linux-x64-docker.tar`

Manual release-workflow runs build the image and upload the offline tar as a
GitHub Actions artifact without pushing it. Tag runs push the two GHCR tags and
add the offline image tar to the GitHub Release.

The image uses Node 24 on Debian because `pi-host` is a Node application and the
released `host-core` binary targets glibc. It cannot use a literal `scratch`
base image. The image includes CA certificates, Git, OpenSSH client, ripgrep,
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
