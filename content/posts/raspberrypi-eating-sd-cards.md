---
title: "How to stop your Raspberry Pi from eating SD Cards"
date: "2022-10-03"
tags: [raspberrypi, linux]
lang: "en"
---

# Problem

SD (or MicroSD) Cards are cheap flash storage. Their lifetime is limited by the performable write operations. If their end of life is reached, they don't work at all or read data unreliably.

A Raspberry Pi produces more IO operations than a typical digital camera (which SD Cards are conceptualized for). The load is mostly produced by writing:

* Temporary files in `/tmp/`
* Variable files in `/var/`
* The swap file in `/var/swap`
* Files in other application and user specific directories

It also depends on the usage of the Pi: e.g. compiling a big pile of software on the Pi is never a good idea in terms of SD Card lifetime.

It's possible to run a Pi only with a SD Card for a long time. One of my Pi's now runs successfully for more than 5 years that way. However, you can't tell it beforehand.

# Solution

So the solution is to reduce the IO operations performed against the SD Card. A simple attempt is to disable swapping: `sudo systemctl disable --now dphys-swapfile`. A better alternative is moving write intense directories to an external drive. There are two options:

* **Option A**: The Raspberry boots directly from a USB drive (e.g. USB thumb drive, SSD or HDD). No SD Card required.
* **Option B**: The Raspberry boots from the SD Card and mounts an external USB drive where the heavy disk IO is performed later on.

## Option A - Fresh installation

This option is pretty easy to realize, just flash the OS image (e.g. Raspberry Pi OS, formerly known as Raspbian) to the USB drive. However, it might not work with all kinds of USB drives. Also, a cheap USB thumb drive might have the same problems as a SD Card. Therefore, a durable SSD should be used instead. An old HDD is also a valid alternative. But your Pi might crash if spinning up the HDD consumes more power than can be supplied.

## Option B - External disk

This is useful if your Pi is already set up and running. The separation will split the filestorage into static files (stored on the SD Card) and dynamic files (stored on the USB drive).

You need to copy the dynamic files to the external drive. But first you should stop all running applications to prevent inconsistency when copying files.

Disable the swap:

```shell
sudo swapoff -a  # disable the swap
cat /proc/swaps  # check that no swap is active
```

Mount the external usb drive. In the example it has the Label "usb0":

```shell
ls /dev/disk/by-label/usb0  # check the device file exists
sudo mkdir -p /media/usb0  # create the mountpoint
sudo mount -o defaults /dev/disk/by-label/usb0 /media/usb0
```

Copy all directories with dynamic data (at least `/tmp` and `/var`):

```shell
sudo cp -ar /tmp/ /media/usb0/tmp
sudo cp -ar /var/ /media/usb0/var
```

> It's optional to copy `/tmp` as it only contains ephemeral data. But the directory must exist and have the correct permissions set!
> {.info }

Add to `/etc/fstab` the new mount records:

```text
/dev/disk/by-label/usb0 /media/usb0 ext4 defaults 0 0
/media/usb0/var         /var        none bind     0 0
/media/usb0/tmp         /tmp        none bind     0 0
```

The external drive will be mounted to `/media/usb0`. The directories `/tmp` and `/var` (dynamic data) will be pointed to the corresponding directories on the external device.

> The external drive is mounted with option `defaults`. If the disk is not connected or cannot be read, the raspi will not boot! As a countermeasure the options `defaults,nofail` could be used. But then the data will be written to the SD card in case of a disk failure. Inconsistent data would be the result.
> {.warning }

Now reboot the Pi: `sudo reboot`.

Check that everything is working:

```shell
touch /tmp/fs.test
ls /media/usb0/fs/tmp/fs.test  # should output the filepath
```

If the swap is not reactivated yet, run `sudo swapon /var/swap`.

