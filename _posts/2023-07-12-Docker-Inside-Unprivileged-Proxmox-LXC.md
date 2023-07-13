---
title: "Docker Inside an Unprivileged Proxmox LXC Container"
date: 2023-07-12 18:00 -0700
categories: [Virtualization, Linux, Docker]
tags: [docker, proxmox, linux,]
excerpt: "Instead of running a full VM just to run a bunch of Docker containers, I wanted to utilize the LXC feature of Proxmox. It allows for running a full Debian system in a container that, instead of emulating the hardware of a complete virtual machine, shares hardware and kernel with the Proxmox host. "
# toc: true
classes: single # single, wide, splash
header:
  image: /assets/images/powershell_header-1280x200.png
  teaser: /assets/images/powershell-250.png
  og_image: /assets/images/powershell-250.png
---

Instead of running a full VM just to run a bunch of Docker containers, I wanted to utilize the LXC feature of Proxmox. It allows for running a full Debian system in a container that, instead of emulating the hardware of a complete virtual machine, shares hardware and kernel with the Proxmox host. 

It also allows to operate containers in an unprivileged mode, increasing security by reducing the permission level. But in order for this to work there are some special preparations needed.

## Note

There is one thing that won't work in this scenario due to the lack of permissions: setting a domain name or an FQDN in a docker container. If either the `domainname` setting is used in the `docker-compose.yml`, or if the `hostname` setting contains an FQDN instead of a regular hostname (no periods!), the start of the container will fail with a permission error.

However, there is nothing speaking against using a hostname with an app that runs inside the container, it just can not be set as a container parameter.

## Docker inside Proxmox LXC

Set up an unprivileged container in Proxmox using the latest Debian template (at the time of writing this is Debian 12 "Bookworm").

### Inside the container

The following describes the basic setup inside the container, the commands are run as root.

#### Configure password-less SSH login

All that's needed to allow a key-based login is adding the respective public key to the root user's `authorized_keys` file:

```sh
echo 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIB0t4snxPv/aibNCtIujhkwVBRclKFtXKDwCG5WBANuj mcfisch@local' > /root/.ssh/authorized_keys
```

Now the local system's key-store (Debian/Ubuntu keyring, Windows SSH-agent, etc.) can store the private part of the key, asking for its password once, and from there password-less SSH logins should make the following even easier, especially if one doesn't use the Proxmox Gui to connect to the container.

#### Update packages

Next we update the repo list and install all package updates.

```sh
apt update && apt upgrade -y
```

A reboot shouldn't be necessary as the container uses the host's kernel, but it can help with getting the services started as supposed. It also helps with verifying that all settings are properly applied on boot.

#### Install additional packages

Some tools might be needed for later steps, or just to satisfy personal preferences. I use `curl` and `vim` a lot, so these are on my list. `fuse-overlayfs` is necessary as in an unprivileged LXC container there are some permission issues prohibiting the default docker storage driver `overlayfs` from working correctly. `vfs` could be an alternative, but it lacks the space-saving features of `overlayfs` and can cause a massive headache by using disk space uncontrollably. That's where `fuse-overlayfs` comes in, the FUSE implementation of `overlayfs`.

```sh
apt install -y curl vim fuse-overlayfs
```

Once `apt` has finished, open the container's options in Proxmox and enable `FUSE` under the `Features` setting. Restart the container.

#### Install Docker

Now the container is ready to get Docker. I prefer the convenience script that Docker provides. And since `curl` is already on the system I use it to fetch the script. For `wget` adjust the command accordingly (`curl`'s `-L` stands for "follow redirects").

```sh
curl -fsSL https://get.docker.com | sh
```

Once the script is done the system should have the docker container service running.

Now configure the new storage driver and restart the service.

```sh
cat <<EOF >/etc/docker/daemon.json 
{
  "storage-driver": "fuse-overlayfs"
}
EOF
systemctl restart docker.service
```

#### Create regular user

In order to avoid working with the root user all the time, as even in a rootless container this is still the superuser that has all available permissions. So I create a regular user that is also allowed to maintain docker resources.

```sh
useradd -m -d /home/mcfisch -U -s /bin/bash mcfisch
usermod -aG docker mcfisch
echo 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIB0t4snxPv/aibNCtIujhkwVBRclKFtXKDwCG5WBANuj mcfisch@local' > /home/mcfisch/.ssh/authorized_keys
chown -R mcfisch:mcfisch /home/mcfisch/.ssh/
chown -R mcfisch:docker /docker/
chmod 700 /home/mcfisch/.ssh/
chmod 600 /home/mcfisch/.ssh/authorized_keys
```

The ssh key allows me to directly connect to the user without setting a password, so a password-based login will always fail.

### Set up a container

Now we can begin setting up docker containers. I usually prefer docker-compose files combined with environmental variables in either `.env` files or stored in my shell profile.

The process usually resembles something like this:

```sh
mkdir -p /docker/grafana && cd /docker/grafana
curl -fsSL https://github.com/docker/awesome-compose/raw/master/prometheus-grafana/compose.yaml -o /docker/grafana/docker-compose.yml
docker compose -f /docker/grafana/docker-compose.yml pull
docker compose -f /docker/grafana/docker-compose.yml up -d
```

If all goes well this will start an instance of each `Prometheus` and `Grafana`, using the sample `docker-compose` file from the Docker github repo.

*Happy Containerizing!*
