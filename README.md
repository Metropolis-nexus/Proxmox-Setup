# Proxmox Setup

## Initial filesystem config

- Boot into the installation media. Select the filesystem as ZFS (RAID1). Deselect all drives and only select 2 SSDs for this.
  - ashift=12
  - compress=zstd
  - checksum=sha256
  - copies=1
  - ARC max size=2048 MiB
  - hdsize=32 GB

## Setup bond
If the server has redundant networking, setup a bond (Ative-Backup or LACP):

![Bond](bond.png)
 
## Intial setup script
The intial setup script to be run after installation is maintained [here](https://github.com/TommyTran732/Linux-Setup-Scripts).

