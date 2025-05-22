# Proxmox Setup

## Format drives as 4Kn

- Boot into a live ISO.
- Install the `nvme-cli` package.
- Check if the NVME drives support 4096 sector size with:

```
nvme id-ns /path/to/drive -H
```

![LBA](LBA.png)

- If the drives support 4096 sector size, run

```
nvme /path/to/drive -b 4096
```

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

![Bond](Bond.png)
 
## Intial setup script
The intial setup script to be run after installation is maintained [here](https://github.com/TommyTran732/Linux-Setup-Scripts).

