---
layout: post
title: "Revisiting mounting TrueNAS shares for Jellyfin LXC"
categories: [linux, devops]
tags: [devops, blog, proxmox, lxc, jelllyfin, truenas, nfs, smb]

---

# The story so far

Some time ago I wrote a post about mounting a TrueNAS SMB share into a Jellyfin LXC container running on Proxmox.

At the time it looked like a perfectly reasonable setup:

```text
TrueNAS VM
    ↓ SMB/CIFS share
Proxmox host mount
    ↓ bind mount
Jellyfin LXC
```

The container started, Jellyfin saw the media library, hardware transcoding worked, and everything seemed stable.

Well... mostly.

After some time I started getting random startup failures after host reboots. Sometimes Jellyfin started correctly, sometimes it completely failed to boot.

This post explains what was actually wrong with the original setup and how I migrated it to a much more reliable NFS-based solution.

Previous article:

https://radx64.github.io/posts/Mount-TrueNas-share-for-Jellyfin-LXC/

---

# The Original Setup

The original guide used a CIFS/SMB mount on the Proxmox host.

Example `/etc/fstab` entry:

```fstab
//192.168.1.22/media /mnt/truenas_media cifs credentials=/root/.smb_jellyfin,iocharset=utf8,_netdev,x-systemd.automount 0 0
```

Then the directory was bind-mounted into the Jellyfin container:

```conf
mp0: /mnt/truenas_media,mp=/media
```

The idea itself was not bad.

The problem was the dependency chain hidden underneath.

---

# What Was Actually Failing

The important thing I missed originally was this:

```text
Jellyfin LXC
    depends on
Proxmox host mount
    depends on
TrueNAS SMB service
    depends on
TrueNAS VM finishing boot
```

And this is where things got fragile.

A VM being marked as "started" does not mean all services inside the VM are ready.

TrueNAS still needs time to:

- import ZFS pools
- initialize middleware
- bring networking online
- start SMB services

Meanwhile Proxmox already tries to start the Jellyfin container.

When LXC starts, it validates all bind mounts during the pre-start phase. If the SMB mount is unavailable at that exact moment, container startup aborts.

Typical errors looked like this:

```text
run_buffer: 571 Script exited with status 19
lxc_init: 845 Failed to run lxc.hook.pre-start
TASK ERROR: startup for container failed
```

Or:

```text
mount error(113): could not connect to 192.168.1.22
CIFS: VFS: Error connecting to socket
```

I initially tried solving this with startup ordering and delays:

```conf
startup: order=2,up=300
```

But this only delays container startup.

It does not guarantee SMB is actually ready.

---

# Another Mistake - Mixing Multiple Mount Systems

At some point I also added custom systemd `.mount` and `.automount` units.

That made the setup even more confusing.

I ended up mixing:

- `/etc/fstab`
- manual `.mount` units
- manual `.automount` units
- CIFS automount options

This created conflicting automount behavior and made debugging unnecessarily difficult.

In the end the solution turned out to be much simpler.

---

# The Proper Solution - NFS

Instead of SMB, I migrated everything to NFS.

The new architecture looks like this:

```text
TrueNAS VM
    ↓ NFS export
Proxmox host mount
    ↓ bind mount
Jellyfin LXC
```

This completely removed the SMB dependency and solved all startup reliability problems.

NFS is simply a much better fit for Linux-to-Linux virtualization environments.

Benefits:

- simpler authentication model
- fewer startup race conditions
- better integration with Linux
- cleaner systemd automount behavior
- more reliable reconnects

---

# Configuring NFS on TrueNAS

In TrueNAS:

1. Go to Shares -> NFS Shares
2. Add your media dataset
3. Allow your LAN subnet
4. Enable NFS service

Example export:

```text
/mnt/Storage/media
```

Allowed network:

```text
192.168.1.0/24
```

Verify exports from Proxmox:

```bash
showmount -e 192.168.1.22
```

Example output:

```text
Export list for 192.168.1.22:
/mnt/Storage/media *
```

---

# Mounting NFS on Proxmox

Install NFS tools:

```bash
apt install nfs-common
```

Create mount directory:

```bash
mkdir -p /mnt/truenas_media
```

Then add this to `/etc/fstab`:

```fstab
192.168.1.22:/mnt/Storage/media /mnt/truenas_media nfs defaults,_netdev,x-systemd.automount,nofail,noatime 0 0
```

Reload systemd:

```bash
systemctl daemon-reload
systemctl restart remote-fs.target
```

Test the mount:

```bash
ls /mnt/truenas_media
```

---

# Jellyfin LXC Configuration

The LXC configuration itself remained almost unchanged:

```conf
mp0: /mnt/truenas_media,mp=/media
```

This part of the original setup was actually correct.

The unstable part was the SMB-backed host mount.

---

# Cleaning Up Old systemd Mount Units

If you previously created custom `.mount` or `.automount` units for the SMB share, remove them.

Stop and disable units:

```bash
systemctl stop mnt-truenas_media.mount
systemctl stop mnt-truenas_media.automount

systemctl disable mnt-truenas_media.mount
systemctl disable mnt-truenas_media.automount
```

Remove unit files:

```bash
rm -f /etc/systemd/system/mnt-truenas_media.mount
rm -f /etc/systemd/system/mnt-truenas_media.automount
```

Reload systemd:

```bash
systemctl daemon-reload
systemctl daemon-reexec
```

Use `/etc/fstab` as the single source of truth.

---

# Final Thoughts

The original SMB setup worked most of the time, which made it difficult to notice the architectural problem early on.

The real issue was not SMB performance.

The real issue was coupling container startup to a network service that may not yet exist during boot.

After migrating to NFS:

- Jellyfin starts reliably
- no more LXC pre-start failures
- no more CIFS reconnect issues
- reboots became predictable again

If you are running:

```text
TrueNAS VM + Proxmox + Jellyfin LXC
```

I strongly recommend using NFS instead of SMB for media storage.
