# Running Podman on Linux

- [Running Podman on Linux](#running-podman-on-linux)
  - [Introduction](#introduction)
  - [Documentation](#documentation)
  - [Installation](#installation)
    - [Option A - Manual Provisioning](#option-a---manual-provisioning)
      - [Creating Virtual Machine](#creating-virtual-machine)
      - [Installing Podman - Rootless Mode](#installing-podman---rootless-mode)
  - [Deploying containers](#deploying-containers)
  - [Inspecting image](#inspecting-image)
  - [Interacting with container](#interacting-with-container)
  - [Managing container lifecycle](#managing-container-lifecycle)
  - [Managing persistent data](#managing-persistent-data)


## Introduction

Pod manager tool or simple Podman is a popular container engine alternative to Docker or LXC. It runs OCI Containers in root or rootless mode.


## Documentation

- [Podman](https://podman.io/)


## Installation

In order to create Virtual Machines in declarative way you can leverage Vagrant with a backend hypervisor. Examples in this repository work on Hyper-V as well as Virtualbox.

For the examples below, I will leverage a WSL2 client environment with Hyper-V backend.


### Option A - Manual Provisioning

The following section describes how to install Podman manually on a Rocky Linux 8 OS.

#### Creating Virtual Machine

Start by provisioning a new Virtual Machine.

```bash
cd installation/rocky-8
vagrant up && vagrant ssh
```

This should land you a user shell a good place to start playing with podman.

#### Installing Podman - Rootless Mode

The benefit of using Podman in rootless mode is that the users don't need elevated privileges to run containers.

This is achieved using user namespaces which are unique for each user and provide process separation.

Within these namespaces, the UID mappings allow user to carry out tasks as a pseudo-root user within their own namespace.

The `podman` package is available from standard `appstream` repository and can be easily installed by `dnf`.

```bash
sudo dnf install -y podman
```

> **Note**: In order to use `sudo` for privilege escalation, it is common that the user is member of `wheel` group. This can be confirmed by using `id -Gn` command. An alternative approach is to create a separate user file in the `/etc/sudoers.d/` directory.

> **Note**: I also highly suggest to install the `bash-completion` package. It will allow you to use `TAB` to complete most of the commands.


Verify that the `slirp4netns` was installed as well. This is required for managing container networking.

```bash
dnf list installed slirp4netns
```

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


## Deploying containers

Now that we have Podman installed, its time to run some containers. These ship in form of an container images and can be pulled from image registry.

To list existing images stored locally use `podman image ls` command. To list all containers (running and stopped) use the `podman ps -a` command.

The list of currently used registries can be retrieved from `podman info` with a filter flag.

```bash
podman info -f '{{range index .Registries "search"}}{{.}}\n{{end}}'
```

The output should contain the following entries.

```bash
registry.access.redhat.com
registry.redhat.io
docker.io
```

These are the default and are defined in `/etc/containers/registries.conf`.

```bash
# Using grep to filter comments and empty lines
grep -v '^\s*$\|^\s*\#' /etc/containers/registries.conf

# Using sed to filter comments and empty lines
sed -E '/^(#|$)/d' /etc/containers/registries.conf
```

In order to search for an image use the `podman search` command.

```bash
podman search httpd
```

To start an container use `podman run` command. The `it` argument starts an interactive terminal session in bash shell. This will automatically download the image if it not present locally.

```bash
podman run -it rhscl/httpd-24-rhel7 /bin/bash
```

To prove that `bin/bash` is the only process running in this container use the `ps -elf` command.

```bash
ps -elf | grep -v ps
```

To detach from interactive console without stopping the container use the `CTRL-p + CTRL-q`. To reattach use `podman attach` with container id.


## Inspecting image

When you are using 3rd party images it is useful to do an inspection in order understand which options are supported.

For example to list the exposed ports you can use the following command:

```bash
podman inspect registry.access.redhat.com/rhscl/httpd-24-rhel7 | grep expose
```


## Interacting with container

To run a container in background use `-d` argument and map these ports to host use the `-p` argument. You can also use the `--publish-all` to map exposed ports defined in the image. Afterwards you can list the mapped ports using `podman port` command.

```bash
podman run --name=www -d -p 8000:8080 rhscl/httpd-24-rhel7
```

To enter an existing container use the `exec` argument.

```bash
podman exec -it www /bin/bash
```

Verify the application from within the container.

```bash
curl -I localhost:8080
```

Verify the application from host.

```bash
curl -I localhost:8000
```

In order to expose the application outside of host you need to ensure host firewall is configured accordingly.

```bash
sudo firewall-cmd --add-port 8000/tcp
```


## Managing container lifecycle

In order to stop an existing container use the `podman stop` command. You can supply container name or id or use `-l` to refer to last container or `-a` to refer to all containers. The container configuration will remain unchanged.

```bash
podman stop www
```

When the container is misbehaving it is possible to send `SIGKILL` to its `PID 1` using the `kill` argument.

```bash
podman kill www
```

To restart a running container use `podman restart`.

```bash
podman restart www
```

To delete a running container use `podman rm` with `-f` flag.

```bash
podman rm -f www
```


## Managing persistent data

In order to persist data that while using containers, we can create a bind mount. This will mount a host directory or a named volume into running container.

The following example maps the `web-files` in the current working directory to default `html` directory within the container.

```bash
podman run --name=www -d \
           -p 8000:8080 \
           -v $PWD/web-files:/var/www/html \
           rhscl/httpd-24-rhel7
```

> **Note**: If SELinux is used, the `container_file_t` type should be set on host directory. `chcon -Rvt container_file_t web-files/`. WIthout this, container user will not be able to read this mapped directory. See below:

```bash
# Before changing SElinux context
podman exec -it www stat -c %A,%C /var/www/html
drwxrwxr-x,unconfined_u:object_r:user_home_t:s0

# After changing SElinux context
podman exec -it www stat -c %A,%C /var/www/html
drwxrwxr-x,unconfined_u:object_r:container_file_t:s0
```