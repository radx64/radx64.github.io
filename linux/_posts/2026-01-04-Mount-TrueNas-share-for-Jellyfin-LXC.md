---
layout: post
title: "Mounting TrueNAS shares for Jellyfin LXC"
categories: [linux, devops]
tags: [devops, blog, proxmox, lxc, jelllyfin, truenas, nfs, smb]

---

## Prologue
I'm playing around with hosting vairious services in LXC and virtual machines on Proxmox. One of the services I'm running is Jellyfin, a media server that allows me to stream my media collection to various devices. My media files are stored on a TrueNAS server (also hosted on a same proxmox host), and I wanted to mount the TrueNAS shares into the Jellyfin LXC container.

Sounds simple enough, right? Well, it turned out to be a bit more complicated than I initially thought. In this post, I'll document the steps I took to successfully mount the TrueNAS shares into the Jellyfin LXC container.


## Checklist
- Proxmox host with LXC container running Jellyfin
- TrueNAS server with NFS or SMB shares configured
- Network connectivity between Proxmox host and TrueNAS server


## Architecture

Here's a high-level overview of the architecture:

```
+--------------------------+ 
|  Proxmox Host            | 
|  +-------------------+   | 
|  |        105        |   | 
|  |    Jellyfin LXC   |   | 
|  +-------------------+   | 
|                          |
|  +-------------------+   | 
|  |        104        |   |
|  | TrueNAS  Server   |   | 
|  +-------------------+   |
+--------------------------+
```

## Approach 1: SMB/CIFS mount in LXC of TrueNAS share

Idea was simple - install `cifs-utils` in the LXC container and mount the TrueNAS SMB share directly in the container. However, this approach failed due to permission issues related to LXC's security model.

As soon as I added the mount command to `/etc/fstab` in the container and use `mount -a` command, I got the following error in `dmesg`

```
[ 1365.121700] audit: type=1400 audit(1767359435.236:971): apparmor="DENIED" operation="sendmsg" class="file" namespace="root//lxc-105_<-var-lib-lxc>" profile="rsyslogd" name="/run/systemd/journal/dev-log" pid=11944 comm="systemd-journal" requested_mask="r" denied_mask="r" fsuid=100000 ouid=100000

```
That means that AppArmor profile for the LXC container (which was set to Unprivileged) is blocking the mount operation. I could try to modify the AppArmor profile to allow the mount, but that would be a security risk and not recommended.

## Approach 2: Mount SMB share on Proxmox host and bind mount into LXC

This should work rigit? Mount smb share on Proxmox host and then bind mount it into the LXC container. So do the same "etc/fstab" trick on the Proxmox host:

```
//<truenas_IP>/media /mnt/truenas_media cifs credentials=/root/.smb_jellyfin,iocharset=utf8,_netdev,x-systemd.automount 0 0
```
Btw, I'm storing the credentials in `/root/.smb_jellyfin` file with following content:

```
username=jellyfin_user
password=super_secret_password
```

This time, the mount on the Proxmox host works fine. Now, I need to bind mount it into the LXC container. So I add the following line to the LXC container configuration file (`/etc/pve/lxc/105.conf`):

```
mp0: /mnt/truenas_media,mp=/mnt/media
```

And this approach works whenever TrueNAS server is reachable during the LXC container startup. However, if TrueNAS server is starting, is down or unreachable during the LXC container startup, the container fails to start with the following error:
```
run_buffer: 571 Script exited with status 19
lxc_init: 845 Failed to run lxc.hook.pre-start for container "105"
__lxc_start: 2047 Failed to initialize container "105"
TASK ERROR: startup for container '105' failed
```

## Approach 2.2: Hack startup order of containers

Proxmox let's you to define startup order and delay for containers. So I set the TrueNAS container to start first with a delay of 60 seconds before starting the Jellyfin container. This way, the TrueNAS server is guaranteed to be up and running before the Jellyfin container starts. This can be done using the Proxmox web interface or by editing the container configuration files directly *Options -> Startup/Shutdown order (order=1, order=2, etc...)*. Delay can be added using `up=60` option in the container configuration file or using web interface.
I know it is not elegant, but it let's give it a shot.

Results were miserable. It failed same way as before. Digging through documentation and forums seems that containers delay is done when container is started, but it cannot be started if the mount point is not available. So this approach is dead end.


## Approach 3: SMB automount mount on Proxmox host and optional mount point in LXC

This time, I decided to use systemd automount feature on the Proxmox host to mount the SMB share only when it is accessed. This way, the mount point will be available when the LXC container starts, even if the TrueNAS server is down or unreachable during startup.

On proxmox host I've created a new systemd script `/etc/systemd/system/mnt-truenas_media.automount`

```
[Unit]
Description=TrueNAS Media
After=network-online.target
Wants=network-online.target

[Mount]
What=//<truenas_IP>/media
Where=/mnt/truenas_media
Type=cifs
Options=_netdev,x-systemd.automount,credentials=/root/.smbcred,iocharset=utf8

[Install]
WantedBy=multi-user.target
```

And enabled it using:
```
systemctl daemon-reload
systemctl enable mnt-truenas_media.automount
```

Then I added the following line to the LXC container configuration file (`/etc/pve/lxc/105.conf`) (Jellyfin container):

```
mp0: /mnt/truenas_media,mp=/mnt/media,optional=1
```
`optional=1` means that if the mount point is not available during container startup, the container will still start without errors.

This approach worked like a charm! The Jellyfin container starts up without any issues, even if the TrueNAS server is down or unreachable during startup. When I access the `/mnt/media` directory inside the container, the SMB share is automatically mounted on the Proxmox host, and the media files are accessible in the Jellyfin container. I've also left containre startup order and delayed Jellyfin container by 60 seconds just to be sure everything is up and running.

## Epilogue

We ended up with something like that:
```
+----------------------------------------------+ 
|                      Proxmox Host            | 
|  +-------------------+                       | 
|  |        105        | ←--------+            | 
|  |    Jellyfin LXC   |   (opt)  |            | 
|  +-------------------+          |            | 
|                             SMB share        |
|                             automount        |
|  +-------------------+          ↑            | 
|  |        104        | ---------+            |
|  | TrueNAS  Server   |                       | 
|  +-------------------+                       |
+----------------------------------------------+
```

And that's it! I've successfully mounted the TrueNAS SMB share into the Jellyfin LXC container using systemd automount on the Proxmox host and optional mount point in the LXC container. This approach ensures that the Jellyfin container starts up without any issues, even if the TrueNAS server is down or unreachable during startup.

And yes, I know, I should use NFS instead of SMB for better performance and compatibility with Linux systems, and I will try that next.

## Disclaimer
This is not a tutorial how to do it. This is just my notes for myself and anyone interested in similar setup. Use it at your own risk.
