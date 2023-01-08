---
title: "LUKS: Encrypted data volumes"
date: "2022-10-01"
tags: [luks, linux]
lang: "en"
---

To realize encrypted data volumes, there are various tools for different use cases:

* **Veracrypt** works cross-platform, predefined volume sizes
* **Gocryptfs** encrypts transparent, dynamic storage usage
* **Restic** encrypts backups, network capable, can do versioning
* **LUKS** provides full disk encryption for linux systems

But LUKS can also be used for regular data volumes which behave a lot like Veracrypt volumes.

## 1. Create a volume

```shell
mkdir -p ~/volumes/  # here the encrypted volume files will be stored
sudo apt-get install cryptsetup pv
dd if=/dev/zero bs=1M count=10240 | pv | dd of=~/volumes/vol1.luks  # will create a 1MiB * 10240 = 10GiB file, called "vol1.luks"
cryptsetup -y -v luksFormat ~/volumes/vol1.luks  # here you have to enter the passphrase to protect the new volume
sudo cryptsetup luksOpen ~/volumes/vol1.luks vol1  # provide the volume as a device located at /dev/mapper/vol1
sudo cryptsetup -v status vol1
sudo mkfs.ext4 /dev/mapper/vol1  # create filesystem inside the volume
mkdir -p ~/volumes/vol1/  # the volume will also be mounted in the "volumes" directory
sudo mount /dev/mapper/vol1 ~/volumes/vol1/
sudo chown $USER:$USER -R ~/volumes/vol1/  # give current user ownership to work with the volume
df -H  # shows disk usage inside the volume
sudo umount /dev/mapper/vol1
sudo cryptsetup luksClose vol1
```

`pv` adds the progress bar for `dd` and can be omitted.

The blocksize provided to `dd` will be overwritten when creating a filesystem inside the container. So it has no impact on the performance later on.

The command "luksFormat" will figure out good encryption parameters per default. But this requires a decent up-to-date version.

## 2. Open the volume

```shell
sudo cryptsetup luksOpen ~/volumes/vol1.luks vol1
sudo mount /dev/mapper/vol1 ~/volumes/vol1/
```

## 3. Close the volume

```shell
sudo umount /dev/mapper/vol1
sudo cryptsetup luksClose vol1
```
