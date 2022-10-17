# Running Podman on Linux

- [Running Podman on Linux](#running-podman-on-linux)
  - [Introduction](#introduction)
  - [Documentation](#documentation)
  - [Installation](#installation)
    - [Option A - Manual Provisioning](#option-a---manual-provisioning)
      - [Create Virtual Machine](#create-virtual-machine)
      - [Install Podman - Rootless Mode](#install-podman---rootless-mode)


## Introduction

Pod manager tool or simple Podman is a popular container engine alternative to Docker or LXC. It runs OCI Containers in root or rootless mode.


## Documentation

- [Podman](https://podman.io/)


## Installation

In order to create Virtual Machines in declarative way you can leverage Vagrant with a backend hypervisor. Examples in this repository work on Hyper-V as well as Virtualbox.

For the examples below, I will leverage a WSL2 client environment with Hyper-V backend.


### Option A - Manual Provisioning

The following section describes how to install Podman manually on a Rocky Linux 8 OS.

#### Create Virtual Machine

Start by provisioning a new Virtual Machine.

```bash
cd installation/rocky-8
vagrant up && vagrant ssh
```

This should land you a user shell a good place to start playing with podman.

#### Install Podman - Rootless Mode

The benefit of using Podman in rootless mode is that the users don't need elevated privileges to run containers.

This is achieved using user namespaces which are unique for each user and provide process separation.

Within these namespaces, the UID mappings allow user to carry out tasks as a pseudo-root user within their own namespace.

The `podman` package is available from standard `appstream` repository and can be easily installed by `dnf`.

```bash
sudo dnf install -y podman
```

Verify that the `slirp4netns` was installed as well. This is required for managing container networking.

```bash
dnf list installed slirp4netns
```

> **Note**: In order to use `sudo` for privilege escalation, it is common that the user is member of `wheel` group. This can be confirmed by using `id -Gn` command. An alternative approach is to create a separate user file in the `/etc/sudoers.d/` directory.

Verify the default namespace configuration and user id mappings.

```bash
# Verify the maximum number of namespaces
sysctl -ar max_user_namespaces

# Verify user id mappings
cat /etc/subuid /etc/subgid
```

After installation, verify the podman `version` and `info`.

```bash
podman version
```

```bash
podman info
```