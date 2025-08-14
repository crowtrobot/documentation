---
description: Lets bake in the clevis stuff for a more fancy template
---

# Template v2.0 With LUKS Built-in

So, that trick of having clevis automatically unlock the LUKS was actually pretty easy, as the documentation said it would be. So I had to try to think of some way to make it harder, and here is what I came up with.  :smile:  I will make this all a bit harder by making an ansible role to set up the LUKS password from clevis and tang. So first I would need a test case. I rebuilt my basic debian stable VM with LUKS encryption.

## The VM's setup

So now I have a VM template with / mounted from luks, using the file /boot/initial\_password as the LUKS password. This is just temporary password that will be replaced by an ansible role.  So when I deploy a VM from this template I don't need to open the console and type in a password. Other than that, it's basically the same small debian installation (no reason this couldn't work with others, I just prefer Debian). &#x20;

## Security concerns?

If anyone is actually reading this, they are probably wondering about the security problems inherent in having the same initial\_password file for all VMs, and having all of the VMs encrypted with the same key.  Those concerns are actually addressed in this plan, but it's going to take a while to explain. &#x20;

Here it feels appropriate to take a tangent to describe a little bit of how LUKS works. When you `cryptsetup luksFormat` a partition, it creates a header section at the beginning of that partition, it generates a random encryption key for the volume, encrypts that key with the password you provide and stores that encrypted form of the key in one of the keyslots in that header. The idea is that you can have more than one password, and the ability to change the password(s) without needing to completely read and re-encrypt all the data on the drive.  So normally if you change the password, the actual encryption key is the same, you have just re-encrypted it with a new password.

But that brings us to `cryptsetup reencrypt` which actually does that step where it reads in the data with the old key and stores it again with a new key.  Why is this important? Well without that every VM I make by cloning that VM template will have the same encryption key. And since the template itself has the initial\_password file, it would be basically trivial for someone to get the actual encryption key from that, and use it to decrypt the LUKS volume of another VM.

## The extra deploy steps

So now I can make a VM from the new template, and just like before use cloud-init to tell it what IP to have and what ssh keys to accept. Then I can use a new ansible role to do these steps:&#x20;

* Install clevis, clevis-luks, clevis-initramfs, and clevis-systemd if they aren't already installed in the template
* Figure out what the root file system device is, for example /dev/mapper/root-crypt, which is contained in /dev/sda2.
* If /boot/initial\_password still exits, re-encrypt with `cryptsetup reencrypt /dev/sda2 -d /boot/initial_password` .  &#x20;
  * Since there is still only that initial\_password, we will use it to tell cryptsetup to reencrypt. This takes less than 5 minutes on fast storage with the template's 8GB disk.
* Add the key from tang with  `clevis luks bind -d /dev/sda2 -s8 -k /boot/initial_password -y tang '{"url":"http://192.168.68.254/"}'`
  * This step uses the inital\_password file to get the actual encryption key, encrypt it with the password from tang, and store that in keyslot 8.
* Add the fallback password for LUKS.  This is a password that can be used to unlock the LUKS volume if there's ever any problem with clevis.  `cryptsetup luksChangeKey –key-file /boot/inititial_password`&#x20;
* `rm /boot/initial_password`&#x20;
  * since it isn't needed anymore&#x20;
* modify /etc/crypttab to not expect to be able to unlock with /boot/initial\_password anymore with `sed -i 's/\/boot\/initial_password/none/g'`
* `update-initamfs -u -k all` To update initramfs so it has the new crypttab, clevis, and all that.
* And do reboot just to prove that it works before going on to finish setting up the VM.

## What the extra steps look like in the ansible role

This text should become a link to the separate page where I have the role with some annotation on what it is doing.&#x20;

## Conclusion

And that's it. After that I can just set up the VM with other roles like before. The only things I've lost by having this encryption are:&#x20;

* disk I/O is theoretically slower since it needs to be encrypted (I haven't been able to actually measure a difference, so I call this negligible)&#x20;
* cloud-initramfs-growroot doesn't work with LUKS, so I will have to make an ansible role or a script that can grow the root filesystem.&#x20;
  * This is actually pretty easy, just need to:&#x20;
    * `growpart /dev/sda2`&#x20;
    * `clevis luks pass -s 8 -d /dev/sda2 | cryptsetup resize /dev/mapper/root-crypt`
      * get the password from clevis and feed it to cryptsetup resize.  &#x20;
    * `xfs_growfs /`&#x20;
      * or whatever is right for the filesystem if it isn't xfs&#x20;
  * Backups of 2 similar VMs deployed from the same template don't have any duplicate data to deduplicate, since they are encrypted with different keys.  I have enough disk space this isn't a concern in my case. &#x20;
