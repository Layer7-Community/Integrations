# Copyright © 2025 Broadcom Inc. and its subsidiaries. All Rights Reserved.
# Applicable to Layer 7 API Gateway Container release 11.1.2+

# There may have been OS packages that you rely on in pre-11.1.2 but are no longer available, this is due to the base change from ubi9-minimal to ubi9-micro
# This dockerfile provides an example on how you can add packages back to your own version of the Layer 7 Gateway image

ARG GW_IMAGE=caapim/gateway:latest

ARG MICRODIR=/tmp/micro-root
ARG MICROSRC=/tmp/source
ARG MICROTGT=/tmp/target

# The UID cannot be modified
ARG ENTRYPOINT_UID=1001

# Create a Gateway image layer and call it 'base'
FROM ${GW_IMAGE} AS base

# Use ubi9:latest image to run dnf and run rsync to find diff
FROM redhat/ubi9:latest AS installer

USER root
ARG MICRODIR
ARG MICROSRC
ARG MICROTGT

# Copy Gateway into 2 directories, SRC (source) and TGT (target). 'rsync' command uses them
# to compare and find the changed files
COPY --from=base / ${MICROSRC}
COPY --from=base / ${MICROTGT}

# Install 'rsync' utility on ubi9:latest
RUN <<EOF bash
dnf ${DNF_ARGS} install \
  --nodocs --noplugins \
  --setopt=install_weak_deps=False \
  -y \
  rsync

# A new directory to hold only the changed files/directories
mkdir ${MICRODIR}

# Run dnf to install packages. Set SRC as the installroot
# Replace <package_A> and <package_B> with the package names. Ex. passwd
dnf ${DNF_ARGS} install \
  --nodocs --noplugins \
  --installroot ${MICROSRC} \
  --setopt=install_weak_deps=False \
  -y \
  <package_A> <package_B>
dnf clean all \
  --installroot ${MICROSRC}

# Cleanup of unnecessary files
rm -rf ${MICROSRC}/var/cache/*
rm -rf ${MICROSRC}/var/log/*
rm -rf ${MICROSRC}/run/*
rm -rf ${MICROSRC}/etc/machine-id
rm -rf ${MICROSRC}/usr/share/gcc*/python

# Run rsync https://linux.die.net/man/1/rsync
# archive mode (This is equivalent to -rlptgoD), verbose, and use checksum to determine diff
rsync -avc --compare-dest=${MICROTGT} ${MICROSRC}/ ${MICRODIR}
EOF

# Add new files into the Gateway layer
FROM base
ARG MICRODIR
ARG ENTRYPOINT_UID

USER root
COPY --from=installer --chown=root:root ${MICRODIR} .
USER ${ENTRYPOINT_UID}