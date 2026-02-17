# Club-proxmox-server
Documentation on how i set this server up

First i set up the local ip address for the proxmox server in /etc/network/interfaces and /etc/hosts to run off of the gateway ip address of the switch in the room.

I then set up the LAN in /etc/network/interfaces. This LAN (192.168.1) will be used for our virtual machines to beable to connect to each other)

<img width="564" height="341" alt="image" src="https://github.com/user-attachments/assets/ebce37cb-8025-432b-9ce3-f0d7d4db99e5" />


Now its time to install a pfsense iso and create a vm with 2gb of ram, 16 gigs of storage and 1 core. Also make sure to have a network device bridge for both your WAN and LAN netwokr interfaces.

<img width="717" height="390" alt="image" src="https://github.com/user-attachments/assets/6b4c78b2-13ef-4f70-a7a6-d2d56b1f8282" />

After installing and setting everything up it should look like this (before setting up vlans)


I added new storage to my server and used lsblk to see the drives installed and their partitions. I used wipefs -all to wipe one drive but couldnt do the other because there was a linux lvm on that drive so i had to remove the logical volumes using lvchange -an and lvremove. After that i wiped the drives and this is what i saw
<img width="720" height="389" alt="image" src="https://github.com/user-attachments/assets/e073a723-4956-4766-918c-cb1f51cbc639" />

Now to add this to my proxmox.

<img width="989" height="170" alt="image" src="https://github.com/user-attachments/assets/88328e36-649d-48b4-bb0e-69b873b4f04f" />

These commands are how i added these ssds to my lvm in proxmox.


So a way easier way to add ssds is to actually just use the web interface.

<img width="1715" height="435" alt="image" src="https://github.com/user-attachments/assets/3d74e467-e7d8-4dd7-84b6-ed46f76d01cc" />

Is this photo, I added a new ssd which is shown by /dev/sdd (hard drive d). To do this make sure to select your node Scroll down and select disk to find any added disk. To set it up all you have to do is click the new ssd and select wipe disk at the top. 


Right now this is what the server is currently looking like.

<img width="310" height="424" alt="image" src="https://github.com/user-attachments/assets/1f95c8db-59be-4f9d-9240-8526a9cc545e" />

In this we joined another server as a cluster which was pretty simple. All you really had to do was select the existing data center at the top and go to this screen.
<img width="1919" height="307" alt="image" src="https://github.com/user-attachments/assets/67acd3a4-43f1-4193-b581-c2bb5f6af561" />

After coming here you would click create cluster then copy the information code into the your other proxmox server when you click join cluster on that server.

As you can see, there is a lot more stuff in both nodes. We have local and local-lvm which serve as our default space for machines and iso images. We also have ssd-snapshots for quick roll backs and box resets as well as extra space for any machines. We also have templates for our automation scripts. Lastly, we have Backups which will be our primary (Hopefully) iso and backup image storage.

<img width="174" height="145" alt="image" src="https://github.com/user-attachments/assets/587a41ea-f54c-4000-9cdf-9efc11f5e42e" />

Here are the different places you can format your disk to. LVM is the default and uses to run the containers/virtual machines. Directory seems to be a jack of all trades. and the others we dont use (not yet).

If i wanted to create a new lvm-thin i would do this
<img width="1581" height="663" alt="image" src="https://github.com/user-attachments/assets/04012572-0278-418d-8315-0bd60cddd31d" />

Obviously, here i dont have another ssd but this is just to show how to create them. To edit these, you would go to datacenter > storage > and then select which one you want to remove, add or edit.

<img width="1919" height="339" alt="image" src="https://github.com/user-attachments/assets/c45ad6c7-4935-40ac-a5b5-11f123783d79" />

So actually we set up a tailscale vpn instead so we can all work on this at home. Its running off of a container in the proxmox server that is advertising the server subnet across the vpn connection. 

Before i get into the vpn setup make sure to setup your repositories correctly.
<img width="1453" height="559" alt="image" src="https://github.com/user-attachments/assets/8bfd4f3b-b419-48b1-be99-ae1a52a0fbf3" />

Since we are using a free version of proxmox we arent getting enterprise updates. This causes our machine to not get updates at all so to fix that we need to disable the enterprise ones and get free repos.

There are two ways to do this. 

The CLI way through /etc/sources.list
<img width="623" height="337" alt="image" src="https://github.com/user-attachments/assets/684fa297-a35f-4326-b566-4b77ca0f789a" />

Or through the web UI
<img width="1607" height="715" alt="image" src="https://github.com/user-attachments/assets/04501dd9-5376-4ac3-b379-4f866d21f14e" />

For the web ui, all you have to do is disable any enterprise repos and add the no-subscription repo.

To set that up you would run this command on the proxmox root shell (not the container)
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/tools/addon/add-tailscale-lxc.sh)" 
Press space when you get the selection screen to select your container and then enter. 

After you set that up. Use tailscale up to print the link and make sure to connect that machine to your tailscale by signing into your account with the link that prints in the shell

<img width="539" height="117" alt="image" src="https://github.com/user-attachments/assets/51743c21-054e-4081-b200-fbae09036e1f" />

After that use these commands to allow the entire subnet of the server to be exposed 

<img width="632" height="95" alt="image" src="https://github.com/user-attachments/assets/381fcd04-0a51-47e1-b626-a8b58d7cf76d" />

Then enable the subnet on ur tailscale admin console.

for any issues with joining servers to your cluster make sure to use nano to edit the /etc/pve/corosync.conf and delete your node manually.

## Setting up WRCCDC environments.  

Using this command you will be able to download all of the images in your chosen drive's dump directory (/mnt/pve/Storage/dump/ or /var/lib/vz/dump/ (Default storage for local). 
`` wget -r -np -nd -A "*.vmz,*.vma.gz" https://archive.wrccdc.org/images/2026/wrccdc-2026-invitationals-2/ ``

After downloading all of the files do `` mv [old file name] [new file name.vma]`` It needs a to be .vma to be recognized by proxmox.

to set them up you have to first extract the server information from the vma file using ``vma extract -v [filename.vma] /tmp/[insert a new directory] `` Doing this extracts the server config file and a hard disk file for your vm.

After this i used this script to basically provission and set up the vm

```
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
qm set $VMID --cipassword 'OMGATREX1?'

# 6. START
echo "Starting VM..."
qm start $VMID
echo "Done! Trex is running."

```
Other way to set up the images would be to install them as .vma/.vma.zst  

If you go to the directory the were uploaded (/mnt/pve/Storage/dump in my case) You will be able to select an option in the web UI called Backups

<img width="239" height="212" alt="image" src="https://github.com/user-attachments/assets/787793e6-cf2e-4249-801e-2054e354d244" />


Here you will be able to see the files you uploaded and if you click restore and select the storage (Where the vms will run) you will be able to run the virtual machines.

<img width="636" height="188" alt="image" src="https://github.com/user-attachments/assets/89d94083-b17b-4cdb-8ed4-c2278d11a112" />

After its restored, it may not start because of the it will introduce a new network bridge to the system. All you have to do is setup a new LINUX network bridge

(COME BACK TO THIS

<img width="1012" height="124" alt="image" src="https://github.com/user-attachments/assets/26afc5e2-ae0b-4afa-b339-cb45baaa9ac4" />

When you run iptables -t nat -L -n -v You will see a NETMAP section in the post routing section. That is to Set up 1:1 NAT. Sadly we cant get it working so if you see it remove it.

iptables -t nat -D POSTROUTING [Line Number]


After doing that we need to add a MASQUERADE rule to push all of the public network connectivity to the LAN

iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

This command does that.


<img width="1231" height="467" alt="image" src="https://github.com/user-attachments/assets/3a35b398-7b02-4ec3-8f96-09888655267c" />

Make a container and install tailscale on it. Set up the netwokr interfaces to have the WAN from the router (Actual WIFI) and the LAN for the competition envrionment. After thats done set up tailscale and advertise the LAN. This should simulate 1:1 NAT

Make sure to also add net.ipv4.ip_forward = 1 to your /etc/sysctl.conf in your container so that the vpn bridge can work

sudo tailscale up --advertise-routes=192.168.220.0/24 The command to start tailscale and advertise the subnet

Now on to automation

https://adaptable-april-ff0.notion.site/actually-doing-evan-s-lab-a44ac1a906a74b5abd0b72062454da6c


https://login.tailscale.com/a/1b05fa43017790
