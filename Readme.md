# Guide

## Create Server

### Add Volume

![Choose Volume](images/01-choose-volume.png)

## Install own image

### Mount

![Choose Volume](images/02-choose-image.png)

### Open console and restart

### Choose language, country and keyboard layout

![Choose language, country and keyboard layout](images/03-install-part1.png)

### Skip network config

![Skip network config](images/04-install-part2.png)
![Skip network config](images/05-install-part3.png)

### Setup encrypted lvm

![Setup encrypted partition](images/06-install-part4.png)
![Setup encrypted partition](images/07-install-part5.png)
![Setup encrypted partition](images/08-install-part6.png)

### Rest of installation

![Rest of installation](images/09-install-part7.png)
![Rest of installation](images/10-install-part8.png)
![Rest of installation](images/11-install-part9.png)
![Rest of installation](images/12-install-part10.png)

At this point unmount the image in the cloud control panel and then hit enter in the terminal.

### Enter passphrase on boot

![Rest of installation](images/13-enter-passphrase.png)

## Setup networking

 Edit `/etc/netplan/01-netcfg.yaml` and add the following part:

    network:
      version: 2
      renderer: networkd
      ethernets:
        ens3:
          addresses:
            - <ip6-address-copied-from-panel>::1/64
          dhcp4: true
          gateway6: fe80::1

After saving apply the netplan:

    netplan apply

Now try to ping an ip outside of your box:

    ping 1.1.1.1

## Encrypt volume and setup file system

Set volume as encrypted:

    cryptsetup -y -v luksFormat /dev/sdb

![Encrypt volume](images/15-encrypt-volume.png)

Create random key for your volume and set proper permissions:

    dd if=/dev/random of=/etc/volume-secret-key bs=512 count=8
    chmod 0600 /etc/volume-secret-key

![Encrypt volume](images/15-encrypt-volume-1.png)

Add your key file to your volume as an ecryption key:

    cryptsetup -v luksAddKey /dev/sdb /etc/volume-secret-key

![Encrypt volume](images/15-encrypt-volume-2.png)

To be able to automatically mount your encrypted volume you first need to get the UUID of your volume:

    cryptsetup luksDump /dev/sdb | grep "UUID"

![Encrypt volume](images/15-encrypt-volume-3.png)

Now you need to edit `/etc/crypttab` and add the following entry for your volume:

    volume UUID=3530a1c8-9de4-4c68-b674-9d9e88ab6e0d /etc/volume-secret-key luks

After doing that you are already able to start the encrypted volume:

    cryptdisks_start volume

![Encrypt volume](images/15-encrypt-volume-4.png)

Install `pv` to see progress on the next step when clearing the volume:

    apt-get update && apt-get install pv

Then clear the volume (this might take a few minutes depending on the size of the volume):

    pv -tpreb /dev/zero | dd of=/dev/mapper/volume bs=128M

![Clear volume](images/16-clear-volume.png)

Now you can create the file system on top of the encrypted volume:

    mkfs.ext4 /dev/mapper/sdb_crypt

![Filesystem](images/17-create-filesystem.png)

After that create a mount folder for the volume ...:

    mkdir /mnt/volume

...and add the following line to `/etc/fstab`:

    /dev/mapper/volume  /mnt/volume ext4    defaults    0   2

![Filesystem](images/17-create-filesystem-1.png)

Now mount the encrypted volume with a swift:

    mount -a

## Installing Nextcloud

Install nextcloud and enable removable media:

    snap install nextcloud
    snap enable nextcloud
    snap connect nextcloud:removable-media

Create a new data folder on your volume:

    mkdir /mnt/volume/nextcloud

Update the config for nextcloud to use that new folder:

    nano /var/snap/nextcloud/current/nextcloud/config/config.php

![data directory](images/18-set-data-directory.png)

Disable nextcloud again and move over all files:

    snap disable nextcloud
    mv /var/snap/nextcloud/common/nextcloud/data/* /mnt/volume/nextcloud

For further configuration visit <https://manandkeyboard.tk/2018/01/07/nextcloud-snap-installation/>

After that you can open up your browser and navigate to your server's ip:

![setup](images/19-setup-nextcloud.png)