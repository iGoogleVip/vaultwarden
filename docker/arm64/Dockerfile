# syntax=docker/dockerfile:1

# This file was generated using a Jinja2 template.
# Please make your changes in `Dockerfile.j2` and then `make` the individual Dockerfiles.
# Using multistage build:
# 	https://docs.docker.com/develop/develop-images/multistage-build/
# 	https://whitfin.io/speeding-up-rust-docker-builds/
####################### VAULT BUILD IMAGE  #######################
# The web-vault digest specifies a particular web-vault build on Docker Hub.
# Using the digest instead of the tag name provides better security,
# as the digest of an image is immutable, whereas a tag name can later
# be changed to point to a malicious image.
#
# To verify the current digest for a given tag name:
# - From https://hub.docker.com/r/vaultwarden/web-vault/tags,
#   click the tag name to view the digest of the image it currently points to.
# - From the command line:
#     $ docker pull docker.io/vaultwarden/web-vault:v2023.8.2
#     $ docker image inspect --format "{{.RepoDigests}}" docker.io/vaultwarden/web-vault:v2023.8.2
#     [docker.io/vaultwarden/web-vault@sha256:b361e79309ef2c4368f880f350166daade41eb0927a9adf376c76e3713027252]
#
# - Conversely, to get the tag name from the digest:
#     $ docker image inspect --format "{{.RepoTags}}" docker.io/vaultwarden/web-vault@sha256:b361e79309ef2c4368f880f350166daade41eb0927a9adf376c76e3713027252
#     [docker.io/vaultwarden/web-vault:v2023.8.2]
#
FROM docker.io/vaultwarden/web-vault@sha256:b361e79309ef2c4368f880f350166daade41eb0927a9adf376c76e3713027252 as vault

########################## BUILD IMAGE  ##########################
FROM docker.io/library/rust:1.72.0-bookworm as build

# Build time options to avoid dpkg warnings and help with reproducible builds.
ENV DEBIAN_FRONTEND=noninteractive \
    LANG=C.UTF-8 \
    TZ=UTC \
    TERM=xterm-256color \
    CARGO_HOME="/root/.cargo" \
    REGISTRIES_CRATES_IO_PROTOCOL=sparse \
    USER="root"

# Create CARGO_HOME folder and don't download rust docs
RUN mkdir -pv "${CARGO_HOME}" \
    && rustup set profile minimal

# Install build dependencies for the arm64 architecture
RUN dpkg --add-architecture arm64 \
    && apt-get update \
    && apt-get install -y \
        --no-install-recommends \
        gcc-aarch64-linux-gnu \
        libc6-dev:arm64 \
        libmariadb-dev:arm64 \
        libmariadb-dev-compat:arm64 \
        libmariadb3:arm64 \
        libpq-dev:arm64 \
        libpq5:arm64 \
        libssl-dev:arm64 \
    #
    # Make sure cargo has the right target config
    && echo '[target.aarch64-unknown-linux-gnu]' >> "${CARGO_HOME}/config" \
    && echo 'linker = "aarch64-linux-gnu-gcc"' >> "${CARGO_HOME}/config" \
    && echo 'rustflags = ["-L/usr/lib/aarch64-linux-gnu"]' >> "${CARGO_HOME}/config"

# Set arm specific environment values
ENV CC_aarch64_unknown_linux_gnu="/usr/bin/aarch64-linux-gnu-gcc" \
    CROSS_COMPILE="1" \
    OPENSSL_INCLUDE_DIR="/usr/include/aarch64-linux-gnu" \
    OPENSSL_LIB_DIR="/usr/lib/aarch64-linux-gnu"

# Creates a dummy project used to grab dependencies
RUN USER=root cargo new --bin /app
WORKDIR /app

# Copies over *only* your manifests and build files
COPY ./Cargo.* ./
COPY ./rust-toolchain.toml ./rust-toolchain.toml
COPY ./build.rs ./build.rs

RUN rustup target add aarch64-unknown-linux-gnu

# Configure the DB ARG as late as possible to not invalidate the cached layers above
ARG DB=sqlite,mysql,postgresql

# Builds your dependencies and removes the
# dummy project, except the target folder
# This folder contains the compiled dependencies
RUN cargo build --features ${DB} --release --target=aarch64-unknown-linux-gnu \
    && find . -not -path "./target*" -delete

# Copies the complete project
# To avoid copying unneeded files, use .dockerignore
COPY . .

# Make sure that we actually build the project
RUN touch src/main.rs

# Builds again, this time it'll just be
# your actual source files being built
RUN cargo build --features ${DB} --release --target=aarch64-unknown-linux-gnu

######################## RUNTIME IMAGE  ########################
# Create a new stage with a minimal image
# because we already have a binary built
FROM docker.io/balenalib/aarch64-debian:bookworm

ENV ROCKET_PROFILE="release" \
    ROCKET_ADDRESS=0.0.0.0 \
    ROCKET_PORT=80

RUN [ "cross-build-start" ]

# Create data folder and Install needed libraries
RUN mkdir /data \
    && apt-get update && apt-get install -y \
    --no-install-recommends \
    ca-certificates \
    curl \
    libmariadb-dev-compat \
    libpq5 \
    openssl \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

RUN [ "cross-build-end" ]

VOLUME /data
EXPOSE 80
EXPOSE 3012

# Copies the files from the context (Rocket.toml file and web-vault)
# and the binary from the "build" stage to the current stage
WORKDIR /
COPY --from=vault /web-vault ./web-vault
COPY --from=build /app/target/aarch64-unknown-linux-gnu/release/vaultwarden .

COPY docker/healthcheck.sh /healthcheck.sh
COPY docker/start.sh /start.sh

HEALTHCHECK --interval=60s --timeout=10s CMD ["/healthcheck.sh"]

CMD ["/start.sh"]
