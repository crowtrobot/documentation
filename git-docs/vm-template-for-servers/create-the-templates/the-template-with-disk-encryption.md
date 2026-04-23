---
description: This template has the root filesystem encrypted with LUKS.
---

# The Template With Disk Encryption

## Make the template

Make a new VM, and choose these options (this is specific to Proxmox, but probably easily translates to other environments):

* Choose the Debian install iso (probably want the netinstall since we will be making a pretty minimal installation).
* Check the box for Qemu Agent
* Set the disk to 8GB. Turn on discard and SSD emulation.
* Pick 1 CPU, with type set to the best that all hosts in your cluster support (x86-64-v3 for example, or host if you don't have proxmox in a cluster).
* Set to only 1024 MB of RAM
* Pick a network that will let the installer download packages (and preferably isolated from production machines)

Once the VM has been created, but before booting it for the first time:

* Go to the VM Options page, and turn off the "Use tablet for pointer". This causes (or maybe only used to?) a few % extra CPU use in each VM, so those that won't have a GUI interface are better off without it.
* Take a snapshot (in case something goes wrong in the install you won't have to start over completely)

Power on the VM, and open the console to go through the Debian installer. Go with the defaults except for these settings:

* For the host name just set debian or template or something generic like that. It will get changed by cloud-init on machines cloned from this template.
* For the root user, set the password to something easy to type (like "password"). We will need to be able to log in as root later, but after that we will disable root logins.
* Create user "debian" with a simple password, like "password". We will remove this user later.
* Choose the manual partitioning.  The order of these matters a little.  To make it possible to grow the root filesystem, it needs to be the last one on the disk. &#x20;
  * Make a new 512MB partition at the beginning of the drive
    * Use as ext4
    * Set the mount point to "/boot"
    * Set the label to boot
    * Set the "bootable" flag on
  * Make another new primary partition using 2G of disk at the beginning
    * Use as physical volume for encryption
    * For the key choose "random key".  That will also automatically make this partition be swap. &#x20;
  * Make another new primary partition using the rest of the disk.
    * Use as: physical volume for encryption
    * Set the key to "passphrase"
  * Select to configure encrypted volumes
    * You need to say yes for it to apply what we already have set up, and then select Finish.
    * Yes again to create the encrypted volume, and yes once more for the other one.  &#x20;
    * Set the encryption passphrase to something simple, like "password"
    * Choose the large encrypted volume
      * Use as xfs or ext4 (or whatever you prefer)
      * Set mountpoint to /
      * Set the options relatime and discard
      * Set the label to root
      * Choose done
  * Finished partitioning and write changes to disk
* We only need the ssh server and standard system utilities package sets.
* Install grub as normal, and finish the installation.

When it has finished with the shutdown and starts to boot up again, shutdown the VM. We will make some more changes before letting it boot the first time. The changes are:

* Remove the CD Drive from hardware.
* Take a new snapshot so if something goes wrong later you can try again from here.

Start the VM, and let it boot up for the first time. You will need to open the console and type in our temporary luks password.  You can log in locally, or ssh in as the user "debian" we created earlier. Ssh does make it easier to copy-and-paste commands. Once logged in run these commands:

* `su -` you will need to put in the root password. You can't ssh in directly to the root user account, and sudo isn't installed yet.
* `apt update`
* `apt dist-upgrade`
* optional: install some utilities that might make the rest of this just a bit easier: `apt install emacs-nox`
* `apt install sudo qemu-guest-agent cloud-init`&#x20;
* `apt clean`
* edit `/etc/initramfs-tools/conf.d/resume` to "RESUME=none".  This disables resume from hibernate, but that wasn't going to work anyway since the swap space gets a new randomly generated key each boot.  We don't want it to even try since that wastes time during boot, and prints some slightly scary messages. &#x20;

Optional:  Lets fix the LUKS volume names.  By default debian installed it with the encrypted volumes named "sda2\_crypt" and "sda3\_crypt".  Those aren't very descriptive, and if another disk is added, sda3 crypt might be on sdb.  I hate everything about this, so lets rename to swap\_crypt and root\_crypt. &#x20;

<details>

<summary>Renaming the LUKS volume name</summary>

* Edit `/etc/crypttab` and change sda2\_crypt to swap\_crypt, and sda3\_crypt to root\_crypt. &#x20;
* Edit `/etc/fstab` to make the same change
* `update-initramfs -uk all` &#x20;
  * There will be some errors because the fstab doesn't match what is currently mounted.  We will fix that in a sec
* `update-grub`
* `reboot`
* Look at the VM console.  Put in the luks passphrase, and then boot will fail, and drop to the initramfs prompt. &#x20;
  * `cryptsetup open /dev/sda3 root_crypt`
  * `exit`
* That got it to boot so we can ssh in and become root again.  And to fix it so that you don't have to do that again, run `update-initramfs -uk all`
* Edit the `/etc/crypttab` again, this time to change the /dev/sda2 to use the UUID like it should have had all along
  * get the UUID with blkid /dev/sda2, it will look like PARTUUID="4f14dc8d-02"
  * edit `/etc/crypttab` and change the swap\_crypt line to look like:
    * swap\_crypt /dev/disk/by-partuuid/4f14dc8d-02 /dev/urandom cipher=aes-xts-plain64,size=256,swap,discard,x-initrd.attach
    * The only change is /dev/sda2 replaced with /dev/disk/by-partuuid/...

</details>

Now lets get rid of the need to type the password on boot.  This is terribly insecure, but it is just temporary.  The idea is to make a template we can deploy a working VM from without needing to get into the console.  After a VM is made from this template, one of the first things that should happen is to replace this with tang.  I could put the tang binding in here now, but we don't want every VM using the same secrets. &#x20;

* Make our temporary password file
  * `dd if=/dev/urandom of=/boot/initial_password bs=1K count=1`
  * `chmod 400 /boot/initial_password`
* Replace the password with that file
  * `cryptsetup luksChangeKey /dev/sda3 /boot/initial_password`
    * This should be the last time you have to type in the LUKS password
* Configure initrd to copy that file
  * edit `/etc/cryptsetup-initramfs/conf-hook`
    * find KEYFILE\_PATERN line, and edit to look like `KEYFILE_PATTERN=/boot/initial_password`
  * edit `/etc/crypttab`
    * Change the root\_crypt (or sda3\_crypt if you didn't do the optional steps to rename it) to replace "none" with "/boot/initial\_password" so that it knows to use that file to unlock LUKS.  It should end up looking like "​​root\_crypt UUID=6344e508-752c-45a0-94b7-3b7897d437a1 /boot/initial\_password luks,discard,x-initrd.attach"
* `update-initramfs -uk all`
* `update-grub`
* And reboot to test.  It should reboot and without needing any help should boot all the way up to where you can ssh in again. &#x20;
* Shutdown, snapshot, and boot up again. &#x20;

Now log out of SSH if you are logged in with SSH. In the VM console, log in as root, and run these commands.  This will leave the VM with no users, and root not allowed to log in.  If this works, a user will be created by cloud-init when a VM is deployed from this template. &#x20;

* `userdel debian`
* `rm -fr /home/debian`
* `passwd -d root`
* `passwd -l root`
* `fstrim -va`
* `shutdown -h now`

Once the VM has shutdown, we will make a few more changes to the VM:

* On the hardware tab, add a cloudInit drive.
* On the Cloud-Init tab:
  * Set a username
  * Copy in your ssh public key
  * Set the IP to DHCP
* One final snapshot. After testing the VM we will revert to this one.

To test, we will grow the VM disk, and boot the VM. SSH in as your user with your ssh key.  This time the root FS won't be automatically resized.  If you look in lsblk the partition got resized, but you need to do "`cryptsetup resize`" on that partition to grow it, then grow the filesystem.  I have a small script I push and run with ansible to take care of that.  As long as it booted and you can ssh in with your key, then cloud-init is working. &#x20;

Shutdown, revert to the last snapshot, delete all snapshots, and convert this VM to a template.  You are done. &#x20;

## Steps to do when deploying from this template

There's some extra steps to take when deploying a VM from this template.  I have automated all of these with ansible, but it is worth having the steps documented in case someone doesn't want to use ansible. &#x20;

* Clone the template to a VM
* Assign the IP, resize the root disk (if needed), choose the RAM, etc. &#x20;
* Power on the VM. &#x20;
* SSH
  * `sudo apt install clevis clevis-initramfs clevis-luks`
  * Re-encrypt the LUKS volume with a new master key.  This keeps every VM deployed from this template from having the same master key (which would let an attacker use information learned from one VM to attack or even completely unlock the LUKS on another)
    * `sudo cryptsetup reencrypt /dev/sda3 -d /boot/initial_password`
  * Bind luks to a tang server.  I put the tang binding in LUKS keyslot 8 to keep is consistent across VMs with "-s8" but you can leave that out if you want.  For this example, the tang server's IP is 192.168.5.6.
    * `sudo clevis luks bind -d /dev/sda3 -s8 -k /boot/initial_password -y tang '{"url":"http://192.168.5.6/"}'`
  * Replace the keyfile on the disks with a fallback password you can type in if clevis fails to unlock LUKS with the tang server.  This is optional, but a really really good idea.  Wouldn't want one mishap with a single tang VM to cause data loss on all of your other VMs. &#x20;
    * `cryptsetup luksChangeKey -d /boot/initial_password /dev/sda3`
      * store the fallback password somewhere safe, like your password manager
  * Remove the old initial\_password file since it isn't needed anymore. &#x20;
    * `rm /boot/initial_password`
    * Replace the initial\_password in crypttab with "none"
      * `sed 's/\/boot\/initial_password/none/g' /etc/crypttab`
      * or just use your text editor
  * Remove the old initial\_password file from the initrd
    * edit `/etc/cryptsetup-initramfs/conf-hook` to remove or comment out the "KEYFILE\_PATTERN=/boot/initial\_password" line
  * update initrd and grub
    * `update-initramfs -uk all`
    * `update-grub`

And you are done.  This VM is now ready to use. &#x20;
