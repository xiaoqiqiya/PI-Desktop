FROM node:24-bookworm-slim

ARG PI_HOST_VERSION=dev
ARG PI_HOST_SOURCE=https://github.com/vastsa/PI-Desktop

LABEL org.opencontainers.image.title="pi-host" \
      org.opencontainers.image.description="Headless PI Agent Host" \
      org.opencontainers.image.version="${PI_HOST_VERSION}" \
      org.opencontainers.image.source="${PI_HOST_SOURCE}"

RUN apt-get update \
  && apt-get install --no-install-recommends -y ca-certificates git openssh-client ripgrep tini \
  && rm -rf /var/lib/apt/lists/*

WORKDIR /opt/pi-host
COPY --chown=node:node . .

RUN mkdir -p /data /workspace \
  && chown node:node /data /workspace

ENV NODE_ENV=production \
    PI_DESKTOP_DATA_DIR=/data \
    PI_HOST_BROWSE_ROOT=/workspace \
    PI_HOST_PORT=4123

USER node
EXPOSE 4123

ENTRYPOINT ["/usr/bin/tini", "--", "node", "/opt/pi-host/pi-host.js"]
CMD ["--host", "127.0.0.1", "--port", "4123", "--data-dir", "/data", "--browse-root", "/workspace"]
