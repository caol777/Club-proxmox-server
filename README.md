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


Now on to automation

https://adaptable-april-ff0.notion.site/actually-doing-evan-s-lab-a44ac1a906a74b5abd0b72062454da6c


https://login.tailscale.com/a/1b05fa43017790
