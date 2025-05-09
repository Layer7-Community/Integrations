# Copyright © 2025 Broadcom Inc. and its subsidiaries. All Rights Reserved.
# Applicable to Layer 7 API Gateway Container release 11.1.2+

# There may have been OS packages that you rely on in pre-11.1.2 but are no longer available, this is due to the base change from ubi9-minimal to ubi9-micro
# This dockerfile provides an example on how you can add packages back to your own version of the Layer 7 Gateway image

# Temporary location path to put the existing root directory of the official Layer 7 API Gateway image
ARG MICRODIR=/tmp/gateway-root

# Entrypoint user ID, this value has to be 1001
ARG ENTRYPOINT_UID=1001

# Layer 7 API Gateway image to build on
ARG GATEWAY_IMAGE=caapim/gateway:latest

# UBI9 image from redhat. This image will provide the dnf command utility to run package installation
ARG UBI9=redhat/ubi9:latest

# Establish the official Layer 7 API Gateway image as "base"
FROM ${GATEWAY_IMAGE} AS base

# Establish the redhat/ubi9:latest image as "utility-provider"
FROM ${UBI9} AS utility-provider
USER root

# Make the temporary directory inside the redhat UBI9 image
ARG MICRODIR
RUN mkdir ${MICRODIR}

# Copy the Layer 7 API Gateway image's (base) root directory into the created directory on the utility-provider
COPY --from=base / ${MICRODIR}

# Set dnf argument
ENV DNF_ARGS=--releasever=9.5\ --disableplugin=subscription-manager\ --disablerepo=*\ --enablerepo=ubi-9-appstream-rpms\ --enablerepo=ubi-9-baseos-rpms

# Run dnf command ( from redhat ubi9) to install packageNameA and packageNameB.
# Change the root context to MICRODIR for dnf to use as a reference to figure out what dependencies are needed
# Replace packageNameA and packageNameB with the package you want to install
RUN <<EOF bash

dnf ${DNF_ARGS} install \
--nodocs --noplugins \
--installroot ${MICRODIR} \
--setopt=install_weak_deps=False \
-y \
packageNameA packageNameB
dnf clean all \
--installroot ${MICRODIR}
EOF


# Override the Layer 7 API Gateway image's root with the one in utility-provider
FROM ${GATEWAY_IMAGE}

USER root

ARG ENTRYPOINT_UID
ARG MICRODIR

# Replace Layer 7 API Gateway's root directory with the content of MICRODIR
COPY --from=utility-provider --chown=root:root ${MICRODIR} .
USER ${ENTRYPOINT_UID}