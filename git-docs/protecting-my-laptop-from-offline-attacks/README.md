# Protecting My Laptop From Offline Attacks

This is a set of pages about various security changes meant to secure my laptop from offline attacks like the classic "evil maid".  Not all of these are right for all situations, and I only really did it as a learning exercise, I don't really think I'm likely to be attacked in a way that makes these changes mandatory. &#x20;

## Threat Models

It's sometimes easier to focus this sort of discussion by starting with some examples of the sort of access you are trying to prevent. So here's a few, some of which are here as examples of access methods that won't be addressed.&#x20;

### Schenario 1&#x20;

Someone sees your computer sitting there. They power on, and start snooping around your files.&#x20;

### Scenario 2&#x20;

For some reason they weren't able to get to your files above (password or something), so they boot a linux install disk, and start snooping around your files.&#x20;

### Scenario 3&#x20;

They can't get it to boot their flashdrive, so they pull the SSD out, plug it into another machine, and start snooping.&#x20;

### Scenario 4&#x20;

They can't access the files for some reason, so they inject some malware that should give them access later.&#x20;

### Scenario 5&#x20;

Instead of malware, they add hardware for something like keylogging.

## Protections

So what can you do? Let's make a list. You don't need to do all of these, decide what is write for you.

1. Set the BIOS password \
   This is simple and usually can either block access to BIOS settings, or even block booting all together. Doesn't protect the files on your drive since it can be moved to another machine.\
   \- Possible problems: Not the greatest protection. Like locking the doorknob but not the deadbolt.
2. Encrypt important files Lots of ways to implement this (encrypted home with cryptfs for example). - Possible problems: \
   It's way too easy to accidentally put an important files in a location that isn't encrypted. \
   If you don't encrypt the system it would be relatively easy for someone to modify the system to get future access.
3. Encrypt the entire drive (LUKS volume for / and /home and swap, etc).\
   Protects the OS files from offline tampering. \
   \- Possible problems: This isn't actually encrypting everything, the boot loader, and usually the kernel and initrd aren't encrypted.  You need to type the password at boot. &#x20;
4. Use hardware like a yubikey as to unlock the LUKS volumes from above. [Link](root-in-luks-volume.md)\
   Extra protections from password guessing/keylogger stealing the password, etc.  \
   \- Possible problems: Yubikeys aren't free, and you should probably have a backup key stored in a safe place or with a trusted friend.  \
   \- You might want to have an extra password in case you lose your yubikey.   &#x20;
5. Validate the early-boot environment ([Measured boot with TPM](measured-boot-with-tpm.md))\
   This one is a bit complicated. One of the functions the TPM is meant to provide is called Platform Integrity or Measured Boot. The TPM vouches for the early boot not being tampered with (new EFI modules, boot loaders, etc.).   \
   \- Possible problems: If someone finds a way to tamper with a TPM, they could break this. \
   \- You can't purely count on software to just say good or bad, because the software might have been tampered with to show "good" state when it isn't.  See the page on this for more
6.  Make Secure Boot work for you \
    Most hardware ships with secure boot configured to trust any file signed by a certificate signed by Microsoft. Microsoft has full control of this process, and might sign something you would call malware.  \
    Instead of this "one key to rule them all" system, we can remove the Microsoft certificate and make one of our own. Then we decide what is signed. Anyone who doesn't have the private key for our certificate should have a very hard time getting anything to boot on our system.\
    \- Possible problems: This can be complicated, and can easily break your system from booting after installing updates. &#x20;

    This can make it harder to fix your system from other problems.

Some of these options are covered in their own pages here. &#x20;
