# Adding file disks to VMs
After creating a file disk and attaching it in the portal, login to the VM and create a partition on the newly created disk.

```dmesg | grep SCSI``` to list the attached disk.
```bash
[    0.492530] SCSI subsystem initialized
[    1.142922] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 244)
[    1.563550] sd 1:0:1:0: [sdb] Attached SCSI disk
[    1.636559] sd 0:0:0:0: [sda] Attached SCSI disk
[    1.695821] sd 3:0:0:1: [sdc] Attached SCSI disk
[   10.433476] Loading iSCSI transport class v2.0-870.
```

```sdc``` here is the newly attached disk.

Create the partition
```bash
sudo fdisk /dev/sdc
```
```n(new) -> p(primary) -> w(write)```

Write the filesystem
```bash
sudo mkfs -t ext4 /dev/sdc1
```

Create a directory to mount the disk and mount
```bash
sudo mkdir /container
sudo mount /dev/sdc1 /container
```

Check the newly mounted disk:
```bash
> df -h
udev            436M     0  436M   0% /dev
tmpfs            90M  684K   89M   1% /run
/dev/sda1        29G  5.0G   24G  18% /
tmpfs           449M     0  449M   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           449M     0  449M   0% /sys/fs/cgroup
/dev/sda15      105M  3.6M  101M   4% /boot/efi
/dev/sdb1       3.9G   16M  3.7G   1% /mnt
tmpfs            90M     0   90M   0% /run/user/1000
/dev/sdc1        63G   53M   60G   1% /blob
```

## Permenant mount (Todo)
```bash
> sudo -i blkid

# CLOUD_IMG: This file was created/modified by the Cloud Image build process
UUID=d7976e5d-71dc-4f7e-82d6-a2c292e0975e       /        ext4   defaults,discard        0 0
LABEL=UEFI      /boot/efi       vfat    defaults,discard        0 0
/dev/disk/cloud/azure_resource-part1    /mnt    auto    defaults,nofail,x-systemd.requires=cloud-init.service,comment=cloudconfig    0       2
```

```sudo cat /etc/fstab```