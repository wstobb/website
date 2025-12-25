---
layout: post
title: 'Custom Fedora Atomic Image'
date: 2025-12-24
categories: [custom-linux]
tags: [fedora-linux, advanced, linux]
---

## Preface

My girlfriend recently showed an interest in using Linux, and asked me for help setting it up. Since I know how finicky Linux can be and how important the usability of her computer is for her, I
decided to set up a custom Fedora Atomic Image for her that contains everything she needs.

This is a guide on how to create your own image for your own computer, but you can find the custom image I made for her [here](https://github.com/wstobb/GFOS). There is also a template available
[here](https://github.com/wstobb/fedora-custom-bootc-template).

## Goals

- Easy to make modifications
- Automated deployment and updates
- 'Git centric'

## Prerequisites

- Github account
- Git repository
- Basic knowledge of OCI containers and bash scripting

## Getting Started

### Directory structure

Create a directory structure populated with files that resembles this:

```
.
├── .github/
│   └── workflows/
├── config/
│   ├── disable_services.list
│   ├── enable_services.list
│   ├── install_packages.list
│   └── remove_packages.list
├── scripts/
│   ├── 00-build.sh
│   ├── 01-repos.sh
│   ├── 02-packages.sh
│   └── 03-services.sh
├── sys_root/
│   └── ...
└── Containerfile
```

### Containerfile

Edit the containerfile with these lines or something similar.

```
FROM scratch as ctx
COPY / /
FROM quay.io/fedora-ostree-desktops/silverblue:42
RUN --mount=type=bind,from=ctx,source=/,target=/ctx \
    /ctx/scripts/00-build.sh \
    ostree container commit
```

#### Explanation

- The first 'FROM' section creates a workspace for the code that is used to modify the container.
- The 'COPY' takes the code and puts it in the workspace.
- The second 'FROM' fetches the most recent fedora bootc image.
- The 'RUN' section runs the main build script and commits the changes to the image.

### Scripts

You now must create the scripts that loop through the configuration files. Here are some examples of what I put together.

#### Build

```bash
#!/bin/bash

set -ouex pipefail

# Copy system root to container
cp -rv /ctx/sys_root/* /

/ctx/scripts/01-repos.sh
/ctx/scripts/02-packages.sh
/ctx/scripts/03-services.sh
```

#### Repos

```bash
#!/bin/bash

set -ouex pipefail

dnf5 install -y 'dnf5-command(copr)'
```

#### Packages

```bash
#!/bin/bash

set -ouex pipefail

if [ -s "/ctx/config/install_packages.list" ]; then
	dnf install -y $(tr '\n' ' ' < /ctx/config/install_packages.list)
fi

if [ -s "/ctx/config/remove_packages.list" ]; then
	dnf remove -y $(tr '\n' ' ' < /ctx/config/remove_packages.list)
fi
```

#### Services

```bash
#!/bin/bash

set -ouex pipefail

if [ -s "/ctx/config/enable_services.list" ]; then
	systemctl enable $(tr '\n' ' ' < /ctx/config/enable_services.list)
fi

if [ -s "/ctx/config/disable_services" ]; then
	systemctl disable $(tr '\n' ' ' < /ctx/config/disable_services.list)
fi
```

#### Note

The line 'set -ouex pipefail' is very important as it stops the Github action from getting stuck if something goes wrong.

### System root

The folder 'sys_root' is where you put the files that you want copied from the repo into the images. An example of how to use this is system configuration or adding extra repository files to dnf.

### The Github Action

You must create a Github action that creates and publishes the image. Here is an example of how I did it:

```yml
---
name: Build bootc image
on:
  pull_request:
    branches:
      - main
    paths-ignore:
      - '**/README.md'
  schedule:
    - cron: '0 0 * * sun'
  push:
    branches:
      - main
    paths-ignore:
      - '**/README.md'
  workflow_dispatch:
env:
  IMAGE_DESC: 'This is an image template'
  IMAGE_NAME: '${{ github.event.repository.name }}'
  IMAGE_REGISTRY: 'ghcr.io/${{ github.repository_owner }}'
  DEFAULT_TAG: 'latest'
jobs:
  build_push:
    name: Build and push image
    runs-on: ubuntu-24.04
    permissions:
      contents: read
      packages: write
      id-token: write
    steps:
      - name: Prepare environment
        run: |
          echo "IMAGE_REGISTRY=${IMAGE_REGISTRY,,}" >> ${GITHUB_ENV}
          echo "IMAGE_NAME=${IMAGE_NAME,,}" >> ${GITHUB_ENV}
          echo "IMAGE_TAG=$(date +%Y%m%d)" >> ${GITHUB_ENV}
      - name: Checkout
        uses: actions/checkout@v5
      - name: Build with Podman
        run: |
          podman build \
            --tag ${{ env.IMAGE_NAME }}:${{ env.DEFAULT_TAG }} \
            --tag ${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }} \
            --file ./Containerfile \
            .
      - name: Login to GHCR
        uses: docker/login-action@v3
        if: github.event_name != 'pull_request' && github.ref == format('refs/heads/{0}', github.event.repository.default_branch)
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Push to GHCR
        if: github.event_name != 'pull_request' && github.ref == format('refs/heads/{0}', github.event.repository.default_branch)
        id: push
        run: |
          podman tag ${{ env.IMAGE_NAME }}:${{ env.DEFAULT_TAG }} ${{ env.IMAGE_REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.DEFAULT_TAG }}
          podman tag ${{ env.IMAGE_NAME }}:${{ env.DEFAULT_TAG }} ${{ env.IMAGE_REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}
          LATEST_DIGEST=$(podman push ${{ env.IMAGE_REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.DEFAULT_TAG }} --digestfile /tmp/latest-digest.txt && cat /tmp/latest-digest.txt)
          podman push ${{ env.IMAGE_REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}
          LATEST_IMAGE_REF="${{ env.IMAGE_REGISTRY }}/${{ env.IMAGE_NAME }}@${LATEST_DIGEST}"
          echo "digest=$LATEST_DIGEST" >> $GITHUB_OUTPUT
          echo "image-ref=$LATEST_IMAGE_REF" >> $GITHUB_OUTPUT
```

### Configuration

Now for the fun part. In those list files you can add the entries for how you want to customize your image. The are expecting a very simple list format. Here is how you would add packages to the
system:

```
firefox
vim
tmux
```

### Finishing up

The setup is done. Now all you have to do is push the code to Github, wait for the image to build, then switch to it from an atomic Fedora desktop using the command:

```bash
sudo bootc switch ghcr.io/{username}/{package_name}:latest
```

If you would like a clear example on more ways this can be used, I encourage you to check out the aforementioned version of this concept [here](https://github.com/wstobb/GFOS).
