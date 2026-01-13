---
description: >-
  How I adapted the hybrid encryption described here to work with CachyOS
  installed in ZFS.
---

# Hybrid Encryption With CachyOS

## Introduction&#x20;

I've decided to try CachyOS. It's nice that it was willing to install in an encrypted zfs. This will make some things easier in setting up the hybrid encryption unlocking, but since CachyOS uses a different system to make the initrd, there will need to be some changes too. &#x20;

## Install the OS

Just do the normal install, but with manual partitioning. Create these partitions:

<table><thead><tr><th width="49">#</th><th width="116">Label</th><th width="114">Size</th><th>Description</th></tr></thead><tbody><tr><td>1</td><td>EFI</td><td>2G</td><td>CachyOS defaults to using 2GB for the EFI partition.  I think this is because they use the EFI partition as /boot</td></tr><tr><td>2</td><td>root-key</td><td>17MB</td><td>Leave unformatted for now, will be LUKS for unlocking zfs</td></tr><tr><td>3</td><td>crypt-swap</td><td>32GB</td><td>Leave unformatted for now.  At least size of RAM (if you want hibernate).  This will be encrypted swap.</td></tr><tr><td>4</td><td>zfsroot1</td><td>remainder of disk</td><td>The rest of the disk.  Select zfs, use as "/" and turn on encryption with a temporary password.  </td></tr></tbody></table>

When the install is done, but before rebooting, make a zfs snapshot&#x20;

```
sudo zfs snapshot -r zpcachyos@hybrid1
```

If you have data you want to load (like your /home from an old installation), now is a good time to zfs receive it. &#x20;

Then reboot into the system and make sure everything is working ok.  You can use the machine now like normal, and continue these steps whenever you are ready.  For now you need to put in the temporary password for the zfs on boot. &#x20;

## Label the partitions

The unformatted partitions didn't get names, but the names do make some things a little easier.  So lets use parted to set the names.  I leave it to you to know your disk names and partition numbers.  For this example, we will name the root-key partition on /dev/nvme3n1p2

```
parted /dev/nvme3n1
name 2 root-key
quit
```

## Create the key&#x20;

Now we need to create the key that we will be using to unlock the zfs. &#x20;

```
tr -d ‘\000’ < /dev/urandom | dd bs=32 count=1 of=/root-key
```

## Make the root-key LUKS space&#x20;

Now we will create the LUKS volume in the root-key partition.  We will also add a backup password now so we have 2 working passwords and are less likely to lose access by making some mistake.  Then we will copy our root-key into this LUKS volume. &#x20;

```
cryptsetup luksFormat /dev/disk/by-partlabel/root-key
cryptsetup luksAddKey /dev/disk/by-partlabel/root-key
cryptsetup open /dev/disk/by-partlabel/root-key root-key
cat /root-key > /dev/mapper/root-key
```

## Make the encrypted swap space

Now we will create the encrypted swap space.  We will use the key that we generated above to unlock the swap so we don't need to type in another password. &#x20;

```
cryptsetup luksFormat /dev/disk/by-partlabel/swap-crypt /root-key
cryptsetup open /dev/disk/by-partlabel/swap-crypt swap-crypt -d /root-key
mkswap /dev/mapper/swap-crypt
blkid /dev/disk/by-partlabel/swap-crypt
```

This should show the info for that partition, like:

```
/dev/disk/by-partlabel/swap-crypt: UUID="198c9ec0-274f-40a9-a104-e085f2fdee08" TYPE="crypto_LUKS" PARTLABEL="swap-crypt" PARTUUID="ff96b556-9dfb-41d0-a88a-e64e583c795e"
```

## Make the script to go in the initrd&#x20;

CachyOS uses mkinitcpio to create the inird. This works a bit differently from other initrd systems and needs its own special setup.  This needs the script to run during initrd, a script to "install" that one when the initrd is being installed.  Lets start with that install script

```
cd /etc/initcpio/install
emacs rootkey
```

Paste in:

```
#!/usr/bin/env bash
build() { 
    add_runscript
}

help() { 
    cat <<EOF My own little extension to unlock zfs and swap using a key in a 
    tiny LUKS volume.
EOF 
}
```

And now make the hook script (the one that will end up in the initrd).  This script will run after the root-key LUKS has been opened, and will copy the root-key file out so it is ready for zfs to use.  It will also open the LUKS for the swap with that key to make sure it is ready before the initrd gets to trying to resume from hibernate. &#x20;

There is also a cleanup function that will be called at the end of initrd before switching root to the real filesystem.  We will delete the key in this step just to make sure it isn't hanging around in RAM when we don't need it anymore. &#x20;

```
cd /etc/initcpio/hooks
emacs rootkey
```

Paste in

```
# 
# 
# 
# 

swap_uuid=198c9ec0-274f-40a9-a104-e085f2fdee08
run_hook() { 
    dd if=/dev/mapper/root-key bs=32 count=1 of=/root-key
    cryptsetup open /dev/disk/by-uuid/$swap_uuid swap-crypt -d /root-key
    cryptsetup close /dev/mapper/root-key
}

run_cleanuphook() { 
    rm /root-key
}
```

And finally we need to add our new script to the config so it gets used. &#x20;

```
emacs /etc/mkinitcpio.conf
```

Find the HOOKS= line, and add encrypt and rootkey, so it looks like this:

```
HOOKS=(base udev autodetect microcode kms modconf block keyboard keymap consolefont encrypt rootkey resume zfs filesystems fsck)
```

## Configure the kernel command line

Get the UUID for the root-key partition&#x20;

```
blkid /dev/disk/by-partlabel/root-key
```

And set the cmdline in the bootloader config /boot/loader/entries/linux-cachyos.conf

```
emacs /boot/loader/entries/linux-cachyos.conf
```

and paste in:

```
quiet splash rw root=ZFS=zpcachyos/ROOT/cos/root cryptdevice=UUID=96c417e9-3bf1-4838-bd27-fee22b1c0aba:root-key RESUME=/dev/mapper/swap-crypt
```

And this one is used for UKI kernel images

```
emacs /etc/kernel/cmdline
```

Paste in

```
quiet splash rw root=ZFS=zpcachyos/ROOT/cos/root cryptdevice=UUID=96c417e9-3bf1-4838-bd27-fee22b1c0aba:root-key RESUME=/dev/mapper/swap-crypt
```

Also, depending on how you configured things, you probably need to put those same additions to the kernel command line in the /etc/sdboot-manage.conf

## Test reboot

Then we will make a snapshot and then recreate the initrd, and reboot to test.  &#x20;

```
sudo zfs snapshot -r zpcachyos@hybrid2
sudo kernel-install add-all
```

And reboot to test.  If it all worked, it should ask for the password for root-key, then the password for the zpool, and then should boot ok. If it booted ok, you should have /dev/mapper/swap-crypt open already.&#x20;

If that's good, then we add the swap into fstab. &#x20;

```
emacs /etc/fstab
```

And add this line:

```
/dev/mapper/swap-crypt none swap sw,nofail 0 0
```

Save that and reboot. This time, when done you should have swap working. This should let you hibernate and resume from hibernation.

Once that is working, we will want to change zfs so we don't need to type in the password for root-key and a password for zfs too. &#x20;

## Change zfs key

This is the dangerous part. If this goes wrong (which is unlikely but not impossible), recovery could be quite a pain.  Do whatever are your normal backup procedures before continuing. &#x20;

We will replace the password with the key from our LUKS root-key volume.

```
zfs change-key -l -o keyformat=raw -o keylocation=file:///root-key zpcachyos
```

Reboot again for final testing.  If it boots fine, and only needed a single password to unlock the encrypted filesystems, then we are good.  The last step is to remove the /root-key file so the only copy is the one in the root-key partition. &#x20;

```
rm /root-key
```

## Setup backups

And as a final touch, I decided that it would be a good idea to backup the EFI fileystem and the root-key when backing up the system.  The easy way to do this was to create a space in ZFS for both of those, and just copy them on boot so those copies get backed up with the rest of the zpool.  We don't want the backup of the root-key encrypted with zfs, because it contains the key we might need to unlock the zfs encryption. &#x20;

So we will create these zfs spaces:

```
zfs create zpcachyos/system
zfs create -o mountpoint=/mnt/efi-backup zpcachyos/system/efi_backup
zfs create -o encryption=off zpcachyos/system/root-key-backup -V 17M
```

Then we will create the backup script:

```
emacs /root/bin/backup_efi_and_key
```

And paste in:

<pre><code>#!/bin/bash
#
# When root is in zpool, /efi cannot be.  Also I am using data in a LUKS volume to unlock
# encrypted zfs datasets.  Both of these should be backed up into zpool so they get
# copied with zfs send backups.  Hopefully recovery just means:
#1. make partitions
#2. make efi filesystem
#3. zfs send from backup zpool into new local zpool
#4. copy efi from backup in fs to efi partition
#5. copy root-key from zpool to root-key partition
#6. install bootloader and reboot
#

# EFI is either /boot/efi, or just /boot on cachyos
efi_mountpoint=/boot/efi
if [[ $(grep DISTRIB_ID /etc/lsb-release) == "DISTRIB_ID=cachyos" ]] ; then
  efi_mountpoint=/boot
fi
efi_backup_dest=/mnt/efi-backup

root_key_partition=/dev/disk/by-partlabel/root-key
root_key_dest="$(realpath /dev/zvol/*/system/root-key-backup)"

# Backup root-key partition
dd if="${root_key_partition}" of="${root_key_dest}" bs=1M 2> /dev/null 

# Backup EFI
# fail if efi is not mounted
if [[ $( stat -c %d "${efi_mountpoint}"  ) == $( stat -c %d "${efi_mountpoint}/..") ]] 
then
<strong>    echo "${efi_mountpoint} not mounted"
</strong>    exit 1
fi

# fail if efi backup is not mounted
if [[ $( stat -c %d "${efi_backup_dest}"  ) == $( stat -c %d "${efi_backup_dest}/..") ]]
then
    echo "${efi_backup_dest} not mounted"
    exit 1
fi

rsync -aHAXx --delete-after "${efi_mountpoint}/" "${efi_backup_dest}/" &#x26;&#x26; zfs umount /mnt/efi-backup
exit 0
</code></pre>

And we want that to run a little while after a reboot.  Lets say 2 minutes, which should keep us from backing it up if it is broken, and should keep the backup from slowing boot.  I previously did this with cron, but arch isn't all that interested in keep us old gray-beards comfortable, so it takes extra steps to make cron work.  So instead, we will use a systemd timer.  This will also let us run the backup script daily.  That's probably not needed, but I wanted to see if it really does work, and surprise! it does. &#x20;

A systemd timer set needs two more files, a service unit file, and a timmer file to trigger the service.  So first lets make the service.  Paste this into /etc/systemd/system/backup-efi-and-key.service

```
[Unit]
Description=Backup EFI partition and root-key to ZFS

[Service]
ExecStart=/root/bin/backup_efi_and_key        
```

And paste this into /etc/systemd/system/backup-efi-and-key.timer

```
[Unit]
Description=Backup EFI partition and root-key to ZFS

[Timer]
OnBootSec=60
OnUnitActiveSec=24h
Persistent=true

[Install]
WantedBy=timers.target
```

And that should be it, we should be able to boot and unlock all the encrypted filesystems with a single password on boot.  It should be easy to have tang unlock everything automatically over the network without a password, or use a yubikey to unlock.  It should even be possible to set up all of these in parallel so you can unlock with whatever process you find most conveneient. &#x20;
