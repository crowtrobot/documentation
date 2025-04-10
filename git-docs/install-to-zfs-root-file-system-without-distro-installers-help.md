# Install to ZFS root file system without distro installer's help

If you just want steps to follow without all the crap about why I'm doing it, skip down to the "4 encryption options" section. &#x20;

## Why?

I have decided that my laptop is working too well, and needs some complications. It also needs a different distribution as the age of some packages in Pop!\_OS is making some things not work well (like python issues in ansible). Basic list of what I want:

* apt/deb packages (customizations I’ve scripted should mostly just work)
* NO SNAPS!
* KDE (doesn’t need to be installed as default by the distro)
* ZFS root filesystem (I want to play with it again)
* Fully encrypted drive (standard rule is encrypt it unless there is a good reason not to)

\
It seems that a lot of distributions that at one time supported installing to boot from zfs don’t anymore. I guess they found it hard to maintain in their installer? But this is Linux, so if you want it to do something bad enough to do the work, it can be done.

I’ve found a lot of instructions online about making a system boot from zfs by doing complicated things like make a special zpool for /boot with some special options set up to make it compatible with grub. I want to keep this relatively simple, so I’m not going to have a separate /boot nor grub.&#x20;

Just like I did with my fully encrypted drive, I will boot from a kernel and initrd that are in the EFI partition. The EFI partition can’t be encrypted, but we can use the secure boot mechanisms to ensure that the kernel and initrd haven’t been tampered with. Once we have an initrd, basically anything you can have in a full linux system is possible, like ZFS root without the special features used for the /boot pool.

\
This does mean that this process will only work for machines booting with UEFI. If you boot in bios mode this won’t work for you.

I’ve chosen Linux Mint for this because I haven’t used it in a while, but have always liked it. Like so many systems, Linux Mint doesn’t support installing into a ZFS root anymore.  These steps should work pretty much the same for most Debian-like Linux distributions. The script added to initrd for unlocking ZFS in the hybrid setup is written specifically for initramfs-tools, and probably won't work for dracut (which is found on most red hat like distros) without significan changes. &#x20;

All we need is either a full live OS environment that the installer runs in, or one we can boot after doing the installation.

What we are going to do here is just do a normal install in a supported file system, and then move that freshly installed system into zfs. &#x20;

## 4 encryption options

I see 4 options here:

* No encryption -&#x20;
  * Only do this for testing. As a general rule, everything should be encrypted unless there’s a good reason not to.
* ZFS native encryption -
  * ZFS has encryption built-in. This won’t encrypt some metadata (like zfs volume names), but encrypts the data. One big advantage here is that you can zfs send the encrypted data to a machine that doesn’t have the key for backups. That backup machine doesn’t need the key so it can’t see the data, but can still do scrub and all that to verify that it is stored safely.
  * The big disadvantage is that you can only have one password that you either type in, or load from a file. No separate passwords for separate users, no TPM unlock, no clevis/tang, etc.
  * ZFS native encryption also doesn’t support re-encrypting. If you need to change the master key you have to send the data somewhere, rebuild the zpool, and then zfs receive the data back.
* LUKS encryption -
  * Wrapping the zpool in LUKS. LUKS enables a lot of password options, like up to 32 separate passwords, each of which can be a typed in password, a keyfile, the TPM, a yubikey, or even a key derived over the network with clevis and tang. LUKS also supports in-place re-encrypting, so you can replace a master key if you suspect it has been compromised.
* Hybrid LUKS/ZFS native -
  * This is my scheme for getting some of the advantages of LUKS to work with ZFS. ZFS is encrypted with a key file, and that key file is then stored in a LUKS volume. During boot you get all the options LUKS has for unlocking, and once LUKS volume is unlocked, the key it holds is automatically used to unlock the ZFS native encryption.

I'm not considering encrypted zfs within LUKS because I don't like the inefficiency of double encryption. &#x20;



## The initial installation

This part is basically going to be a mostly normal install of whatever distribution you want. There's some steps to take first to create partitions we will want later.  You could probably use the GUI of the installer for this, but I had trouble figuring out how to get what I wanted out of that, and just went back to the tools I know.  So I launched cfdisk, made a GPT partition table, and made these partition:

1. 1G EFI partition (I will call it  /dev/sda1). Make sure to pick the type EFI
   1. You don’t need to do this if you already have EFI partition, but I wanted to make mine a bit larger. I don’t mind wasting a little bit of space to make sure I don’t run out of space while installing a new kernel-install.
2. Swap partition that is larger than system RAM (lets call it /dev/sda2)
   1. This again is optional but it’s best not to put swap in zfs at all. That might not be the problem it used to be, but it was a painful lesson. :smile: Make it larger than your RAM if you want hibernation support.
3. A 17MB partition (sda3).&#x20;
   1. This will only be used if you do the hybrid encryption option. Even if you are doing zfs native, you can make this partition and ignore it for now, just to give yourself the option to easily switch to the hybrid option in the future.
   2. Why 17MB? The LUKS header and keyslots use the first 16MB, and we need another 32 bytes after that. But we also want the partitions aligned on the MB so 16MB+32 rounded up to the next MB is 17MB.
4. A regular partition for rest of drive (sda4)
   1. This will eventually be our root zpool.

Before starting the installer, lets format that EFI partition (if we made a new one), because some installers seem to crash if they find an unformatted EFI partition. &#x20;

```
mkfs.fat -F 32 /dev/sda1
```

And here we branch to 2 options for the install.  We can either do a normal install with ext4 or whatever the installer supports in that last partition, or on a separate drive if we have one. The separate drive is a little easier and faster, but isn't always an option.  Either way, do the normal install process. At the end of the install choose the option to keep running from the install disk, or if your distribution doesn’t offer that, boot immediately into a live linux environment that can run zfs (like linux mint). We don’t want to boot into the installed system yet.



## Make the ZFS

To do this we are going to need ZFS tools.  I'm using a Linux mint install disk and on there I was able to just install the zfs tools with apt.  While we are doing this, lets install some other utilities we might need.  Obviously you can leave the emacs out if you prefer a different editor. &#x20;

```
sudo su -
apt update
apt install zfsutils emacs mbuffer
```

<details>

<summary>If you only have one drive</summary>

If you only have the one drive, we will first need to copy the installed system off to another computer on the network, reformat to zfs, and then copy it back over the network onto the local drive again.  This is the step where we copy of to another machine. &#x20;

We could use netcat, but I like mbuffer as it can add a buffer (and status info).  First ssh to the remote machine and run:

```
mbuffer -I 1234 -o temp_root_fs.tar.gz
```

On the machine we are installing on:

```
mount /dev/sda4 /mnt
cd /mnt
apt install mbuffer
tar -cz . | mbuffer -m 1G -O 192.168.1.2:1234
```

Obviously put the IP of the machine that will hold or tar files in place of the example 192.168.1.2.  When that is done:

```
cd /
umount /mnt
```

</details>

Make the zpool in that last partition. There’s basically 3 options here, no encryption, LUKS encryption, and zfs native encryption. The hybrid sertup will get set up the same as zfs native for this step. &#x20;

Look in /dev/disk/by-id to find the id for our disk. We will name the disk by this ID when making the zpool to avoid potential future confusion if what is sda now becomes sdb in the future.

Use the zpool command to make our zpool (you might need to add -f if this partition has held a previous file system).

<details>

<summary>no encryption</summary>

```
zpool create -o ashift=12 -o autotrim=on -O acltype=posixacl \
    -O xattr=sa -O dnodesize=auto -O compression=lz4 \ 
    -O normalization=formD -O relatime=on -O canmount=off \
    -O mountpoint=/ -R /mnt test-rpool /dev/disk/by-id/scsi*part4
```

</details>

<details>

<summary>LUKS encryption</summary>

Make the LUKS volue, open it, and make the zpool inside of it. &#x20;

```
cryptsetup luksFormat --type=luks2 /dev/sda4
cryptsetup open /dev/sda3 rpool-crypt
zpool create -o ashift=12 -o autotrim=on -O acltype=posixacl \
  -O xattr=sa -O dnodesize=auto -O compression=lz4 
  -O normalization=formD -O relatime=on -O canmount=off 
  -O mountpoint=/ -R /mnt test-rpool /dev/mapper/rpool-crypt
```

</details>

<details>

<summary>ZFS native encryption and Hybrid</summary>

This is the same as the no encryption but with "-O encryption=on -O keylocation=prompt -O keyformat=passphrase" added

```
zpool create -o ashift=12 -o autotrim=on -O acltype=posixacl \
  -O xattr=sa -O dnodesize=auto -O compression=lz4 \
  -O normalization=formD -O relatime=on -O canmount=off \
  -O mountpoint=/ -O encryption=on -O keylocation=prompt \
  -O keyformat=passphrase -R /mnt test-rpool /dev/disk/by-id/scsi*part4
```

While creating that pool we used the "-R /mnt" option, which sets the "altroot" property. This sets a path for mounts to happen in so when we make a filesystem configured to mount for example in /home it will be mounted instead in /mnt/home.&#x20;

</details>

Now lets make some datasets (sometimes called filesystems). &#x20;

It is generally considered best practice not to add data to the top of the zpool, so we will make a dataset to be our root file system. &#x20;

Originally I planned to just have a file system (dataset) for / and another for /home so that I could rollback the system while keeping the current state of user files. This is actually more complicated than that. We probably want to keep /var/log too, and there’s some other parts of /var (and /tmp) that we shouldn’t waste disk space on keeping snapshots of.

So, based mostly on the recommendations from [https://openzfs.github.io/openzfs-docs/Getting%20Started/Debian/Debian%20Buster%20Root%20on%20ZFS.html](https://openzfs.github.io/openzfs-docs/Getting%20Started/Debian/Debian%20Buster%20Root%20on%20ZFS.html) here’s my list of zfs create commands. I will probably be making some modifications to this in the future.&#x20;

The ones made with “canmount=off” are just there to make the paths work. For example test-rpool/var only exists so that we can make test-rpool/var/tmp and the others, and have them correctly automatically inherit their mountpoint. The data in /var won’t be in test-rpool/var but in test-rpool/ROOT/mint-root. I am worried this might be confusing so I might come back to this later and test an alternative option (like zfs create -o mountpoint=/var/log test-rpool/var\_log).

```

zfs create -o canmount=off -o mountpoint=none test-rpool/ROOT
zfs create -o canmount=noauto -o mountpoint=/ test-rpool/ROOT/mint-root
zfs mount test-rpool/ROOT/mint-root
zfs create test-rpool/home
zfs create -o mountpoint=/root test-rpool/home/root
chmod 700 /mnt/root
zfs create -o canmount=off test-rpool/var
zfs create -o canmount=off test-rpool/var/lib
zfs create test-rpool/var/log
zfs create test-rpool/var/spool
zfs create -o com.sun:auto-snapshot=false test-rpool/var/cache
zfs create -o com.sun:auto-snapshot=false test-rpool/var/lib/nfs
zfs create -o com.sun:auto-snapshot=false test-rpool/var/tmp
chmod 1777 /mnt/var/tmp
zfs create test-rpool/var/lib/AccountsService
zfs create test-rpool/var/lib/NetworkManager
zfs create -o com.sun:auto-snapshot=false test-rpool/tmp
chmod 1777 /mnt/tmp
zfs mount -a

```

usually “zfs mount -a” isn’t needed, but because we need the “canmount=noauto” on some we need to manually trigger a mount.

<details>

<summary>Alternate layout</summary>

There's some things about that arrangement of ZFS datasets I don't like, so here I am going to create an alternative set that I hope to test later.  The idea is that the system stuff is under test-rpool/system, user date under test-rpool/data. &#x20;

```
zfs create -o canmount=off -o mountpoint=none test-rpool/system
zfs create -o canmount=noauto -o mountpoint=/ test-rpool/system/mint-root
zfs mount test-rpool/system/mint-root
zfs create -o mountpoint=/var/log test-rpool/system/var_log
zfs create -o mountpoint=/var/spool test-rpool/system/var_spool
zfs create -o mountpoint=/var/cache -o com.sun:auto-snapshot=false test-rpool/system/var_cache
zfs create -o mountpoint=/var/lib/nfs -o com.sun:auto-snapshot=false test-rpool/system/var_lib_nfs
zfs create -o mountpoint=/var/tmp -o com.sun:auto-snapshot=false test-rpool/system/var_tmp
chmod 1777 /mnt/var/tmp
zfs create -o mountpoint=/var/lib/AccountServices test-rpool/system/var_lib_AccountsService
zfs create -o mountpoint=/var/lib/NetworkManager test-rpool/system/var_lib_NetworkManager
zfs create -o mountpoint=/tmp -o com.sun:auto-snapshot=false test-rpool/system/tmp
chmod 1777 /mnt/tmp

zfs create -o canmount=off -o mountpoint=none test-rpool/data
zfs create -o mountpoint=/home test-rpool/data/home
zfs create -o mountpoint=/root test-rpool/data/root
chmod 700 /mnt/root

zfs mount -a
```



</details>

## Copy the system into ZFS

<details>

<summary>If you have two drives</summary>

Mount the drive we installed to in /tmp/src and copy all the files into the new zfs.

```
mkdir /tmp/src
mount /dev/sdb1 /tmp/src
rsync -aHAXxP /tmp/src/ /mnt/
```



</details>

<details>

<summary>If you have only one drive</summary>

Now we need to copy the installed system back over the network from the temporary storage. &#x20;

On the local system (the machine we are installing on):

```
ip -c a 
# just to see what our IP is, we will need that in a min
cd /mnt
mbuffer -I 1234 | tar -xz
```

On the remote machine:

```
mbuffer -i temp_root_fs.tar.gz -O 192.168.1.3:1234
```

Obviously replace that IP with the one for the machine you are installing on

</details>

## chroot into the new installation

We are going to switch into the new system and run there as if we had booted into it, by using the chroot command. This will let us fix some things so we can boot into this system for real. We need to do a bit of prep work before the chroot will work.

```
mount -t tmpfs tmpfs /mnt/run
mkdir /mnt/run/lock
mount --make-private --rbind /dev /mnt/dev
mount --make-private --rbind /proc /mnt/proc
mount --make-private --rbind /sys /mnt/sys
```

Copy some files so dns will work inside the chroot

```
cp -rp /run/systemd /mnt/run/
```

And now we chroot. From here on until we exit this shell, every command we run will see the new installation at / even though it is really in /mnt. So we can install things in the new system with apt just like we would if we had booted into it.

Now we can chroot and install some stuff we need. &#x20;

```
chroot /mnt bash --login
apt update
apt install emacs htop zfsutils-linux zfs-initramfs linux-headers-generic \
    linux-image-generic zfs-dkms crytpsetup-initramfs systemd-boot systemd-ukify
```

### Fix the efi directory in fstab. &#x20;

```
blkid /dev/sda1
```

To get the UUID of the EFI partition.  Then edit `/etd/fstab` to change the /boot/efi line like this:

```
UUID=60B5-48B4  /boot/efi   vfat    umask=0077     0   1
```

### Install systemd-boot boot loader

Now we will mount it and make a directory.  Systemd-boot will look for this directory later

```
mount /boot/efi
mkdir -p /boot/efi/loader/entries
```

Since we aren’t using grub anymore, we want to remove it from the efi partition so it doesn’t cause confusion (for us or the computer). This might be a little different for you, but for me inside of /boot/efi/EFI there was “ubuntu”, “Boot”, “Linux”, and “systemd”. Don’t do this if you have another OS you want to keep (dual booting).

```
cd /boot/efi/EFI
rm -fr ubuntu
# or whatever you have that isn’t needed
```

And now lets install thee systemd-boot boot loader

```
bootctl install --path=/boot/efi
```

### Change the boot order to make sure systemd-boot is first

```
efibootmgr
```

This will show the current order (looks like “BootOrder: 0004,0002,0000,0001”). It also shows what each of those numbers means. For example, for me systemd-boot looks like:\
`Boot0002* Linux Boot Manager HD(1,GPT,ab819271-0516-404b-a3ef-72397f144963,0x800,0x800000)/File(\EFI\systemd\systemd-bootx64.efi)`

```
efibootmgr -o 0002
```

Set the boot order to be only 0002, which we just saw was systemd-boot.  Obviously you need to pick the right one for your system. &#x20;

### Configure the kernel-install utility

```
#echo 3 >/etc/kernel/tries
# There seems to be a bug with this causing duplicated uki images in efi. 
# I need to look into that, but for now, don't use "tries".
cp /usr/lib/kernel/install.conf /etc/kernel/
emacs /etc/kernel/install.conf
```

And add:

`layout=uki`

Save and exit

### Configure ukify:

```
emacs /etc/kernel/ukify.conf
```

add:\
`[UKI]`\
`SignKernel=no`

Save and exit.

### Configure kernel command line

Whatever you put in this file will be used as the kernel command line. So if you need extra options here to make things work for your hardware, like “modprobe.blacklist=psmouse” be sure to add those here too. After this is working, you probably want to edit this file to add “quiet splash” for the more pretty boot.

```
emacs /etc/kernel/cmdline
```

Add: \
`root=ZFS=test-rpool/ROOT/mint-root boot=zfs zfsforce=1`

### Remove /  from fstab

Edit the fstab and remove the line for / since we aren’t using the old root anymore.

```
emacs /etc/fstab
```

remove the line for /. ZFS has it’s own way of recording mount points so you don’t need an entry here for / at all.

### Set up hybrid LUKS partition

If you are doing the hybrid LUKS/native ZFS encryption setup, here is where most of that happens

<details>

<summary>Setup hybrid LUKS</summary>

```
cryptsetup luksFormat --type=luks2 /dev/sda3
  # Just do a password for now, change it when everything is done.  
cryptsetup open /dev/sda3 root-key
blkid /dev/sda3
  # Looking for the UUID
emacs /etc/crypttab 
```

and add a line like (using the UUID you just got):

```
root-key UUID="99098cec-a750-47b7-b1a4-2840a87dd847" none luks,initramfs,readonly
```

Make a file of exactly 32 bytes of random data (using tr to make sure none of them are null bytes, since one of those caused me trouble).  Then copy that into our LUKS volume, and change the ZFS password to use this file as a raw key (which will replace the temporary password set earlier). &#x20;

```
tr -d '\000' < /dev/urandom | dd bs=32 count=1 of=/root-key
cat /root-key > /dev/mapper/root-key
zfs change-key -o keyformat=raw -o keylocation=file:///root-key test-rpool
```

Download the zfs-hybrid-unlock script from [https://github.com/crowtrobot/hybrid-zfs-root-unlock/blob/main/zfs-hybrid-unlock](https://github.com/crowtrobot/hybrid-zfs-root-unlock/blob/main/zfs-hybrid-unlock) and save to /etc/initramfs-tools/scripts/local-top/zfs-hybrid-unlock .  Edit the file to make sure that it has the right root\_key\_file and luks\_key\_volume for your system. &#x20;

This simple script will extract the key for zfs from our LUKS volume and put it where zfs expects to find it.

```
chmod +x /etc/initramfs-tools/scripts/local-top/zfs-hybrid-unlock
```

</details>



### Set up the swap space

Even if you aren’t encrypting other stuff, it’s still a good idea to encrypt the swap. &#x20;

If you aren’t doing the hybrid setup, you need to make a keyfile for the swap. It will be ok for this file to just stay here. The idea is that if someone can read your swap they might get from it the key to unlock encryption on your file system, but if they already have your file system they probably don’t care about your swap.

If you didn't make /root-key above, make one now with

```
tr -d ‘\000’ < /dev/urandom | dd bs=32 count=1 of=/root-key
```

Make the swap space

```
cryptsetup luksFormat /dev/sda2 --type=luks2 /root-key
cryptsetup open /dev/sda2 swap-crypt -d /root-key
mkswap /dev/mapper/swap-crypt
blkid /dev/sda2
```

That last command gave us the UUID, we will need to put in an /etc/crypttab line line this:

```
swap-crypt UUID="cc52d606-917a-4352-80ff-b53fdcfc81da" /root-key luks,discard,initramfs
```

And we will add a line for swap in the /etc/fstab, like this:

```
/dev/mapper/swap-crypt none swap sw 0 0
```

### Rebuild the initrd

After all these changes we will need to update our initrd so it has the current fstab, etc. The kernel-install and ukify we installed set up hooks for themselves so when the new initrd is created, it will automatically trigger the creation of a unified kernel image (UKI), which should automatically be copied to the efi partition, and then the entries in the systemd-boot config updated to match.

```
update-initramfs -ck all
```

## &#x20;Reboot into the new OS

And that’s basically it. After that shutdown, remove the install disk, and power on. If you didn’t already you will need to go into the BIOS setup and turn off secure boot. It’s possible to add your own key, and configure ukify to sign images with that key, and then turn secure boot back on, but only try after everything is working. &#x20;

The first boot might fail and drop to an emergency initramfs shell. This is because the zpool claims it was last mounted by a different OS (the install disk), so we need to manually import it. The “zfsforce=1” option we added to the kernel command line was supposed to fix this for us, but if it didn’t, just run

```
zpool import -f test-rpool
```

Then hold the power button to power off, and then power on again.  Then it boots normally, and will continue to boot normally next time you reboot.

## Close root-key LUKS volume after boot (hybrid setup only)

When done booting, there will still be a /dev/mapper/root-key device.  We won't need it again until the next boot, so might as well close it so the key isn't easily read.  To do that lets make a cron job

```
sudo crontab -e
```

This will open an editor showing the crontab (cron job table) for root.  Add a line like this:

```
@reboot /usr/sbin/cryptsetup close root-key
```

Instead of specifying a time and date for this job to run, we used the "@reboot" time.  This is a special way of saying "when the system has just been booted up" and makes for an easy way to do simple jobs like this on boot. &#x20;

## Finishing touches

That should be a usable system.  If you set up the hybrid encryption you will want to backup the /root-key file somewhere.  I will put mine in the password manager I use for secrets for all my systems.  Once it is backed up, you can remove that file so the only copy on the system is the encrypted on in the LUKS volume. &#x20;



I plan to figure out and add here some ZFS on root specific stuff like:

* zfs send backup without the remote machine having the key
* Restores (as if the disk failed, go from nothing to running restored system)
* Making ZFS snapshots, and using them to roll back to undo changes (is there a linux tool like bectl?)
* Get secure boot working again



