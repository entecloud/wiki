---
date: 2025-09-21T10:00:00+05:30
title: Installing Linux with FDE and BTRFS
---

Ever since I started exploring more on privacy / encryption, I always wanted to try out new tools, experiment with them, break them and finally learn something from it. [LUKS](https://gitlab.com/cryptsetup/cryptsetup/) is one of those tools I discovered lately, but been using consistently for past few years. I made sure that every personal machine I touched, is configured with some kind of encryption, be it LUKS, or a simple rclone crypt remote.

Okay, enough with the backgrounds, let's get down to business.

## Step 0: Prerequisites

I don't know whether this will be qualified as a beginner level instructions, as I don't cover all the installation setup, just the disk partitioning phase of a typical Linux installation. I assume you already know the other stuff which is not that deep.

Although, this would work with any distro of your choice, I used Debian Trixie while making this instructions. Not that it matters, just saying.

I will be using `nvme0n1` as my disk name throughout this instructions. It might not match with your disk, but the partition suffixes like `p1`, `p2` will be same (almost). But do cross check with your disk setup.

## Step 1: Partitioning

Even if we wanted to go with Full Disk Encryption (FDE), we still have to make a boot partition unencrypted, at least that I know of. I do recall seeing GRUB supports the encrypted boot partition, but I haven't seen a "happy-ending" posts for those who had.

So, the partition table would look like:

| Partition Name | Type      | Size      | Purpose       | Mount Point |
| -------------- | --------- | --------- | ------------- | ----------- |
| `nvme0n1p1`    | EFI (ESP) | 500 MB    | EFI Partition | NA          |
| `nvme0n1p2`    | FAT32     | 1 GB      | Boot          | `/boot`     |
| `nvme0n1p3`    | LUKS      | As needed | Root          | `/`         |

Now, the partitioning will be different on different distro installations, but since I used Debian, the "Partion Disks" is where we do this kind of stuff.

Now when we create the encrypted physical partition for `nvme0n1p3`, it'll ask you to configure it with a passphrase. Once it's done, it'll unlock that partition and mapped to something like `nvme0n1p3_crypt`, in which you'll be able to create the underlying file system and set mount point. Again, this might vary across distributions on how to set it, but on Debian, once you click on "Configure Encrypted Partitions", it'll unlock it. After unlocking, create a `btrfs` file system on the unlocked device, and set the mount point to `/`.

Now, the partitioning has been completed, **DO NOT START INSTALL** right away. We have lot to do before we install the base sytem.

## Step 2: Log in to BusyBox

Once the partitioning has been done, and **BEFORE** installing the base system, log into the busybox by pressing **CTRL+ALT+F2**

## Step 3: Unmount current targets

When you use `lsblk`, you'll see the following mounts being set (atleast)

| Name                          | Type | Size                | MountPoints        |
| ----------------------------- | ---- | ------------------- | ------------------ |
| `/dev/nvme0n1p1`              | part | 500M                | `/target/boot/efi` |
| `/dev/nvme0n1p2`              | part | 1GB                 | `/target/boot/`    |
| `/dev/mapper/nvme0n1p3_crypt` | part | <as size specified> | `/target/`         |

Notice the last one, which is our root partition will be using a mapper, not the device itself.

Now, unmnount all `/target` points.

```bash
umount /target/boot/efi
umount /target/boot/
umount /target
```

## Step 4: Create Subvolumes

Now, let's mount our root `btrfs` volume to `/mnt`

```bash
# Cross check your generated mapper name
mount /dev/mapper/nvme0n1p3_crypt /mnt

# Set the CWD to /mnt
cd /mnt
```

If you do the `ls` on `/mnt`, it'll show only one folder which is `@rootfs`. Now, let's create separate sub volumes for our purposes.

```bash
# MAKE SURE YOU ARE INSIDE /mnt
# Rename current root fs
mv @rootfs @

# Create home directory
btrfs su cr @home

# Create root directory
btrfs su cr @root

# Create log directory
btrfs su cr @log

# Create tmp directory
btrfs su cr @tmp

# Create opt directory
btrfs su cr @opt
```

## Step 5: Mount Subvolumes to the target

We've created our base sub volumes. Now to get the installer configure the base system, we need to mount these directories to the `/target`.

```bash
# Mount root fs
mount -o noatime,compress=zstd,subvol=@ /dev/mapper/nvme0n1p3_crypt /target

# Create the folders to mount
mkdir -p /target/boot/efi
mkdir -p /target/home
mkdir -p /target/root
mkdir -p /target/var/log
mkdir -p /target/tmp
mkdir -p /target/opt

# Mount Subvolumes
mount -o noatime,compress=zstd,subvol=@home /dev/mapper/nvme0n1p3_crypt /target/home
mount -o noatime,compress=zstd,subvol=@root /dev/mapper/nvme0n1p3_crypt /target/root
mount -o noatime,compress=zstd,subvol=@log /dev/mapper/nvme0n1p3_crypt /target/var/log
mount -o noatime,compress=zstd,subvol=@tmp /dev/mapper/nvme0n1p3_crypt /target/tmp
mount -o noatime,compress=zstd,subvol=@opt /dev/mapper/nvme0n1p3_crypt /target/opt

# Mount back boot and EFI partitions
mount -o /dev/nvme0n1p2 /target/boot
mount -o /dev/nvme0n1p1 /target/boot/efi
```

## Step 6: Persist mounts via fstab

Now the mounts are set for installation. But we need to persist these mounts in fstab. Open the `fstab` using `nano`

```bash
nano /target/etc/fstab
```

Add the following mounts to `fstab`. Make sure the mapper name matches with your current set up.

```
/dev/mapper/nvme0n1p3_crypt /        btrfs noatime,compress=zstd,ssd,space_cache=v2,discard=async,subvol=@     0 0
/dev/mapper/nvme0n1p3_crypt /home    btrfs noatime,compress=zstd,ssd,space_cache=v2,discard=async,subvol=@home 0 0
/dev/mapper/nvme0n1p3_crypt /root    btrfs noatime,compress=zstd,ssd,space_cache=v2,discard=async,subvol=@root 0 0
/dev/mapper/nvme0n1p3_crypt /var/log btrfs noatime,compress=zstd,ssd,space_cache=v2,discard=async,subvol=@log  0 0
/dev/mapper/nvme0n1p3_crypt /tmp     btrfs noatime,compress=zstd,ssd,space_cache=v2,discard=async,subvol=@tmp  0 0
/dev/mapper/nvme0n1p3_crypt /opt     btrfs noatime,compress=zstd,ssd,space_cache=v2,discard=async,subvol=@opt  0 0
```

## Step 7: Finish the installation

That's it. We've created btrfs subvolumes, mounted it for installer to finish and updated fstab for persistance. Now all you have to do is to go back to the installer screen via **CTRL+ALT+F1** and finish the rest of the steps.

Once you reboot after installation, it will prompt you for LUKS password after bootloader screen.
