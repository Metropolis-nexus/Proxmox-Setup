# Proxmox Setup

## Use 4K sectors

- Boot into a live ISO.
- Install the `nvme-cli` package.
- Check if the NVME drives support 4096 sector size with:

```bash
nvme id-ns /path/to/drive -H
```

![LBA](LBA.png)

- If the drives support 4096 sector size, run

```bash
nvme /path/to/drive -b 4096
```

## Initial filesystem config

- Boot into the installation media. Select the 2 drives and use ZFS RAID 1. 
  - ashift=12
  - compress=zstd
  - checksum=sha256
  - copies=1
  - ARC max size=16384 MiB
  - hdsize=32 GB

- Install Proxmox

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

## Reformat the first drive
Format the first drive for the encrypted installation (different from the unencrypted drive the system currently runs on). For our purposes, lets call this first drive drive1.

```bash
drive1='/dev/disk/by-id/nvme-FIRSTDRIVESERIALNUMBER'
```

```bash
zpool detach rpool "${drive1}"
blkdiscard --force "${drive1}" # Or secure erase
wipefs "${drive1}" --all
sgdisk -g "${drive1}"
sgdisk -I -n 1:0:+1G -t 0:ef00 -c 0:'ESP' "${drive1}"
mkfs.fat "${drive1}-part1" -F 32 -n 'ESP'
proxmox-boot-tool clean
proxmox-boot-tool init "${drive1}-part1"
sgdisk -I -n 2:0:+32G -t 0:8309 -c 0:'PVE' "${drive1}"
sgdisk -I -n 3:0:0 -t 0:bf01 -c 0:'PVE Data' "${drive1}"
```

## Configure pve pool

```bash
apt install cryptsetup-initramfs
cryptsetup luksFormat "${drive1}-part2"
cryptsetup open --allow-discards --persistent "${drive1}-part2" cryptroot1

zpool create \
    -m none \
    -o ashift=12 \
    -o autoexpand=on \
    -o autotrim=on \
    -o comment='PVE' \
    -o failmode=wait \
    -O acltype=posix \
    -O atime=off \
    -O checksum=blake3 \
    -O compression=zstd-3 \
    -O dnodesize=auto \
    -O overlay=off \
    -O volmode=dev \
    -O xattr=sa \
    -O normalization=formD \
    pve /dev/mapper/cryptroot1
```

## Reconfigure the initramfs

```bash
echo "cryptroot1 ${drive1}-part2 none initramfs,keyscript=decrypt_keyctl" >> /etc/crypttab
sed -i 's/rpool/pve/' /etc/kernel/cmdline
zfs set mountpoint=none rpool
zfs set mountpoint=none rpool/ROOT
zfs set mountpoint=none rpool/ROOT/pve-1
zfs set mountpoint=none rpool/data
zfs set mountpoint=none rpool/var-lib-vz
update-initramfs -u -k all
```

## Migrate to pve pool

```bash
zfs snapshot -r rpool/ROOT@copy
zfs send -R rpool/ROOT@copy | zfs receive pve/ROOT
```

## Clean up and partition the second drive

- Reboot into the new system
- Datacenter -> Storage -> Remove `local-zfs`.

```bash
zfs export rpool
rm -rf /rpool
drive2='/dev/disk/by-id/nvme-SECONDDRIVESERIALNUMBER'
blkdiscard --force "${drive2}" # Or secure erase
wipefs "${drive2}" --all
sgdisk -g "${drive2}"
sgdisk -I -n 1:0:+1G -t 0:ef00 -c 0:'ESP' "${drive2}"
mkfs.fat "${drive2}-part1" -F 32 -n 'ESP'
proxmox-boot-tool clean
proxmox-boot-tool init "${drive2}-part1"
sgdisk -I -n 2:0:+32G -t 0:8309 -c 0:'PVE' "${drive2}"
sgdisk -I -n 3:0:0 -t 0:bf01 -c 0:'PVE Data' "${drive2}"
```

## Add mirrored drives to encrypted pool

```bash
cryptsetup luksFormat "${drive2}-part2" # Use the same password as the other partition
cryptsetup open --allow-discards --persistent "${drive2}-part2" cryptroot2
zpool attach pve /dev/mapper/cryptroot1 /dev/mapper/cryptroot2
```

## Reconfigure the initramfs

```bash
echo "cryptroot2 ${drive2}-part2 none initramfs,keyscript=decrypt_keyctl" >> /etc/crypttab
update-initramfs -u -k all
```

## Change root password

```bash
passwd
```

## Rotate Proxmox Secrets

```bash
rm -rf .ssh/*
ssh-keygen -t rsa -b 16384 -C root@<NODE NAME> -N '' -f /root/.ssh/id_rsa # You must generate an RSA key. If you generate an ed25519 key and name it id_rsa, Proxmox will keep spamming the authorized_keys file every boot.
rm /etc/ssh/ssh_host_*
dpkg-reconfigure openssh-server
rm -rf /etc/pve/priv/*
reboot
```

## Setup TLS certificate

- Datacenter -> ACME -> Add an account
- Setup CAA records
- Node Name -> Certificates -> ACME -> Add a challenge and order a certificate

## Cleanup old snapshots

```bash
zfs destroy pve/ROOT/pve-1@copy
zfs destroy pve/ROOT@copy
```
