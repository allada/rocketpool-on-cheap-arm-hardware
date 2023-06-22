# Run a rocketpool node on low power hardware (ARM)
Here we are going to be going over how to run a rocketpool node on low power hardware. This article serves a couple purposes:
* A reference guide for myself and others to use to recreate the setup.
* More material online for people to find when searching for how to do similar things.
* A starting point for others to build off of.

I choose the [Rock5b](https://radxa.com/product/detailed?productName=rock5b) hardware, but most of these steps should be the same on other ARM hardware. Another good alternative is the [Orange Pi 5 Plus](http://www.orangepi.org/html/hardWare/computerAndMicrocontrollers/details/Orange-Pi-5-plus.html). There will be things in this article specific to the Rock5B, so skip over them if you are using other hardware.

In this article you will need:
* SD card (I used 8GB)
* M.2 NVMe SSD (I suggest at least 2TB for future proofing)
* Rock5b (or other ARM hardware)

# Rock5b setup
Here we will be going over the step-by-step process of getting the Rock5b board setup. At the time of writing this, the device is shipped with firmware that must be upgraded in order to run an Ethereum node on the hardware.

## Download and flash Ubuntu to SD card
Download the Rock5b ubuntu 20.04 image and burn it to an SD card from another computer and check it's sha256:

```sh
wget https://github.com/radxa/debos-radxa/releases/download/20221031-1045/rock-5b-ubuntu-focal-server-arm64-20221031-1328-gpt.img.xz
sha256sum rock-5b-ubuntu-focal-server-arm64-20221031-1328-gpt.img.xz
```

The results should look something like:
```
88346bd656695757c0ca5e266570626a822295f9d3962a2afe51710048410c4a  rock-5b-ubuntu-focal-server-arm64-20221031-1328-gpt.img.xz
```

Now find your SD card device using `lsblk` and note the name. Then burn the image to the SD using:
```sh
xzcat -d rock-5b-ubuntu-focal-server-arm64-20221031-1328-gpt.img.xz | sudo dd of=/dev/{sd_card_block_device_here} bs=1M status=progress
```

Normally we'd verify the image, but since we only need to get the device to boot, we don't need to.

## Flash SPI on Rock5b
Now put the SD card into the Rock 5b along with the NVMe and boot it up. It may take 1-2 mins for the first boot.

> **Note**
> The username and password is `rock`.

Switch to root user:

```sh
sudo su root
```

Now on to flashing the SPI (the special on-board block device used to know where to boot and such). This is needed because
at the time of writing this, the default SPI does not support ability to boot from NVMe, the new one does. Run the following on the Rock5b as `root`:
```sh
# Update our certs for security.
apt update
DEBIAN_FRONTEND=noninteractive apt install ca-certificates

# Download and extract our files.
wget -O - https://dl.radxa.com/rock5/sw/images/others/zero.img.gz | gzip -d > zero.img
wget https://dl.radxa.com/rock5/sw/images/loader/rock-5b/release/rock-5b-spi-image-g49da44e116d.img
sha256sum zero.img rock-5b-spi-image-g49da44e116d.img
```

> **Warning**
> DO NOT SKIP THE SHA256 CHECK OR YOU COULD BRICK YOUR DEVICE.

The previous command should result in:
```
080acf35a507ac9849cfcba47dc2ad83e01b75663a516279c8b9d243b719643e  zero.img
503a01030d06d4a8d127653da1820d0a2ed2fbd29617328c6da44335c3ec9f0b  rock-5b-spi-image-g49da44e116d.img
```

Make sure the SPI flash is available:
```sh
ls /dev/mtdblock*
```

It should report back: `/dev/mtdblock0`

> **Warning**
> The next few steps could brick your device if done incorrectly. Make sure you have stable power and are not going to lose power during this process.

Now we are going to clear the SPI flash with our `zero.img` (this may take a few mins):
```sh
dd if=zero.img of=/dev/mtdblock0
sha256sum /dev/mtdblock0 zero.img
```

It should result in:
```
080acf35a507ac9849cfcba47dc2ad83e01b75663a516279c8b9d243b719643e  zero.img
080acf35a507ac9849cfcba47dc2ad83e01b75663a516279c8b9d243b719643e  /dev/mtdblock0
```

Now actually flash our image:
```sh
dd if=rock-5b-spi-image-g49da44e116d.img of=/dev/mtdblock0
sync
sha256sum /dev/mtdblock0 rock-5b-spi-image-g49da44e116d.img
```

It should result in:
```
503a01030d06d4a8d127653da1820d0a2ed2fbd29617328c6da44335c3ec9f0b  zero.img
503a01030d06d4a8d127653da1820d0a2ed2fbd29617328c6da44335c3ec9f0b  /dev/mtdblock0
```

## Install Ubuntu on NVMe
First we'll download the same image we did before to the Rock 5b or transfer it, assume commands from here on are on the Rock 5b.

> **Note**
> All commands in this section are assumed to be ran as `root` and on the Rock5b.

Find your NVMe device using `lsblk` and note the name, we'll assume it's `nvme0n1` and we'll download and write the image to it. We stream it directly instead of writing it first because SD cards are often super slow. This is much faster in most cases:
```sh
wget -O - https://github.com/radxa/debos-radxa/releases/download/20221031-1045/rock-5b-ubuntu-focal-server-arm64-20221031-1328-gpt.img.xz | xzcat -d | dd of=/dev/nvme0n1 bs=1M
sync
head -c 2000000000 /dev/nvme0n1 | sha256sum
```

The results should look something like:
```
7010dd100d4139a5a9d595b55ae327fc71f6454997b2f7c025090c10d15c79f0  -
```

### Reconfigure partition table
By default the boot image will use the entire disk. It is generally a good idea to make a boot partition and a `data` partition. Here we will be resizing our boot partition to only use the first 16G of the NVMe and leave the rest unused (we'll set up the unused part later):

```sh
e2fsck -f /dev/nvme0n1p2       # Check disk for errors.
resize2fs /dev/nvme0n1p2 16G   # Move all files and inodes of filesystem to first 16G.
sgdisk /dev/nvme0n1 -e         # Move secondary header.
parted --script /dev/nvme0n1 \
    resizepart 2 16G \
    mkpart primary 16G 100%    # Resize the second (root) partition to ~16G and then make new partition for remaining.
lsblk
```

This should result in something like:
```
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
mtdblock0    31:0    0   16M  0 disk 
mmcblk0     179:0    0  7.4G  0 disk 
├─mmcblk0p1 179:1    0  512M  0 part /boot
└─mmcblk0p2 179:2    0  6.9G  0 part /
nvme0n1     259:0    0  1.8T  0 disk 
├─nvme0n1p1 259:4    0  512M  0 part 
├─nvme0n1p2 259:5    0 14.4G  0 part 
└─nvme0n1p3 259:6    0  1.8T  0 part
```

Now we need to boot into the NVMe's OS.

> **Note**
> We are still booted into the SD card, so we need to shutdown, unplug the SD card and then boot.

```sh
sync
shutdown now
```

Now wait a few mins (~2-5 mins). Sadly the Rock5b does not have a good way to tell when the device is off, you just need to watch for the blue and green light on the front to stop flickering or just wait a few mins.

Now unplug the SD Card once the device is off and unplugged. Then plug the device back in.

### Verify it boots to the NVMe

When the device starts up run:
```sh
lsblk
```

And should look like:
```
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
mtdblock0    31:0    0   16M  0 disk 
nvme0n1     259:0    0  1.8T  0 disk 
├─nvme0n1p1 259:1    0  512M  0 part 
├─nvme0n1p2 259:2    0 14.4G  0 part /
└─nvme0n1p3 259:3    0  1.8T  0 part 
```

## Compile & install new kernel (Rock5b)
> **Note**
> This is specifically needed for `erigon`, but we are now using `geth`. Geth likely does not need this to be done, but it is still highly reccomended.

We will assume all commands from here on are ran as `root` and on the Rock5b:
```sh
sudo su root
```

Install packages required to compile our kernel:
```sh
apt update
DEBIAN_FRONTEND=noninteractive apt install -y git  device-tree-compiler libncurses5 libncurses5-dev build-essential libssl-dev mtools bc python dosfstools bison flex rsync u-boot-tools make
```

Download the kernel packages specific to the Rock5b:
```sh
mkdir ~/rk3588-sdk
cd ~/rk3588-sdk

# Download our files.
wget https://github.com/radxa/kernel/archive/refs/tags/5.10.66-20-rockchip.tar.gz
wget https://github.com/radxa/rkbin/archive/d6aad64d4874b416f25669748a9ae5592642a453.tar.gz
wget https://github.com/radxa/build/archive/223bcb503769e862a05ebe08c2d49348c050e732.tar.gz

# Ensure our hashes match.
sha256sum 5.10.66-20-rockchip.tar.gz d6aad64d4874b416f25669748a9ae5592642a453.tar.gz 223bcb503769e862a05ebe08c2d49348c050e732.tar.gz

# Extract files/folders.
tar xzf 5.10.66-20-rockchip.tar.gz
tar xzf d6aad64d4874b416f25669748a9ae5592642a453.tar.gz
tar xzf 223bcb503769e862a05ebe08c2d49348c050e732.tar.gz

# Remove unneeded files.
rm 5.10.66-20-rockchip.tar.gz d6aad64d4874b416f25669748a9ae5592642a453.tar.gz 223bcb503769e862a05ebe08c2d49348c050e732.tar.gz

# Cleanup to easier to deal with locations.
mv kernel-5.10.66-20-rockchip kernel
mv build-223bcb503769e862a05ebe08c2d49348c050e732 build
mv rkbin-d6aad64d4874b416f25669748a9ae5592642a453 rk3588-sdk
```

Should result with:
```
658a6447bf9ee0f422b318188551dd93dac1cd1e0de35f6dd79aa07b2e3f5eee  5.10.66-20-rockchip.tar.gz
65ebb49f8f7a1d1e2ca8a4d83abc6b0861ada3a53fda715d848a2a7843ae5dde  d6aad64d4874b416f25669748a9ae5592642a453.tar.gz
50602d76fff00e334b40be059aea0859b9de10fc53811e22fa2a605793cb4f2e  223bcb503769e862a05ebe08c2d49348c050e732.tar.gz
```

Change some of the configurations of the kernel. Specifically we need to:
* Enable 64k page memory support (for `erigon` and just a good idea to have).
* Support larger Virutal Address (VA) space (for `erigon` and just a good idea to have).
* Disable NTFS Filesystem (required because it appears to be broken on this kernel + architecture combo).
* Add patch fixing a bug with bit shifting using wrong value.

Now it's build time! This will take a very long time (~1-2 hours):
```
# Make a few minor patches to the kernel config.
# In order for erigon to work it needs 64k page memory support.
cd ~/rk3588-sdk/kernel
make rockchip_linux_defconfig
patch .config <<'EOT'
--- .config-bak 2023-05-11 02:52:40.262561826 +0000
+++ .config 2023-05-11 02:55:59.956912407 +0000
@@ -249,12 +249,12 @@
 CONFIG_ARM64=y
 CONFIG_64BIT=y
 CONFIG_MMU=y
-CONFIG_ARM64_PAGE_SHIFT=12
-CONFIG_ARM64_CONT_PTE_SHIFT=4
-CONFIG_ARM64_CONT_PMD_SHIFT=4
-CONFIG_ARCH_MMAP_RND_BITS_MIN=18
-CONFIG_ARCH_MMAP_RND_BITS_MAX=24
-CONFIG_ARCH_MMAP_RND_COMPAT_BITS_MIN=11
+CONFIG_ARM64_PAGE_SHIFT=16
+CONFIG_ARM64_CONT_PTE_SHIFT=5
+CONFIG_ARM64_CONT_PMD_SHIFT=5
+CONFIG_ARCH_MMAP_RND_BITS_MIN=14
+CONFIG_ARCH_MMAP_RND_BITS_MAX=29
+CONFIG_ARCH_MMAP_RND_COMPAT_BITS_MIN=7
 CONFIG_ARCH_MMAP_RND_COMPAT_BITS_MAX=16
 CONFIG_STACKTRACE_SUPPORT=y
 CONFIG_ILLEGAL_POINTER_VALUE=0xdead000000000000
@@ -361,10 +361,12 @@
 CONFIG_ARM64_4K_PAGES=y
 # CONFIG_ARM64_16K_PAGES is not set
 # CONFIG_ARM64_64K_PAGES is not set
-CONFIG_ARM64_VA_BITS_39=y
-# CONFIG_ARM64_VA_BITS_48 is not set
-CONFIG_ARM64_VA_BITS=39
+# CONFIG_ARM64_VA_BITS_42 is not set
+CONFIG_ARM64_VA_BITS_48=y
+# CONFIG_ARM64_VA_BITS_52 is not set
+CONFIG_ARM64_VA_BITS=48
 CONFIG_ARM64_PA_BITS_48=y
+# CONFIG_ARM64_PA_BITS_52 is not set
 CONFIG_ARM64_PA_BITS=48
 # CONFIG_CPU_BIG_ENDIAN is not set
 CONFIG_CPU_LITTLE_ENDIAN=y
@@ -388,7 +390,6 @@
 CONFIG_HAVE_ARCH_PFN_VALID=y
 CONFIG_HW_PERF_EVENTS=y
 CONFIG_SYS_SUPPORTS_HUGETLBFS=y
-CONFIG_ARCH_WANT_HUGE_PMD_SHARE=y
 CONFIG_ARCH_HAS_CACHE_LINE_SIZE=y
 CONFIG_ARCH_ENABLE_SPLIT_PMD_PTLOCK=y
 # CONFIG_PARAVIRT is not set
@@ -2678,7 +2679,6 @@
 
 # CONFIG_WAN is not set
 # CONFIG_IEEE802154_DRIVERS is not set
-# CONFIG_VMXNET3 is not set
 # CONFIG_NETDEVSIM is not set
 # CONFIG_NET_FAILOVER is not set
 # CONFIG_ISDN is not set
@@ -6953,9 +6953,7 @@
 CONFIG_FAT_DEFAULT_IOCHARSET="utf8"
 # CONFIG_FAT_DEFAULT_UTF8 is not set
 # CONFIG_EXFAT_FS is not set
-CONFIG_NTFS_FS=y
-# CONFIG_NTFS_DEBUG is not set
-CONFIG_NTFS_RW=y
+# CONFIG_NTFS_FS is not set
 # end of DOS/FAT/EXFAT/NT Filesystems
 
 #
EOT

# There was a bug with ARM when the page size was changed due to a hard codded bit shift.
# Future kernel version were patched.
# See: https://lore.kernel.org/buildroot/87bknwptfd.fsf@dell.be.48ers.dk/T/#iZ2e.:..:20221211152153.1833386-1-giulio.benetti::40benettiengineering.com:1ernel.h-fix-build-failure-on-aarch64.patch
patch drivers/gpu/arm/midgard/mali_base_kernel.h <<'EOT'
297,301c297,301
< #define BASEP_MEM_INVALID_HANDLE               (0ull  << 12)
< #define BASE_MEM_MMU_DUMP_HANDLE               (1ull  << 12)
< #define BASE_MEM_TRACE_BUFFER_HANDLE           (2ull  << 12)
< #define BASE_MEM_MAP_TRACKING_HANDLE           (3ull  << 12)
< #define BASEP_MEM_WRITE_ALLOC_PAGES_HANDLE     (4ull  << 12)
---
> #define BASEP_MEM_INVALID_HANDLE               (0ull  << PAGE_SHIFT)
> #define BASE_MEM_MMU_DUMP_HANDLE               (1ull  << PAGE_SHIFT)
> #define BASE_MEM_TRACE_BUFFER_HANDLE           (2ull  << PAGE_SHIFT)
> #define BASE_MEM_MAP_TRACKING_HANDLE           (3ull  << PAGE_SHIFT)
> #define BASEP_MEM_WRITE_ALLOC_PAGES_HANDLE     (4ull  << PAGE_SHIFT)
303,304c303,304
< #define BASE_MEM_COOKIE_BASE                   (64ul  << 12)
< #define BASE_MEM_FIRST_FREE_ADDRESS            ((BITS_PER_LONG << 12) + \
---
> #define BASE_MEM_COOKIE_BASE                   (64ul  << PAGE_SHIFT)
> #define BASE_MEM_FIRST_FREE_ADDRESS            ((BITS_PER_LONG << PAGE_SHIFT) + \
EOT
make savedefconfig
cp defconfig arch/arm64/configs/rockchip_linux_defconfig

# Now compile our kernel and build them to a .deb file.
cd ~/rk3588-sdk
mkdir -p out
```

### Compile the kernel
> **Note**
> This will take 1-2 hours.

```sh
./build/pack-kernel.sh -d rockchip_linux_defconfig -r 99
```

### Install the kernel
```sh
dpkg -i out/packages/linux-headers-5.10.66-99-rockchip-g_5.10.66-99-rockchip_arm64.deb
dpkg -i out/packages/linux-image-5.10.66-99-rockchip-g_5.10.66-99-rockchip_arm64.deb
dpkg -i out/packages/linux-libc-dev_5.10.66-99-rockchip_arm64.deb
```

Now reboot.

### Verify the kernel
> **Note**
> Remember to login as `root`.

```sh
uname -a
```

It should the new kernel you just installed:
```
Linux rock-5b 5.10.66-99-rockchip-g #rockchip SMP Tue May 2 03:05:04 UTC 2023 aarch64 aarch64 aarch64 GNU/Linux
```

### Cleanup needless build files
```sh
rm -rf ~/rk3588-sdk
```

## Install ZFS

ZFS filesystem is choosen because it makes it very easy to:
* Keep data separate (ie: partition data based on use)
* Backup data (ie: snapshot and send to another drive)
* Expand data (ie: add more drives to a pool)

However, it comes at a small penalty of about 2-10% performance loss on EVM workloads. The tradeoffs are often worth it because of the ease of use and ability to quickly make backups and do other maintance.


### Install ZFS compiletime dependencies
```sh
DEBIAN_FRONTEND=noninteractive apt install -y build-essential autoconf automake libtool gawk alien fakeroot dkms libblkid-dev uuid-dev libudev-dev libssl-dev zlib1g-dev libaio-dev libattr1-dev libelf-dev linux-headers-generic python3 python3-dev python3-setuptools python3-cffi libffi-dev python3-packaging git libcurl4-openssl-dev debhelper-compat dh-python po-debconf python3-all-dev python3-sphinx
wget https://github.com/openzfs/zfs/archive/refs/tags/zfs-2.1.11.tar.gz
sha256sum zfs-2.1.11.tar.gz
```

Should result with:
```
3740435916157b62f589406e2f24c07dda80570bae456922f1d497318c50f4c9  zfs-2.1.11.tar.gz
```

### Extract and build ZFS
```sh
tar xzf zfs-2.1.11.tar.gz
rm -rf zfs-2.1.11.tar.gz
cd zfs-zfs-2.1.11
sh autogen.sh
./configure
make -s deb -j$(nproc)
dpkg -i libzfs5_2.1.11-1_arm64.deb
dpkg -i libzpool5_2.1.11-1_arm64.deb
dpkg -i libnvpair3_2.1.11-1_arm64.deb
dpkg -i libuutil3_2.1.11-1_arm64.deb
dpkg -i kmod-zfs-5.10.66-99-rockchip-g_2.1.11-1_arm64.deb
dpkg -i zfs_2.1.11-1_arm64.deb
modprobe zfs
```

### Cleanup unused files
```sh
cd ~/
rm -rf ~/zfs-zfs-2.1.11
DEBIAN_FRONTEND=noninteractive apt remove -y git device-tree-compiler libncurses5 libncurses5-dev build-essential libssl-dev mtools bc python dosfstools bison flex rsync u-boot-tools make build-essential autoconf automake libtool gawk alien fakeroot dkms libblkid-dev uuid-dev libudev-dev libssl-dev zlib1g-dev libaio-dev libattr1-dev libelf-dev linux-headers-generic python3 python3-dev python3-setuptools python3-cffi libffi-dev python3-packaging git libcurl4-openssl-dev debhelper-compat dh-python po-debconf python3-all-dev python3-sphinx
apt autoremove -y
```

### Verify ZFS is working
```sh
zfs list
```

## Configure NVMe on ZFS
```sh
zpool create tank \
  -m none \
  -o ashift=12 \
  -o autotrim=on \
  -O compression=lz4 \
  -O dedup=off \
  -O xattr=sa \
  -O acltype=posixacl \
  -O atime=off \
  -O sync=standard \
  nvme0n1p3
```


# Configure the system
> **Note**
> Remember to run commands as `root`.

```sh
# Remove the apt package that came with the Rock5b image.
rm /etc/apt/sources.list.d/apt-radxa-com.list
apt update
DEBIAN_FRONTEND=noninteractive apt upgrade -y
DEBIAN_FRONTEND=noninteractive apt dist-upgrade -y
```

## Install docker using docker's repo
```sh
DEBIAN_FRONTEND=noninteractive apt install -y lsb-release ca-certificates
# Add docker gpg keys so apt can use docker's repo.
mkdir -p /etc/apt/keyrings
curl -fsSL "https://download.docker.com/linux/ubuntu/gpg" | $SUDO_CMD gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
# Install newest docker versions from docker's package repo.
DEBIAN_FRONTEND=noninteractive apt update
DEBIAN_FRONTEND=noninteractive apt install -y docker-ce docker-ce-cli docker-compose-plugin containerd.io
```

## Install & Configure rocketpool
Download rocketpool cli:
```sh
wget https://github.com/rocket-pool/smartnode-install/releases/latest/download/rocketpool-cli-linux-$(uname -m | sed s/aarch64/arm64/ | sed s/x86_64/amd64/) -O /usr/bin/rocketpool
chmod +x /usr/bin/rocketpool
sha256sum /usr/bin/rocketpool
```

### Install Rocketpool dependencies
```sh
sudo -u rock -g rock rocketpool service install -y
```

### Configure Rocketpool
```sh
sudo -u rock -g rock rocketpool service config
```

Settings that need to be set:
```
2/9 Network = "Ethereum Mainnet"
3/9 Client Mode = "Locally Managed"
4/9 Execution Client > Selection = "Geth"
5/9 Consensus Client > Selection = "Nimbus"
...you choose next options...
5/9 Consensus Client > Checkpoint Sync
  - 1. Go to https://sync.invis.tools/ and copy the latest checkpoint
  - 2. Use another checkpoint sync service
  - 3. [Very slow] Sync from genesis (this can take months).
...you choose next options...
6/9 Use Fallback Clients = "No" (unless you have additional EL clients)
7/9 Metrics = "Yes"
...you choose next options...
...save and exit...
```

Sadly we need to change a couple more settings that are not in the wizard:
```sh
sudo -u rock -g rock rocketpool service config
```

```
Execution Client (ETH1)
  - Check "Use Pebble DB" (this db is shown to be a bit faster than leveldb)
Consensus Layer (ETH2)
  - Change "Pruning Mode" -> "Pruned" (This prevents nimbus from using too much disk space)
```

### Configure Rocketpool to use ZFS
Sadly docker does not support setting bind propagation as a volume, so we override it manually. This will propgate child mounts into the docker container.
```sh
# Create ZFS mountpoints for nimbus and geth.
zfs create -o mountpoint=/geth/data/geth/ tank/geth_data
zfs create -o mountpoint=/geth/data/geth/chaindata/ tank/geth_data/chaindata
zfs create -o mountpoint=/geth/data/geth/chaindata/ancient/ tank/geth_data/chaindata/ancient
zfs create -o mountpoint=/nimbus/data/nimbus/ tank/nimbus_data

# Configure rocketpool to use ZFS mount points.
cat > /home/rock/.rocketpool/override/eth1.yml <<'EOT'
services:
  eth1:
    volumes:
      - /geth/data/:/ethclient
EOT
cat > /home/rock/.rocketpool/override/eth2.yml <<'EOT'
services:
  eth2:
    volumes:
      - /nimbus/data/:/ethclient
EOT
```

# Final system configuration

## Bootup script
```sh
cat <<'EOT' > /root/start_rocketpool_node.sh
#!/bin/bash
set -eux

# Give a second for the nvme drives to come online if needed.
sleep 3

# Force import zfs datasets.
zpool import tank -F

# Sleep just in case import takes a second to be ready.
sleep 3

# Tell zfs driver to cache only 512MB in ram, because zfs stores stuff in ram using mmap.
echo 1036870912 > /sys/module/zfs/parameters/zfs_arc_max

zfs destroy tank/swap || true
zfs create -s -V 32G -b $(getconf PAGESIZE) \
    -o compression=zle \
    -o sync=always \
    -o primarycache=metadata \
    -o secondarycache=none \
    tank/swap
sleep 3 # It takes a moment for our zvol to be created.
mkswap -f /dev/zvol/tank/swap
swapon /dev/zvol/tank/swap

docker stop rocketpool_eth1 rocketpool_eth2 || true

sleep 1

# We do this 3 times because the order which they get mounted is random and this ensures we get them all.
umount /geth/data/geth /geth/data/geth/chaindata/ancient /geth/data/geth/chaindata || true
umount /geth/data/geth /geth/data/geth/chaindata/ancient /geth/data/geth/chaindata || true
umount /geth/data/geth /geth/data/geth/chaindata/ancient /geth/data/geth/chaindata || true
umount /nimbus/data/nimbus || true

# Give a sec for anything to catchup if needed.
sleep 1

# Clean our folders in prep for them being mounted.
rm -rf /nimbus/data/nimbus/* || true
rm -rf /geth/data/geth/* || true

# Mount our zfs datasets.
zfs mount tank/geth_data
zfs mount tank/geth_data/chaindata
zfs mount tank/geth_data/chaindata/ancient
zfs mount tank/nimbus_data

# Give a sec for any mounting to be performed.
sleep 1

# Start rocketpool services.
sudo -u rock rocketpool service start -y
EOT

chmod +x /root/start_rocketpool_node.sh
echo "@reboot root sh /root/start_rocketpool_node.sh" >> /etc/crontab
```

## Add log cleanup cron
```sh
echo "* 0,12 * * * root journalctl --vacuum-size=100M" >> /etc/crontab
```

# Install check health scripts
This health check script will check if the node is healthy and if not it will attempt to restart the relevant services. The big one here is on startup `rocketpool_validator` sometimes does not pickup on the fact that `geth` and `nimbus` are actually caught up, so it will restart it in such case.

This also setsup a service to run the health check every minute. We use a service instead of a cron so we get logging with pruning.

```sh
cat <<'EOT' > /root/check_health.sh
#!/bin/bash
set -xeuo pipefail

function get_el_seconds_behind() {
  hex_timestamp=$(docker exec rocketpool_eth1 wget -O - -q --header "Content-Type: application/json" --post-data '{"jsonrpc":"2.0","method":"eth_getBlockByNumber","params":["latest", false],"id":1}' 127.0.0.1:8545 \
      | jq -r '.result.timestamp' \
      | cut -c3-)
  current_timestamp=$(echo "ibase=16; ${hex_timestamp^^}" | bc)
  echo $(date +"%s") - $current_timestamp  | bc
}

touch /tmp/rocketpool_health_check.lock
{
  # This will ensure we only ever have 1 of these scripts running on a single machine.
  if ! flock --nonblock -x 3 ; then
    echo "Script is already running."
    exit 1
  fi

  # Sometimes the logs get currupted, in such case clear them out and wait to try again.
  if ! docker logs rocketpool_validator --since 3m >/dev/null 2>&1 ; then
    cat /dev/null > $(docker inspect --format='{{.LogPath}}' rocketpool_validator)
    sleep 3m
  fi

  if ! (docker logs rocketpool_validator --since 3m | grep 'node_status=synced') ; then
    # It looks like sometimes this does not work as expected, so sleep for a min
    # then try again to ensure we don't kill it needlessly.
    sleep 1m
    if ! (docker logs rocketpool_validator --since 3m | grep 'node_status=synced') ; then
      docker restart rocketpool_validator
      sleep 6m
    fi
  fi

  # If we are more than 180 seconds behind and our uptime is less than an hour then restart eth1
  # and maybe eth2.
  if [ "$(get_el_seconds_behind)" -gt "180" ] && [ "$(awk '{print int($1)}' /proc/uptime)" -gt "3600" ]; then
    docker restart rocketpool_eth1

    # We don't want to kill nimbus if possible, so sleep for 15 mins then check if geth caught up.
    # If geth has not caught up in 15 mins, nimbus could be the problem, so we'll restart nimbus.
    sleep 900s
    if [ "$(get_el_seconds_behind)" -gt "180" ]; then
      docker restart rocketpool_eth2
    fi
    sleep 3600s # Sleep for 1 hour to ensure we give time for geth and nimbus to catch up.
  fi
} 3</tmp/rocketpool_health_check.lock
EOT
cat <<'EOT' > /etc/systemd/system/validator-health-check.service
[Unit]
Description=Checks health of validator node and restarts services as needed
Wants=validator-health-check.timer

[Service]
Type=oneshot
User=root
ExecStart=bash /root/check_health.sh

[Install]
WantedBy=multi-user.target
EOT
cat <<'EOT' > /etc/systemd/system/validator-health-check.timer
[Unit]
Description=Checks health of validator node and restarts services as needed
Requires=validator-health-check.service

[Timer]
Unit=validator-health-check.service
OnCalendar=*-*-* *:*:00

[Install]
WantedBy=timers.target
EOT
systemctl daemon-reload
systemctl enable validator-health-check
systemctl start validator-health-check
```
