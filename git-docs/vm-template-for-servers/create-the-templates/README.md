---
description: >-
  The other pages in this area talk about deploying from the template, so it
  seemed reasonable that there should also be one about the steps to create the
  template.
---

# Create The Templates

We will actually create two templates. They are mostly the same, but one has the root filesystem encrypted with LUKS.&#x20;

The unencrypted template can automatically grow its disk on boot, but we don't want that to happen with the encrypted disk until after it has been re-encrypted with a new unique key, because it will have to re-write the entire filesystem and if the disk was already grown it can more than double the time that process will take.

## The Plain Template

First we will start with the plain, not encrypted template. Make a new VM, and choose these options (this is specific to Proxmox, but probably easily translates to other environments):

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
* Choose the manual partitioning
  * Make a new 2G partition at the beginning of the drive, and select "use as swap".
  * Make another new primary partition using the rest of the disk.
    * Use as xfs or ext4 (whichever you prefer)
    * Set the mount options to relatime and discard
    * Set the label to "root"
    * Set the "bootable" flag on
* We only need the ssh server and standard system utilities package sets.
* Install grub as normal, and finish the installation.

When it has finished with the shutdown and starts to boot up again, shutdown the VM. We will make some more changes before letting it boot the first time. The changes are:

* Remove the CD Drive from hardware.
* Take a new snapshot so if something goes wrong later you can try again from here.

Start the VM, and let it boot up for the first time. You can log in locally, or ssh in as the user "debian" we created earlier. Ssh does make it easier to copy-and-paste commands. Once logged in run these commands:

* su - you will need to put in the root password. You can't ssh in directly to the root user account, and sudo isn't installed yet.
* apt update
* apt disk-upgrade
* optional: install some utilities that might make the rest of this just a bit easier: apt install emacs-nox
* apt install sudo qemu-guest-agent cloud-init cloud-initramfs-growroot ifupdown2 isc-dhcp-client
* apt clean

Now log out of SSH if you are logged in with SSH. In the VM console, log in as root, and run these commands.  This will leave the VM with no users, and root not allowed to log in.  If this works, a user will be created by cloud-init when a VM is deployed from this template. &#x20;

* userdel debian
* rm -fr /home/debian
* passwd -d root
* passwd -l root
* fstrim -va
* shutdown -h now

Once the VM has shutdown, we will make a few more changes to the VM:

* On the hardware tab, add a cloudInit drive.
* On the Cloud-Init tab:
  * Set a username
  * Copy in your ssh public key
* One final snapshot. After testing the VM we will revert to this one.

To test, we will grow the VM disk, and boot the VM. SSH in as your user with your ssh key, and df -h to check that the filesystem was grown. If it all looks good, shutdown the VM and revert to the last snapshot. Then delete all snapshots and convert the VM into a template, and you are done. &#x20;

* For not encrypted \*\* Make new VM, name deb13-template \*\*\* Choose debian install disk \*\*\* Select QQemu Agent (we will install it) \*\*\* 8GB disk \*\*\*\* discard and ssd emulation on (wasn't with last template, but seems like a good idea) \*\*\* 1 CPU \*\*\* 1GB RAM \*\*\* vmbr1 (for now), firewall off \*\* Modify VM \*\*\* turn off tablet pointer \*\* Snapshot \*\* Power on \*\* Setup steps \*\*\* Pick english, qwerty, etc \*\*\* hostname debian, no domain \*\*\* root password: password \*\*\* user debian, password password \*\*\* Mountain tz \*\*\* manual partitions \*\*\*\* pick the disk \*\*\*\* new partition \*\*\*\*\* 2G \*\*\*\*\* primary \*\*\*\*\* begining \*\*\*\*\* use as swap \*\*\*\*\* done \*\*\*\* new partition \*\*\*\*\* Remaining size \*\*\*\*\* Primary \*\*\*\*\* Use as xfs \*\*\*\*\* options: relatime and discard \*\*\*\*\* label: root \*\*\*\*\* Bootable flag on \*\*\*\*\* done \*\*\* pick default package sources \*\*\* only ssh server and standard system utils \*\*\* yes to install grub, and pick only drive \*\* When it wants to reboot, power down \*\* Remove CD Drive \*\* Take another snapshot \*\* Boot for first time \*\* apt update \*\* apt dist-upgrade \*\* apt install sudo emacs-nox htop qemu-guest-agent cloud-init cloud-initramfs-growroot this time don't install ifupdown2 isc-dhcp-client \*\* apt clean \*\* log in as root \*\*\* emacs /etc/network/interfaces \*\*\*\* comment out the lo and eth parts \*\*\* userdel debian \*\*\* rm -fr /home/debian \*\*\* passwd -d root \*\*\* passwd -l root \*\*\* fstrim -va \*\* shutdown \*\* on hardware tab, add cloudinit drive \*\* on cloud-init tab \*\*\* user: alain \*\*\* SSH public key: paste in public key \*\*\* Hardware: set NIC to down \*\* snapshot c1 \*\* Test \*\*\* connect network \*\*\* grow disk \*\*\* power on \*\*\* Check that disk grew \*\*\* Check that user login works \*\* if it is all good \*\*\* revert to most recent snapshot \*\*\* delete snapshots \*\*\* convert to template
* For encrypted \*\* Make new VM, name deb13-crypt-template \*\*\* Choose debian install disk \*\*\* Select Qemu Agent (we will install it) \*\*\* 8GB disk \*\*\*\* discard and ssd emulation on (wasn't with last template, but seems like a good idea) \*\*\* 1 CPU \*\*\* 1GB RAM \*\*\* vmbr1 (for now), firewall off \*\* Modify VM \*\*\* turn off tablet pointer \*\* Snapshot (a1) \*\* Power on \*\* Setup steps \*\*\* Pick english, qwerty, etc \*\*\* hostname debian, no domain \*\*\* root password: password \*\*\* user debian, password password \*\*\* Mountain tz \*\*\* manual partitions \*\*\*\* pick the disk \*\*\*\* new partition \*\*\*\*\* 512MB \*\*\*\*\* primary \*\*\*\*\* begining \*\*\*\*\* use as ext4 \*\*\*\*\* /boot \*\*\*\*\* label boot \*\*\*\*\* bootable flag on \*\*\*\* new partition \*\*\*\*\* 2G \*\*\*\*\* primary \*\*\*\*\* begining \*\*\*\*\* Use as physical volume for encryption \*\*\*\*\* Key: random key (this automatically makes it into swap and such too) \*\*\*\*\* done \*\*\*\* new partition \*\*\*\*\* Remaining size \*\*\*\*\* Primary \*\*\*\*\* Use as: physical volume for encryption \*\*\*\*\* key: passphrase \*\*\*\*\* done \*\*\*\* Configure encrypted volumes \*\*\*\* Finish \*\*\*\* Yes \*\*\*\* Yes (let it set up the encrypted volumes so we can make filesystems) \*\*\*\* Set password to password \*\*\*\* Pick the large encrypted volum \*\*\*\*\* Ext4 \*\*\*\*\* Mountpoint / \*\*\*\*\* options: relatime and discard \*\*\*\*\* label: root \*\*\*\*\* Bootable flag on \*\*\*\*\* done \*\*\* pick default package sources \*\*\* only ssh server and standard system utils \*\*\* yes to install grub, and pick only drive \*\* When it wants to reboot, power down \*\* Remove CD Drive \*\* Take another snapshot (a2) \*\* Boot for first time \*\*\* password for luks \*\* apt update \*\* apt dist-upgrade \*\* apt install sudo emacs-nox htop qemu-guest-agent cloud-init #systemd-resolved Not this time: ifupdown2 isc-dhcp-client \*\* apt clean \*\* emacs /etc/cloud/cloud.cfg \*\*\* Remove the growpart and resizefs modules from "cloud\_init\_modules" section. \*\* emacs /etc/crypttab \*\*\* Change the crypt device names, like this: swap\_crypt /dev/sda2 /dev/urandom cipher=aes-xts-plain64,size=256,swap,discard,x-initrd.attach

root\_crypt UUID=6344e508-752c-45a0-94b7-3b7897d437a1 none luks,discard,x-initrd.attach \*\*\* x-initrd.attach is new, should it really be there? \*\* emacs /etc/initramfs-tools/conf.d/resume \*\*\* Change to RESUME=none \*\* update-grub \*\* emacs /etc/fstab \*\*\* Change names: /dev/mapper/root\_crypt / ext4 discard,relatime,errors=remount-ro 0 1

## /boot was on /dev/sda1 during installation

UUID=d52bcedf-8b27-40e9-b7db-dd7dc442e7c4 /boot ext4 defaults 0 2 /dev/mapper/swap\_crypt none swap sw 0 0 /dev/sr0 /media/cdrom0 udf,iso9660 user,noauto 0 0 \*\* update-initramfs -uk all \*\* reboot \*\*\* Will fail to initrd shell \*\*\* cryptsetup open /dev/sda3 root\_crypt \*\*\* ctrl-d \*\* ssh in, then su - to root \*\* emacs /etc/crypttab \*\*\* to add initial\_password file root\_crypt UUID=6344e508-752c-45a0-94b7-3b7897d437a1 /boot/initial\_password luks,discard,x-initrd.attach \*\*\* Also change swap partition name to UID, like: swap\_crypt /dev/disk/by-partuuid/bab993e9-02 /dev/urandom cipher=aes-xts-plain64,size=256,swap,discard,x-initrd.attach

\*\* Make initial password file \*\*\* dd if=/dev/urandom of=/boot/initial\_password bs=1K count=1 \*\*\* chmod 400 /boot/initial\_password \*\* Replace password with password file \*\*\* cryptsetup luksChangeKey /dev/sda3 /boot/initial\_password \*\* emacs /etc/cryptsetup-initramfs/conf-hook \*\*\* find KEYFILE\_PATERN line, and edit to look like KEYFILE\_PATTERN=/boot/initial\_password \*\* update-initramfs -uk all \*\* reboot \*\*\* should boot without needing to type luks password \*\* shutdown \*\* snapshot \*\* boot \*\* log in as root \*\*\* emacs /etc/network/interfaces \*\*\* userdel debian \*\*\* rm -fr /home/debian \*\*\* passwd -d root \*\*\* passwd -l root \*\*\* fstrim -va \*\*\*\* comment out the lo and eth parts

\*\* shutdown \*\* on hardware tab, \*\*\* add cloudinit drive \*\*\* set NIC to down \*\* on cloud-init tab \*\*\* user: alain \*\*\* SSH public key: paste in public key \*\* snapshot d \*\* Test \*\*\* connect network \*\*\* grow disk \*\*\* power on \*\*\* Disk won't grow automatically (ansible role does it). \*\*\* Check that user login works \*\* if it is all good \*\*\* revert to most recent snapshot \*\*\* delete snapshots \*\*\* convert to template
