# Argo CD v3.5.1
FROM quay.io/argoproj/argocd@sha256:0deb1a1c917629b960ead995ae3b6069450a866992676599658687ef9a641ee8 AS argocd

# Final Image
FROM docker.io/library/ubuntu:26.04

USER root

ENV ARGOCD_USER_ID=999
ENV DEBIAN_FRONTEND=noninteractive

RUN groupadd -g $ARGOCD_USER_ID argocd && \
    useradd -r -u $ARGOCD_USER_ID -g argocd argocd && \
    mkdir -p /home/argocd && \
    chown argocd:0 /home/argocd && \
    chmod g=u /home/argocd && \
    apt-get update && \
    apt-get dist-upgrade -y && \
    apt-get install -y \
    git git-lfs tini curl python3-pip pipx gpg tzdata unzip && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Create a symlink for python to point to python3
RUN ln -s /usr/bin/python3 /usr/bin/python

# Install AWS CLI v2 (arch-aware). AWS CLI v2 only publishes x86_64 and aarch64
# binaries, so the util image targets linux/amd64 and linux/arm64.
ARG TARGETARCH
RUN AWS_ARCH=$( [ "$TARGETARCH" = "arm64" ] && echo aarch64 || echo x86_64 ); \
    curl "https://awscli.amazonaws.com/awscli-exe-linux-${AWS_ARCH}.zip" -o "awscliv2.zip" && \
    unzip awscliv2.zip && \
    ./aws/install && \
    rm -rf awscliv2.zip aws

# Install the Microsoft Azure CLI globally with pipx (Microsoft's documented
# fallback). pipx --global installs the venv under /opt/pipx and symlinks the
# `az` launcher into /usr/local/bin so all users (incl. the non-root `argocd`
# user) can run it, without touching the system Python's own packages.
RUN pipx install --global azure-cli

# Install the Google Cloud SDK (arch-aware). gcloud publishes separate
# x86_64 and Arm (aarch64) archives; arm64 relies on the system Python
# (3.10-3.14), which the base image provides.
ARG TARGETARCH
ARG GCLOUD_VERSION=582.0.0
RUN GCLOUD_ARCH=$( [ "$TARGETARCH" = "amd64" ] && echo x86_64 || echo arm ); \
    curl -sL "https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-cli-${GCLOUD_VERSION}-linux-${GCLOUD_ARCH}.tar.gz" -o /tmp/gcloud.tar.gz && \
    tar xzf /tmp/gcloud.tar.gz -C /home/argocd && \
    /home/argocd/google-cloud-sdk/install.sh --quiet --disable-installation-options --no-compile-python && \
    rm -f /tmp/gcloud.tar.gz

ENV PATH="/home/argocd/google-cloud-sdk/bin:${PATH}"

COPY entrypoint.sh /usr/local/bin/uid_entrypoint.sh
RUN chmod +x /usr/local/bin/uid_entrypoint.sh

# Install Argo CD CLI from argocd
COPY --from=argocd /usr/local/bin/argocd /usr/local/bin/argocd

# Copy util wrapper script
COPY util.sh /usr/local/bin/argocd-operator-util

ENV USER_NAME=argocd
ENV HOME=/home/argocd

USER argocd
WORKDIR /home/argocd
