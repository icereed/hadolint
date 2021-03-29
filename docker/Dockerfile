# Image shared across all stages
FROM ubuntu:focal as common-base
RUN apt-get update && apt-get install libtinfo5 libnuma-dev libgmp-dev -y
RUN mkdir -p /app
WORKDIR /app
ENV GHC_VERSION=8.10.4

# Build-time dependencies
FROM common-base as build-dependencies
RUN apt-get install build-essential llvm-9 git wget xz-utils zlib1g-dev -y

# Stage 1: GHC
FROM build-dependencies as ghc-builder-source
RUN case "$(uname -m)" in \
  x86_64|amd64) \
  LONG_ARCH="x86_64" \
  ;; \
  aarch64|arm64) \
  LONG_ARCH="aarch64" \
  ;; \
  esac && \
  wget -q https://downloads.haskell.org/\~ghc/${GHC_VERSION}/ghc-${GHC_VERSION}-${LONG_ARCH}-deb10-linux.tar.xz && \
  unxz ghc-${GHC_VERSION}-${LONG_ARCH}-deb10-linux.tar.xz && \
  tar xf ghc-${GHC_VERSION}-${LONG_ARCH}-deb10-linux.tar && \
  rm ghc-${GHC_VERSION}-${LONG_ARCH}-deb10-linux.tar

FROM ghc-builder-source as ghc-builder
RUN cd ghc-${GHC_VERSION} && \
  ./configure && \
  make install && \
  cd .. && \
  rm -rf ghc-${GHC_VERSION}

# Stage 2: Cabal
FROM ghc-builder as cabal-builder-source
RUN git clone https://github.com/haskell/cabal.git

FROM cabal-builder-source as cabal-bootstrap-builder
RUN cd /app/cabal && find ./bootstrap -type f -name "*.json" && \
  ./bootstrap/bootstrap.py -d ./bootstrap/linux-${GHC_VERSION}.json -w /usr/local/bin/ghc

FROM cabal-bootstrap-builder as cabal-builder
RUN cd /app/cabal && \
  ./_build/bin/cabal update && \
  ./_build/bin/cabal v2-build cabal-install && \
  ./_build/bin/cabal install cabal-install && \
  cd /app && \
  rm -rf cabal && \
  echo 'PATH=$PATH:$HOME/.cabal/bin; export PATH' >>$HOME/.profile

# Stage 3: Build hadolint
FROM cabal-builder AS builder

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

RUN apt-get update \
  && apt-get install --no-install-recommends -y \
  build-essential=12.3 \
  libffi-dev=3.2.* \
  libgmp-dev=2:6.1.* \
  zlib1g-dev=1:1.2.* \
  curl=7.52.* \
  ca-certificates=* \
  git=1:2.11.* \
  netbase=5.4 \
  && rm -rf /var/lib/apt/lists/*

WORKDIR /opt/hadolint/
COPY stack.yaml package.yaml /opt/hadolint/
RUN stack --no-terminal --install-ghc test --only-dependencies

COPY . /opt/hadolint
RUN scripts/fetch_version.sh \
  && stack install --ghc-options="-fPIC" --flag hadolint:static

# COMPRESS WITH UPX
RUN case "$(uname -m)" in \
  x86_64|amd64) \
  LONG_ARCH="x86_64" \
  SHORT_ARCH="amd64" \
  ;; \
  aarch64|arm64) \
  LONG_ARCH="aarch64" \
  SHORT_ARCH="arm64" \
  ;; \
  esac && curl -sSL https://github.com/upx/upx/releases/download/v3.96/upx-3.96-${SHORT_ARCH}_linux.tar.xz \
  | tar -x --xz --strip-components 1 upx-3.96-${SHORT_ARCH}_linux/upx \
  && ./upx --best --ultra-brute /root/.local/bin/hadolint

# Stage 4: Create for distros
FROM debian:stretch-slim AS debian-distro
COPY --from=builder /root/.local/bin/hadolint /bin/
CMD ["/bin/hadolint", "-"]

FROM alpine:3 AS alpine-distro
COPY --from=builder /root/.local/bin/hadolint /bin/
CMD ["/bin/hadolint", "-"]

FROM scratch
COPY --from=builder /root/.local/bin/hadolint /bin/
CMD ["/bin/hadolint", "-"]
