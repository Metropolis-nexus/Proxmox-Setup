# Proxmox Setup

## Use 4K sectors

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

- Boot into the installation media. Select the 1 single drive and set the filesystem as ZFS. 
  - ashift=12
  - compress=zstd
  - checksum=sha256
  - copies=1
  - ARC max size=2048 MiB
  - hdsize=32 GB

## Setup bond
If the server has redundant networking, setup a bond (Ative-Backup or LACP):

![Bond](Bond.png)
 
Then set `vmbr0` bridge port to the bond.

## Intial setup script
The intial setup script to be run after installation is maintained [here](https://github.com/TommyTran732/Linux-Setup-Scripts).

## Setup Firewalling

Datacenter -> Firewall -> Add rules for PVE web panel, SSH, and ICMP
Datacenter -> Firewall -> Options -> Firewall -> Enable
Node name -> Firewall -> Options -> Enable TCP flags filter and nftables

![Firewall-1](Firewall-1.png)
![Firewall-2](Firewall-2.png)
![Firewall-3](Firewall-3.png)