# Proxmox Setup

## Initial filesystem config

- Boot into the installation media. Select the filesystem as ZFS with RAID1. Deselect all drives and only select 2 SSDsfor this.
  - ashift=12
  - compress=zstd
  - checksum=sha256
  - copies=1
  - ARC max size=2048 MiB
  - hdsize=20 GB
 
## Intial setup script
The intial setup script to be run after installation is maintained [here](https://github.com/TommyTran732/Linux-Setup-Scripts).

