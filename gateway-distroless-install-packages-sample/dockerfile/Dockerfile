# Copyright (c) 2025 Broadcom Inc. and its subsidiaries. All Rights Reserved.

ARG GW_IMAGE=caapim/gateway:latest
ARG BUILDER_IMAGE=docker.io/debian:13

# Provide Packages
################################################
FROM ${BUILDER_IMAGE} AS builder

USER root
RUN cd /tmp && \
    apt-get update && \ 
    apt-get install -y apt-rdepends && \
    # Download curl and all dependencies in its dependency tree as mentioned at https://packages.debian.org/trixie/curl 
    apt-get download curl \
                    libbrotli1 \
                    libc6 \
                    libgssapi-krb5-2 \
                    libidn2-0 \
                    libssh2-1t64 \
                    libldap2 \
                    libnghttp2-14 \
					libnghttp3-9 \
                    libpsl5t64 \
                    librtmp1 \
                    libssl3t64 \
                    libzstd1 \
					libgnutls30t64 \ 
					libhogweed6t64 libgmp10 libnettle8t64 \
					libsasl2-2 libp11-kit0 libffi8 libtasn1-6 \
                    zlib1g \
                    libunistring5 \
                    libcurl4t64 && \
    # Download jq and all dependencies in its dependency tree as mentioned at https://packages.debian.org/trixie/curl 
    apt-get download jq \
                    libc6 \
                    libonig5 \
                    libjq1 && \
    mkdir /dpkg && \
    mkdir -p /dpkg/var/lib/dpkg/status.d && \
    for deb in *.deb; do \
        package_name=$(dpkg-deb -I ${deb} | awk '/^ Package: .*$/ {print $2}'); \
        echo "Processing: ${package_name}"; \
        dpkg --ctrl-tarfile $deb | tar -Oxf - ./control > /dpkg/var/lib/dpkg/status.d/${package_name}; \
        dpkg --extract $deb /dpkg || exit 10; \
    done

FROM ${GW_IMAGE}

USER root
COPY --from=builder /dpkg /

USER 1001
WORKDIR /
