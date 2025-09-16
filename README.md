# Club-proxmox-server
Documentation on how i set this server up

First i set up the local ip address for the proxmox server in /etc/network/interfaces and /etc/hosts to run off of the gateway ip address of the switch in the room.

I then set up the LAN in /etc/network/interfaces. This LAN (192.168.1) will be used for our virtual machines to beable to connect to each other)

<img width="564" height="341" alt="image" src="https://github.com/user-attachments/assets/ebce37cb-8025-432b-9ce3-f0d7d4db99e5" />


Now its time to install a pfsense iso and create a vm with 2gb of ram, 16 gigs of storage and 4 cores. Also make sure to have a network device bridge for both your WAN and LAN netwokr interfaces.

<img width="717" height="390" alt="image" src="https://github.com/user-attachments/assets/6b4c78b2-13ef-4f70-a7a6-d2d56b1f8282" />

After installing and setting everything up it should look like this (before setting up vlans)
