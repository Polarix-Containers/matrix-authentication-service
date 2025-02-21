ARG VERSION=0.14.1
ARG UID=200014
ARG GID=200014

ARG RUSTC_VERSION=1.85.0
ARG NODEJS_VERSION=20
ARG OPA_VERSION=1.1.0
ARG CARGO_AUDITABLE_VERSION=0.6.6


FROM node:${NODEJS_VERSION}-alpine AS frontend

ARG VERSION

RUN apk -U upgrade \
  && apk add build-base

WORKDIR /app/frontend

ADD https://raw.githubusercontent.com/element-hq/matrix-authentication-service/refs/tags/v${VERSION}/frontend/package.json https://raw.githubusercontent.com/element-hq/matrix-authentication-service/refs/tags/v${VERSION}/frontend/package-lock.json /app/frontend/

RUN npm ci

ADD https://github.com/element-hq/matrix-authentication-service.git#v${VERSION}:frontend /app/frontend/
ADD https://github.com/element-hq/matrix-authentication-service.git#v${VERSION}:templates /app/templates/

RUN --network=none \
  npm run build \
  && mkdir -p /share/assets \
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

FROM rust:${RUSTC_VERSION}-alpine AS builder

ARG VERSION
ARG CARGO_AUDITABLE_VERSION
ARG RUSTC_VERSION

RUN apk -U upgrade \
  && apk add build-base

RUN cargo install --locked \
  cargo-auditable@=${CARGO_AUDITABLE_VERSION}

# Set the working directory
WORKDIR /app

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
COPY --from=frontend /share /share
COPY --from=policy /app/policies/policy.wasm /share/policy.wasm
ADD https://github.com/element-hq/matrix-authentication-service.git#v${VERSION}:templates /share/templates
ADD https://github.com/element-hq/matrix-authentication-service.git#v${VERSION}:translations /share/translations

COPY --from=ghcr.io/polarix-containers/hardened_malloc:latest /install /usr/local/lib/
ENV LD_PRELOAD="/usr/local/lib/libhardened_malloc.so"

USER mas

ENTRYPOINT ["/usr/local/bin/mas-cli"]