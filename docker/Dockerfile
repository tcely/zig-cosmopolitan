# syntax=docker/dockerfile:1
# check=error=true

# Global ARG for the base image tags
ARG DEBIAN_VERSION=13

# Stage 1: Extractor - Download and Verify Assets
FROM debian:${DEBIAN_VERSION}-slim AS extractor

RUN apt-get update && apt-get install -y \
    curl \
    jq \
    gnupg2 \
    libarchive-tools \
    pixz \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /build

RUN lines() { printf -- '%s\n' "$@"; } && \
    { lines \
    'extract() {' \
    '    local file; file="$1"; shift;' \
    '    bsdtar -xf "${file}" "$@" ;' \
    "}" \
    "err() { printf 1>&2 -- '%s\n' \"\$@\"; }" \
    "fatal() { err \"\$1\"; exit 1; }"; \
    } > /build/common.sh

# Record GPG Key Metadata (Stable identity)
ENV COSMO_GPG_PUBLIC_ID="0x65A68CC8F87C5311"
ENV COSMO_GPG_FINGERPRINT="9924 9E69 A705 668F 9983  9365 65A6 8CC8 F87C 5311"
ARG HOST_GH_WEB="github.com"
ARG HOST_COSMO_AUTHOR="justine.lol"
ADD "https://${HOST_GH_WEB}/jart.gpg" /build/author.asc

# Cosmopolitan Toolchain Arguments
ARG HOST_COSMO_DL="cosmo.zip"
ARG COSMO_VER="4.0.2"
ARG COSMO_SHA="85b8c37a406d862e656ad4ec14be9f6ce474c1b436b9615e91a55208aced3f44"
ADD "https://${HOST_GH_WEB}/jart/cosmopolitan/releases/download/${COSMO_VER}/cosmocc-${COSMO_VER}.zip" /build/cosmocc.zip

# Verify Toolchain Digest and Signature
RUN . /build/common.sh && \
    cd /build || fatal "Failed to enter /build" && \
    printf '%s *cosmocc.zip\n' "${COSMO_SHA}" > cosmocc.zip.sha256 && \
    sha256sum --check cosmocc.zip.sha256 || fatal "Cosmo SHA256 mismatch" && \
    SIG_URL="https://${HOST_COSMO_DL}/pub/cosmocc/cosmocc-${COSMO_VER}.zip.sig" && \
    if curl -o /build/cosmocc.zip.sig -sSfL "${SIG_URL}"; then \
        gpg2 --import author.asc || fatal "Failed to import GPG key" && \
        ACTUAL_FPR=$(gpg2 --with-colons --with-fingerprint --list-keys "${COSMO_GPG_PUBLIC_ID}" | awk -F: '/^fpr/ {print $10}') && \
        printf '%s' "${ACTUAL_FPR}" | grep -iq "$(printf '%s' "${COSMO_GPG_FINGERPRINT}" | tr -d ' ')" || fatal "Fingerprint mismatch" && \
        printf '%s:6:\n' "${ACTUAL_FPR}" | gpg2 --import-ownertrust && \
        gpg2 --verify cosmocc.zip.sig cosmocc.zip || fatal "GPG signature verification failed"; \
    fi && \
    mkdir -p /opt/cosmocc && \
    extract cosmocc.zip -C /opt/cosmocc

# Zig Metadata, Source, and Bootstrap Extraction
ARG HOST_ZIG_DL="ziglang.org"
ADD "https://${HOST_ZIG_DL}/download/index.json" /build/zig_index.json

RUN . /build/common.sh && \
    { \
        jq -c 'to_entries | map(select(.key != "master")) | sort_by(.key | split(".") | map(tonumber)) | last | .value' \
        /build/zig_index.json; \
    } >/build/zig_metadata.json || fatal "Failed to extract metadata" && \
    # 1. Process Source
    SRC_URL=$(jq -r '.src.tarball' /build/zig_metadata.json) && \
    SRC_SHA=$(jq -r '.src.shasum' /build/zig_metadata.json) && \
    curl -o zig-src.tar.xz -L "${SRC_URL}" || fatal "Failed to download Zig source" && \
    printf '%s *zig-src.tar.xz\n' "${SRC_SHA}" > zig-src.tar.xz.sha256 && \
    sha256sum --check zig-src.tar.xz.sha256 || fatal "Zig source SHA256 mismatch" && \
    mkdir /build/zig && \
    pixz -d zig-src.tar.xz && \
    extract zig-src.tar -C /build/zig --strip-components=1 && \
    rm zig-src.tar.xz.sha256 && \
    # 2. Process Bootstrap
    BOOT_URL=$(jq -r '.bootstrap.tarball' /build/zig_metadata.json) && \
    BOOT_SHA=$(jq -r '.bootstrap.shasum' /build/zig_metadata.json) && \
    curl -o zig-boot.tar.xz -L "${BOOT_URL}" || fatal "Failed to download Zig bootstrap" && \
    printf '%s *zig-boot.tar.xz\n' "${BOOT_SHA}" > zig-boot.tar.xz.sha256 && \
    sha256sum --check zig-boot.tar.xz.sha256 || fatal "Zig bootstrap SHA256 mismatch" && \
    pixz -d zig-boot.tar.xz && \
    extract zig-boot.tar -C /build/zig --strip-components=1 && \
    rm zig-boot.tar.xz.sha256

# Stage 2: Builder - Bootstrap and Verify
FROM debian:${DEBIAN_VERSION}-slim AS builder_base
RUN apt-get update && apt-get install -y file && rm -rf /var/lib/apt/lists/*
COPY --from=extractor /opt/cosmocc /opt/cosmocc
COPY --from=extractor /build/zig /build/zig
ENV PATH="/opt/cosmocc/bin:${PATH}"
WORKDIR /build/zig

RUN ls -ARhl stage1/ src/main.zig


# Bootstrap sequence
FROM builder_base AS builder_stage1
RUN cosmocc -o zig-wasm2c.com stage1/wasm2c.c -O2 -std=c99 && \
    ./zig-wasm2c.com stage1/zig1.wasm zig1.c && \
    cosmocc -o zig1.com zig1.c stage1/wasi.c -std=c99 -Os -lm -Wl,-z,stack-size=0x10000000

FROM builder_stage1 AS builder
RUN cat -v zig1.c ; ls -hl zig1.com src/main.zig ; \
    ./zig1.com -ofmt=c -lc -OReleaseSmall \
    -target x86_64-linux-musl --mod root:src/main.zig --deps root && \
    cosmocc -o zig2.com zig2.c -O2 -lm -Wl,-z,stack-size=0x10000000

# Validation
RUN printf '#include <stdio.h>\nint main() { printf("Hello Static Musl\\n"); return 0; }\n' > hello.c && \
    ./zig2.com build-exe hello.c -target x86_64-linux-musl -lc --name hello_static && \
    ./hello_static | grep -q "Hello Static Musl" && \
    file hello_static | grep -q "statically linked"

# Stage 3: Export Binary Target
FROM scratch AS export_binary
COPY --from=builder /build/zig/zig-wasm2c.com /zig-wasm2c
COPY --from=builder /build/zig/zig2.com /zig.com

# Stage 4: Final Distroless
FROM gcr.io/distroless/cc-debian${DEBIAN_VERSION}:debug
USER nonroot
WORKDIR /app
COPY --from=builder /build/zig/zig2.com /app/zig
ENTRYPOINT ["/app/zig"]
CMD ["version"]
