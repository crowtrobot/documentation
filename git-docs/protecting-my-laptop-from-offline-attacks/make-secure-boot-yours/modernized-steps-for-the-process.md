---
description: >-
  The same basic process as in the parent page, but using some more modern tools
  to make it easier
---

# Modernized Steps For The Process

This time this is with CachyOS because I have to keep changing things so I have problems to fix. &#x20;

## Step 1 - Make the certificates

We need to make the keys and certificates with the PK signing the KEK, and that signing the DB, which will then sign our kernels.  Getting this right took a lot of steps before, but now it's just one command!

```
sbctl create-keys
```

Note that you might need to install sbctl if it isn't already installed. &#x20;

```
pacman -S sbctl
```

## Step 2 - Create the PK, KEK, and DB

This was done automatically by sbctl, so this step just exists now to keep the steps lined up with the old ones. &#x20;

## Step 3 - Add the keys to EFI

Usually the EFI won't let you add certificates unless you turn of secure boot and turn on Setup Mode.  Some EFIs have a different process where you submit the key, then boot into the setup screen to approve adding them. &#x20;

The sbctl tool seems to know what it is doing, and I think it will tell you if you need to do something.  For example, some systems have EFI optional add-ons that are signed with the Microsoft keys, which sucks.  On these systems you have to add the Microsoft keys to the DB which kinda ruins some of the points of doing this process.  If you need that, add the "-m" switch. &#x20;

```
sbctl enroll-keys
```

## Step 4 - Sign the bootloader and kernel

The easiest way to do this, is with systemd-boot as the bootloader.  What's nice is with my install of CachyOS, systemd-boot was the default, so yay! &#x20;

So lets sign the bootloader

```
sudo cp /boot/EFI/BOOT/BOOTX64.EFI /boot/EFI/BOOT/BOOTX64.EFI.unsigned
sudo sbctl sign /boot/EFI/BOOT/BOOTX64.EFI
```

Great, now we need the kernels signed.  We want to use UKI (unified kernel image) format kernels so that the initrd and everything else for early boot is getting signed. &#x20;

```
sudo nano /etc/kernel/install.conf
```

And add this line

```
layout=uki
```

Make sure that the right kernel command line options are getting used:

```
emacs /etc/kernel/cmdline
```

And now to re-install the kernels so they get signed

```
kernel-install add-all
```

## Steps  5, 6 and 7 aren't needed

The old step 5 to switch to installing kernels as UKI images, but we did that as part of step4.  Step 6 and 7 were about automating the signing, but because this is being handled automatically by kernel-install, it is already automated.  &#x20;
