---
description: This is for the template without LUKS on the root volume
---

# The Plain Template

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
  * Make a new 2G partition at the beginning of the drive, and select swap area for "use as". &#x20;
  * Make another new primary partition using the rest of the disk.
    * Use as xfs or ext4 (whichever you prefer)
    * Set the mount options to relatime and discard
    * Set the label to "root"
    * Set the "bootable" flag on
  * Choose "Finish partitioning and write changes to disk"
* We only need the ssh server and standard system utilities package sets.
* Install grub as normal, and finish the installation.

When it has finished with the shutdown and starts to boot up again, shutdown the VM. We will make some more changes before letting it boot the first time. The changes are:

* Remove the CD Drive from hardware.
* Take a new snapshot so if something goes wrong later you can try again from here.

Start the VM, and let it boot up for the first time. You can log in locally, or ssh in as the user "debian" we created earlier. Ssh does make it easier to copy-and-paste commands. Once logged in run these commands:

* `su -` you will need to put in the root password. You can't ssh in directly to the root user account, and sudo isn't installed yet.
* `apt update`
* `apt dist-upgrade`
* optional: install some utilities that might make the rest of this just a bit easier: `apt install emacs-nox`
* `apt install sudo qemu-guest-agent cloud-init cloud-initramfs-growroot netplan.io systemd-resolved`
* `apt clean`
* `rm /etc/network/interfaces`

Now log out.  This part can't be done in ssh, go to the VM console, log in as root, and run these commands.  This will leave the VM with no users, and root not allowed to log in.  If this works, a user will be created by cloud-init when a VM is deployed from this template. &#x20;

* `userdel debian` (if this fails, reboot and try again)
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
  * Set to ip to DHCP
* One final snapshot. After testing the VM we will revert to this one.

To test, we will grow the VM disk, and boot the VM. SSH in as your user with your ssh key, and `df -h` to check that the filesystem was grown.&#x20;

Also test with the cloudinit set to a static IP and make sure that works (that it can set the VM's DNS settings). &#x20;

If it all looks good, shutdown the VM and revert to the last snapshot. Then delete all snapshots and convert the VM into a template, and you are done. &#x20;
