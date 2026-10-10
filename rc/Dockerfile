ARG VERSION=1.27.0-rc.0
ARG UID=200014
ARG GID=200014

ARG RUSTUP_VERSION=1.29.0
ARG NODEJS_VERSION=24
ARG OPA_VERSION=1.13.1
ARG CARGO_AUDITABLE_VERSION=0.7.2


FROM node:${NODEJS_VERSION}-alpine AS frontend

ARG VERSION

RUN apk -U upgrade \
  && apk add build-base

WORKDIR /app

RUN corepack enable

ADD https://raw.githubusercontent.com/element-hq/matrix-authentication-service/refs/tags/v${VERSION}/package.json https://raw.githubusercontent.com/element-hq/matrix-authentication-service/refs/tags/v${VERSION}/pnpm-workspace.yaml https://raw.githubusercontent.com/element-hq/matrix-authentication-service/refs/tags/v${VERSION}/pnpm-lock.yaml /app/
ADD https://raw.githubusercontent.com/element-hq/matrix-authentication-service/refs/tags/v${VERSION}/frontend/package.json /app/frontend/

RUN pnpm install --frozen-lockfile --no-runtime

ADD https://github.com/element-hq/matrix-authentication-service.git#v${VERSION}:frontend /app/frontend/
ADD https://github.com/element-hq/matrix-authentication-service.git#v${VERSION}:templates /app/templates/

RUN --network=none \
  pnpm --filter mas-frontend run build

WORKDIR /app/frontend

RUN --network=none \
  mkdir -p /share/assets \
  && cp ./dist/manifest.json /share/manifest.json \
  && rm -f ./dist/index.html* ./dist/manifest.json* \
  && cp ./dist/* /share/assets/

# ======================================= #

FROM alpine AS policy

ARG VERSION
ARG BUILDARCH
ARG OPA_VERSION

RUN apk -U upgrade \
  && apk add build-base tar

# Download Open Policy Agent
ADD --chmod=755 https://github.com/open-policy-agent/opa/releases/download/v${OPA_VERSION}/opa_linux_${BUILDARCH}_static /usr/local/bin/opa

WORKDIR /app/policies
ADD https://github.com/element-hq/matrix-authentication-service.git#v${VERSION}:policies /app/policies

RUN --network=none make -B \
  && chmod a+r ./policy.wasm

# ======================================= #

FROM alpine AS builder

ARG VERSION
ARG BUILDARCH
ARG RUSTUP_VERSION
ARG CARGO_AUDITABLE_VERSION

ENV RUSTUP_HOME=/usr/local/rustup
ENV CARGO_HOME=/usr/local/cargo
ENV PATH=/usr/local/cargo/bin:$PATH
ENV VERGEN_GIT_DESCRIBE=${VERSION}

RUN apk -U upgrade \
  && apk add build-base curl

RUN case "${BUILDARCH}" in \
    amd64) RUSTUP_TRIPLE="x86_64-unknown-linux-musl" ;; \
    arm64) RUSTUP_TRIPLE="aarch64-unknown-linux-musl" ;; \
    *) echo "unsupported architecture: ${BUILDARCH}" >&2; exit 1 ;; \
  esac \
  && curl -fsSLo rustup-init "https://static.rust-lang.org/rustup/archive/${RUSTUP_VERSION}/${RUSTUP_TRIPLE}/rustup-init" \
  && chmod +x rustup-init \
  && ./rustup-init -y --no-modify-path --default-toolchain none \
  && rm rustup-init

WORKDIR /app
ADD https://raw.githubusercontent.com/element-hq/matrix-authentication-service/refs/tags/v${VERSION}/rust-toolchain.toml ./
RUN rustup toolchain install

RUN cargo install --locked \
  cargo-auditable@=${CARGO_AUDITABLE_VERSION}

# Copy the code
ADD https://github.com/element-hq/matrix-authentication-service.git#v${VERSION} /app
ENV SQLX_OFFLINE=true

# Network access: cargo auditable needs it
RUN cargo auditable build \
  --locked \
  --release \
  --bin mas-cli \
  --no-default-features \
  --features docker

RUN --network=none \
  mkdir -p /install \
  && cp "target/release/mas-cli" /install

# ======================================= #

FROM alpine

LABEL maintainer="Thien Tran contact@tommytran.io"

ARG VERSION
ARG UID
ARG GID

RUN apk -U upgrade \
  && apk add libstdc++

RUN --network=none \
  addgroup -g ${GID} mas \
  && adduser -u ${UID} --ingroup mas --disabled-password --system mas

COPY --from=builder /install /usr/local/bin/
COPY --from=frontend /share /usr/local/share/mas-cli
COPY --from=policy /app/policies/policy.wasm /usr/local/share/mas-cli/policy.wasm
ADD https://github.com/element-hq/matrix-authentication-service.git#v${VERSION}:templates /usr/local/share/mas-cli/templates
ADD https://github.com/element-hq/matrix-authentication-service.git#v${VERSION}:translations /usr/local/share/mas-cli/translations

COPY --from=ghcr.io/polarix-containers/hardened_malloc:latest /install /usr/local/lib/
ENV LD_PRELOAD="/usr/local/lib/libhardened_malloc.so"

USER mas

ENTRYPOINT ["/usr/local/bin/mas-cli"]