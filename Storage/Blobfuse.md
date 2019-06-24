# Mounting blob storage on VMs

Use blobfuse to mount a blob container on a VM. 


### Installation
Configure the Microsoft package repository.
```bash
wget https://packages.microsoft.com/config/ubuntu/18.04/packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
sudo apt-get update
```
Install blobfuse
```bash
sudo apt-get install blobfuse
```

### Preparing directories and disks

Now there is two ways to mount the disk, the first one is to create a ramdisk and the other one using ephemeral disks.

## Ramdisk
```bash
mkdir /mnt/ramdisk
sudo mount -t tmpfs -o size=16g tmpfs /mnt/ramdisk
sudo mkdir /mnt/ramdisk/blobfusetmp
sudo chown  /mnt/ramdisk/blobfusetmp
```

## Ephemeral disk
```bash
sudo mkdir /mnt/resource/blobfusetmp -p
sudo chown "$(whoami)" /mnt/resource/blobfusetmp
````

Create a mount directory for the blob
```bash
mkdir ~/data/container -p
```

Create a fuse_connection.cfg file in home directory with the following:
```bash
accountName myaccount
accountKey storageaccesskey
containerName mycontainer
```

and then make it accessible only to root;
```bash
chmod 600 fuse_connection.cfg
```

or assign them as environment variables;
```bash
export AZURE_STORAGE_ACCOUNT=myaccountname
export AZURE_STORAGE_ACCESS_KEY=myaccountkey
# Add flag --container-name to blobfuse if using env variables
```

### Mounting the blob
```bash
blobfuse /home/"$(whoami)"/data/container \
         --tmp-path=/blob/blobfusetmp \
         --config-file=/home/"$(whoami)"/fuse_connection.cfg \
         -o attr_timeout=240 \
         -o entry_timeout=240 \
         -o negative_timeout=120 \
         -o allow_other \
         -o ro
```

### Unmounting
```bash
fusermount -u /home/mohi/data/container
```

#### Read-only mount for others
To allow the user to mount a read-only blob for others edit ```/etc/fuse.conf``` to
include ```allow_other``` option or uncomment if applicable.