# Club Proxmox Server

Documentation for the Proxmox server I built to host our cybersecurity club's virtual infrastructure: practice ranges, competition environments, and shared labs the whole club can reach remotely.

This is a running build log. It covers the initial host and network setup, storage, clustering, remote access over Tailscale, and how I stood up the WRCCDC practice environments from the public image dumps.

## Table of Contents

- [Host and Network Setup](#host-and-network-setup)
- [pfSense](#pfsense)
- [Storage](#storage)
- [Clustering](#clustering)
- [Remote Access with Tailscale](#remote-access-with-tailscale)
- [Setting Up WRCCDC Environments](#setting-up-wrccdc-environments)
- [Automation](#automation)

## Host and Network Setup

First I set up the local IP address for the Proxmox server in `/etc/network/interfaces` and `/etc/hosts`, running off the gateway IP of the switch in the room.

Then I set up the LAN in `/etc/network/interfaces`. This LAN (`192.168.1.x`) is what our virtual machines use to talk to each other.

<img width="564" height="341" alt="image" src="https://github.com/user-attachments/assets/ebce37cb-8025-432b-9ce3-f0d7d4db99e5" />

## pfSense

Next I installed a pfSense ISO and created a VM for it: 2GB RAM, 16GB storage, 1 core. Make sure the VM has a network device bridge for both your WAN and LAN interfaces.

<img width="717" height="390" alt="image" src="https://github.com/user-attachments/assets/6b4c78b2-13ef-4f70-a7a6-d2d56b1f8282" />

After installing and configuring it, this is what it looks like (before setting up VLANs).

## Storage

I added new storage to the server and used `lsblk` to see the installed drives and their partitions. I ran `wipefs --all` to wipe one drive, but couldn't do the same to the other because it had a Linux LVM on it. To clear that one I had to remove the logical volumes first with `lvchange -an` and `lvremove`, then wipe it. Here's what I saw after:

<img width="720" height="389" alt="image" src="https://github.com/user-attachments/assets/e073a723-4956-4766-918c-cb1f51cbc639" />

Then I added the drives to Proxmox.

<img width="989" height="170" alt="image" src="https://github.com/user-attachments/assets/88328e36-649d-48b4-bb0e-69b873b4f04f" />

The commands above are how I added the SSDs to my LVM from the command line.

**The easier way:** you can do the whole thing from the web interface instead.

<img width="1715" height="435" alt="image" src="https://github.com/user-attachments/assets/3d74e467-e7d8-4dd7-84b6-ed46f76d01cc" />

In this screenshot I added a new SSD, shown as `/dev/sdd`. To do it through the UI, select your node, scroll down to **Disks**, and find the new disk. To set it up, click the new SSD and select **Wipe Disk** at the top.

Here's what the server looks like right now:

<img width="310" height="424" alt="image" src="https://github.com/user-attachments/assets/1f95c8db-59be-4f9d-9240-8526a9cc545e" />

### Storage Types

Here are the different formats you can use for a disk. **LVM** is the default and is what runs the containers and VMs. **Directory** is more of a jack-of-all-trades. The others we don't use yet.

<img width="174" height="145" alt="image" src="https://github.com/user-attachments/assets/587a41ea-f54c-4000-9cdf-9efc11f5e42e" />

To create a new **LVM-thin**, you'd do this:

<img width="1581" height="663" alt="image" src="https://github.com/user-attachments/assets/04012572-0278-418d-8315-0bd60cddd31d" />

I don't have another SSD to demo with here, so this is just to show the process. To edit storage, go to **Datacenter > Storage** and select the one you want to add, remove, or edit.

<img width="1919" height="339" alt="image" src="https://github.com/user-attachments/assets/c45ad6c7-4935-40ac-a5b5-11f123783d79" />

## Clustering

We joined a second server as a cluster, which was surprisingly simple. Select the existing Datacenter at the top and go to the cluster screen.

<img width="1919" height="307" alt="image" src="https://github.com/user-attachments/assets/67acd3a4-43f1-4193-b581-c2bb5f6af561" />

Click **Create Cluster**, then copy the join information and paste it into your other Proxmox server when you click **Join Cluster** on that machine.

After the second node was joined, both nodes had a lot more available. We have `local` and `local-lvm` as the default space for machines and ISO images, `ssd-snapshots` for quick rollbacks and box resets (plus extra machine space), `templates` for our automation scripts, and `Backups`, which is meant to be our primary ISO and backup-image storage.

> **Troubleshooting:** if you have issues joining a node to the cluster, edit `/etc/pve/corosync.conf` with `nano` and remove the problem node manually.

## Remote Access with Tailscale

We set up a Tailscale VPN so the whole club can work on the server from home. It runs off a container in the Proxmox server that advertises the server's subnet across the VPN connection.

### Fix Your Repositories First

Before setting up the VPN, get your repositories configured correctly.

<img width="1453" height="559" alt="image" src="https://github.com/user-attachments/assets/8bfd4f3b-b419-48b1-be99-ae1a52a0fbf3" />

Since we're on the free version of Proxmox, we don't get enterprise updates, which means the machine doesn't get updates at all. To fix that, disable the enterprise repos and add the free (no-subscription) ones.

There are two ways to do this:

**CLI:** edit `/etc/apt/sources.list`

<img width="623" height="337" alt="image" src="https://github.com/user-attachments/assets/684fa297-a35f-4326-b566-4b77ca0f789a" />

**Web UI:** disable any enterprise repos and add the no-subscription repo.

<img width="1607" height="715" alt="image" src="https://github.com/user-attachments/assets/04501dd9-5376-4ac3-b379-4f866d21f14e" />

### Install Tailscale on a Container

Run this on the Proxmox **root shell** (not inside the container):

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/tools/addon/add-tailscale-lxc.sh)"
```

When the selection screen appears, press **space** to select your container, then **enter**.

Once that's done, run `tailscale up` to print the login link, then sign into your account with that link to connect the machine to your tailnet.

<img width="539" height="117" alt="image" src="https://github.com/user-attachments/assets/51743c21-054e-4081-b200-fbae09036e1f" />

### Advertise the Subnet

Use these commands to expose the server's entire subnet across the VPN:

<img width="632" height="95" alt="image" src="https://github.com/user-attachments/assets/381fcd04-0a51-47e1-b626-a8b58d7cf76d" />

Then enable the subnet from your Tailscale admin console.

## Setting Up WRCCDC Environments

This is where the server actually earns its keep. I pulled the public WRCCDC image dumps and stood them up as practice VMs.

### Download the Images

This command downloads all the images into your chosen drive's dump directory (`/mnt/pve/Storage/dump/`, or `/var/lib/vz/dump/` for default local storage):

```bash
wget -r -np -nd -A "*.vmz,*.vma.gz" https://archive.wrccdc.org/images/2026/wrccdc-2026-invitationals-2/
```

After downloading, rename the files so Proxmox recognizes them. They need a `.vma` extension:

```bash
mv [old file name] [new file name].vma
```

### Method 1: Extract and Provision by Script

First, extract the server info from the VMA file. This pulls out the server config and a hard-disk file for your VM:

```bash
vma extract -v [filename.vma] /tmp/[new directory]
```

Then I used this script to provision and set up the VM:

```bash
#!/bin/bash

# 1. DEFINE VARIABLES
TARGET="/tmp/restore-trex"

# Trex Settings (Matches your screenshot)
VMID=10006
NAME="team00-trex"
MAC="BC:24:11:95:93:3B"
IP="192.168.220.12/24"
STORAGE="local-lvm"

# 2. CREATE VM SHELL
echo "Creating VM $VMID ($NAME)..."
qm create $VMID \
  --name $NAME \
  --memory 4096 --cores 2 --sockets 1 --numa 0 \
  --cpu x86-64-v2-AES \
  --ostype win10 \
  --agent 1 \
  --scsihw virtio-scsi-single \
  --net0 virtio,bridge=vmbr0,macaddr=$MAC

# 3. IMPORT DISK
echo "Importing disk from $TARGET..."
qm importdisk $VMID "$TARGET/disk-drive-scsi0.raw" $STORAGE

# 4. ATTACH DISK & BOOT SETTINGS
qm set $VMID --scsi0 $STORAGE:vm-$VMID-disk-0
qm set $VMID --boot order=scsi0

# 5. CLOUD-INIT SETTINGS
echo "Configuring Network..."
qm set $VMID --ide2 $STORAGE:cloudinit
qm set $VMID --ipconfig0 ip=$IP,gw=192.168.220.2
# Nameserver is set to localhost because THIS machine is the DNS server
qm set $VMID --nameserver 127.0.0.1
qm set $VMID --ciuser Administrator
qm set $VMID --cipassword '<SET_A_PASSWORD>'

# 6. START
echo "Starting VM..."
qm start $VMID
echo "Done! Trex is running."
```

> **Security note:** set `--cipassword` from an environment variable or prompt instead of hardcoding it. Anything committed here is public.

### Method 2: Restore from the Web UI

You can also install the images directly as `.vma`/`.vma.zst` backups.

Go to the directory they were uploaded to (`/mnt/pve/Storage/dump` in my case) and you'll see a **Backups** option in the web UI.

<img width="239" height="212" alt="image" src="https://github.com/user-attachments/assets/787793e6-cf2e-4249-801e-2054e354d244" />

Here you can see your uploaded files. Click **Restore**, select the storage where the VMs will run, and it'll build the machine.

<img width="636" height="188" alt="image" src="https://github.com/user-attachments/assets/89d94083-b17b-4cdb-8ed4-c2278d11a112" />

### Fixing the Network After Restore

After a restore, the VM may not start because it introduces a new network bridge to the system. Fix it by setting up a new **Linux network bridge**.

<img width="1012" height="124" alt="image" src="https://github.com/user-attachments/assets/26afc5e2-ae0b-4afa-b339-cb45baaa9ac4" />

_(revisit this section)_

When you run `iptables -t nat -L -n -v`, you'll see a **NETMAP** entry in the POSTROUTING section. That's meant to set up 1:1 NAT, but we couldn't get it working, so remove it if you see it:

```bash
iptables -t nat -D POSTROUTING [line number]
```

Then add a MASQUERADE rule to push all public connectivity to the LAN:

```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

<img width="1231" height="467" alt="image" src="https://github.com/user-attachments/assets/3a35b398-7b02-4ec3-8f96-09888655267c" />

### Simulating 1:1 NAT with a Tailscale Container

Since we couldn't get true 1:1 NAT working, here's the workaround:

Make a container and install Tailscale on it. Configure its network interfaces so it has the WAN from the router (the actual WiFi) and the LAN for the competition environment. Then set up Tailscale and advertise the LAN. This simulates 1:1 NAT.

Add `net.ipv4.ip_forward = 1` to `/etc/sysctl.conf` inside the container and apply it with `sysctl -p`, so the VPN bridge works.

Next, configure the container's permission file on the Proxmox host node. This is based on your container ID:

```bash
nano /etc/pve/lxc/105.conf
```

<img width="1158" height="341" alt="image" src="https://github.com/user-attachments/assets/b2f8e701-32e0-4037-bfcb-d58321e58eba" />

Add these lines, then reboot the container:

```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

Finally, start Tailscale and advertise the subnet:

```bash
sudo tailscale up --advertise-routes=192.168.220.0/24
```

## Automation

Automation notes and lab-provisioning work are documented separately here:

- [Lab automation notes (Notion)](https://adaptable-april-ff0.notion.site/actually-doing-evan-s-lab-a44ac1a906a74b5abd0b72062454da6c)

> **Heads up:** the original README ended with a live Tailscale login link (`login.tailscale.com/a/1b05fa43017790`). Those links authenticate a device to your tailnet, so I've left it out here. Remove it from the live repo if it's still there.
