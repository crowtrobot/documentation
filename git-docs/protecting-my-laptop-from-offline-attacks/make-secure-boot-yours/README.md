---
description: Configuring secure boot to use your own certificate instead of Microsoft's.
---

# Make Secure Boot Yours

## Basic goal

I wanted to learn more about secure boot. I knew some of what happens with it, but not as much as I thought I knew. For example, I was surprised to learn that the TPM is related, but isn't really required in secure boot. You can add in security features from the TPM to improve security, Secure Boot is really just about verifying that whatever you boot is signed by a trusted certificate.

Normal secure boot is designed so that parts of the boot process are prevented from running untrusted code. This is enforced by verifying that the file's hash is on the approved list, or that it is signed by a CA that is approved. Unfortunately this seems to have created a scenario where if someone pays for a signature that doesn't really mean that the software is secure \[[1](https://thehackernews.com/2022/08/researchers-uncover-uefi-secure-boot.html)]. Or they can find a vulnerability in a trusted program (maybe one of the known vulnerabilities MS has patched, but not yet blacklisted for whatever reason \[[2](https://arstechnica.com/information-technology/2023/03/unkillable-uefi-malware-bypassing-secure-boot-enabled-by-unpatchable-windows-flaw/)]). Or if they try hard enough to find online the private key for gigabyte motherboards that recently leaked. Or just get Microsoft to directly sign the malware \[[3](https://arstechnica.com/gadgets/2021/06/microsoft-digitally-signs-malicious-rootkit-driver/)].

1 https://thehackernews.com/2022/08/researchers-uncover-uefi-secure-boot.html

2 https://arstechnica.com/information-technology/2023/03/unkillable-uefi-malware-bypassing-secure-boot-enabled-by-unpatchable-windows-flaw/

3 https://arstechnica.com/gadgets/2021/06/microsoft-digitally-signs-malicious-rootkit-driver/

So, The plan was to sign my boot files with my own cert. This should really limit problems with evil-maid attacks, or similar situations. The root filesystem is encrypted so pulling the disk out shouldn't let an attacker access my files, and with secure boot they should find it very difficult to get anything they might leave behind booted since their files shouldn't be signed by my key, and they don't know the LUKS key to store the files inside my filesystem.

This obviously does nothing for remote attacks, and with enough persistence could be bypassed locally, but I think it should be hard enough to get my key that it is just impractical, making other avenues of attack become the easier options. And believe me, my system isn't prepared for any rubber-hose cryptanalysis.

Specifically this should limit options to boot software on my machine to specific, individually blessed kernels. Windows installers, Linux boot disks, and other similar options should simply not work since they have not been signed by my certificate.

## Weaknesses/Possible Problems

* Your computer must be booting in EFI mode.  There might be a way to make secure boot work without booting in EFI mode, but I didn't try. &#x20;
* There is no defense here for the personal touch of rubber-hose cryptanalysis.  It might be possible to set this up with something like a VeraCrypt hidden volumes, or some similar plausible deniability mechanism, but that's far beyond the scope of the goals of this document (but maybe this is a good starting to build from?  If you create a guide that extends this, I would like to read it). &#x20;
* With patience and prolonged access someone might copy a kernel, and keep it for a while until a sufficiently bad vulnerability is found in it, then load that old kernel back into the system and boot with it. This seems unlikely, but if you don't like that option existing, you could invalidate each old kernel by adding its hash to the dbx (the blocked list, files matching the hash or signing certificate listed in the DBX are not trusted, even if validly signed by a trusted cert).  Or just replace your certificate periodically, which would make all the old kernels signed with the old one invalid. &#x20;
* Depending on how your system boots, you might need to take steps to sign the boot loader (for example grub) and the kernel. Maybe you also need to configure the boot loader to validate the signature on the kernel, or maybe not. I switched to using systemd-boot as the boot loader, which requires very little configuration and can't load anything without a valid signature when secure boot is on. &#x20;
* You should take some care to make sure the initrd is also validated. For this I will pack the initrd into the same file as the kernel and it will get validated by the same signature.
* Any kernel modules you want to load will also need to be signed.  I will add another document about how I got DKMS to sign the nvidia driver one of my test machines used.  (TODO - Link here when that has been written)

## How Secure Boot works

Sources/references:\
https://ubuntu.com/blog/how-to-sign-things-for-secure-boot\
https://wiki.debian.org/SecureBoot https://wiki.gentoo.org/wiki/User:Sakaki/Sakaki%27s\_EFI\_Install\_Guide/Configuring\_Secure\_B

<figure><picture><source srcset="../../.gitbook/assets/sb_dark.png" media="(prefers-color-scheme: dark)"><img src="../../.gitbook/assets/sb_light.png" alt="" width="563"></picture><figcaption><p>Diagram of the relationship between certificates in Secure Boot</p></figcaption></figure>



There's 4 main parts to secure boot (and an optional 5th and 6th):

1. The PK validates the KEK.\
   The PK (platform key). This certificate has to sign any updates to the other parts once it exists. This is the machine's root cert. The EFI will only accept the KEK if it was signed by this cert (usually this is the hardware vendor's key, but in this doc we will make ourh own)
2. The KEK which validates the db and dbx\
   The KEK (key exchange key). This Certificate must sign anything being updated in db or dbx. The KEK can sign executables, but usually it signs certs that then sign executables.
3. The DBX is a blacklist of things that aren't accepted, even if otherwise validated.\
   The dbx is the forbidden signatures database. Revoked certs and hashes of forbidden binaries goes here.
4. DB is which validates the bootloader, or shim.\
   The signature database (the db). This db can have certificates, and hashes of files. Basically something is ok for use if it's hash is in this DB, or if it is signed by a cert in db, and isn't in the DBX.
5. If using the shim, it might have it's own MOK certificate(s) which validates the next boot loader it loads.  Optional MOK - The machine owner's key.  This is the set of certs that the shim will accept. The shim is signed by Microsoft, and then it can check that your boot loader or kernel is signed by a cert it trusts (for example, Ubuntu can use their cert to sign their kernels without needing Microsoft to sign them, because Microsoft signed the shim).  Adding certificates to the MOK can be done without needing Microsoft to sign them.  Like a second-class level of secure boot. &#x20;
6. If using shim, it might load an MOKX, which is the block-list for the machine owner key.  This seems to be rarely used, since it is pretty easy to replace a certificate in the MOK. &#x20;

Secure boot (when enabled), makes sure the machine will only load EFI executables which fit one of these:

* file is unsigned, but its hash is in db and not also in dbx
* file is signed, where that signature can be verified to come from a cert in db, and does not match an entry in dbx
* file is signed, where that signature is verifiable by a public key in db, or a public key in KEK, and where neither that key, not the signature itself, appears in dbx.

Or after the shim has loaded (the shim itself must match the above rules), these rules are added to the above:

* file is signed by a certificate in the MOK and does not match an entry in MOKX

## Why not just use shim

That's probably fine most of the time, but I did want to dig into the depths here. And how can it claim the "secure" part of the "Secure Boot" name if it allows windows to be booted? :smile:  If I'm going to learn how this all works in depth, I might as well go far enough that nothing but things I've approved will boot. &#x20;

## The process

These steps have been somewhat simplified with new tools.  If you are running something newer, check out the newer process [here](modernized-steps-for-the-process.md).

Be careful through this. There will be a part where the EFI will only accept signed kernels, but you won't have any kernels signed yet. If you have to reboot during that time you will need to turn Secure Boot off in your EFI setup. Some systems also won't let you put Secure Boot into setup mode while Secure Boot is on, but oddly there also seem to be some that require it to be on. &#x20;

### Step 1 - Make the certificates.

First, make the certs we will need.  Create a safe place for these files, like a directory that only root has access to, and cd there.  Then run:

```
openssl req -new -x509 -newkey rsa:2048 -subj "/CN=My own platform key/" -keyout PK.key -out PK.crt -days 3650 -nodes -sha256 
openssl req -new -x509 -newkey rsa:2048 -subj "/CN=My own key-exchange-key/" -keyout KEK.key -out KEK.crt -days 3650 -nodes -sha256 
openssl req -new -x509 -newkey rsa:2048 -subj "/CN=my own kernel-signing key/" -keyout db.key -out db.crt -days 3650 -nodes -sha256 
openssl req -new -x509 -newkey rsa:2048 -subj "/CN=my own dbx key/" -keyout dbx.key -out dbx.crt -days 3650 -nodes -sha256
```

### Step 2 - Make a new PK, KEK, and DB

Then we need to make the .auth file that the BIOS expects. This is some special format for the certificates, I think it is just a list of certs signed by a cert. It needs a unique ID, but it doesn't seem to matter too much what that ID is.

```
cert-to-efi-sig-list -g "$(uuidgen)" PK.crt PK.esl
```

Then sign that file, using the key and cert for the PK.

```
sign-efi-sig-list -k PK.key -c PK.crt PK PK.esl PK.auth
```

And then the same to make the file for the KEK (signed by the PK)&#x20;

```
cert-to-efi-sig-list -g "$(uuidgen)" KEK.crt KEK.esl
sign-efi-sig-list -a -k PK.key -c PK.crt KEK KEK.esl KEK.auth
```

And the same again for the db&#x20;

```
cert-to-efi-sig-list -g "$(uuidgen)" db.crt db.esl 
sign-efi-sig-list -a -k KEK.key -c KEK.crt db db.esl db.auth
```

And once more for the DBX.  This is not strictly needed since we control all aspects of these certificates, but it will be harder to add one later, so might as well do it now. &#x20;

```
cert-to-efi-sig-list -g "$(uuidgen)" dbx.crt dbx.esl 
sign-efi-sig-list -a -k KEK.key -c KEK.crt dbx dbx.esl dbx.auth
```

If you look around the internet there are steps for exporting the original DBX so you can import it into your new one. I didn't bother.  If I don't specifically sign it, then it won't be trusted.  I hope it goes without saying that you shouldn't sign suspicious files.  &#x20;

Some BIOSs want the certificates in .der format.  On my test machines I was able to load these files into EFI from within Linux instead of using the EFI setup screen, so I didn't need these .der files.  So I don't have tested steps to create them.  If your EFI is broken enough to only allow adding them from setup, you will need to figure out how to use openssl to make the .der files. &#x20;

### Step 3 - Add these keys to the EFI

I went into the EFI setup, to the secure boot settings, and put it in the setup mode (which clears the keys so you can add new ones).&#x20;

Check that the secure boot variables are cleared by running `sudo efi-readvar`

Which should say:

```
Variable PK has no entries 
Variable KEK has no entries 
Variable db has no entries 
Variable dbx has no entries 
Variable MokList has no entries
```

If that's good, then we will load the secure boot variables.

```
efi-updatevar -e -f db.esl db 
efi-updatevar -e -f KEK.esl KEK
```

One of my test machines wouldn't accept my DBX.  The other worked fine.  I am not too worried about needing the DBX, so I haven't yet taken the time to figure out why that one machine wouldn't accept it.   TODO: Need to look into this some more, it seems like I should be able to add to the DBX. &#x20;

The -e works because we are still in setup mode (we don't have all the key stores loaded yet).  Now we load the last bit, the PK.\
`efi-updatevar -f PK.auth PK`

This should take us out of setup mode because there's now a PK.

So lets verify. We will dump out these values and compare them:

<pre><code>efi-readvar -v PK -o new_PK.esl 
efi-readvar -v KEK -o new_KEK.esl 
<strong>efi-readvar -v db -o new_db.esl 
</strong>diff -s new_PK.esl PK.esl 
diff -s new_KEK.esl KEK.esl 
diff -s new_db.esl db.esl
</code></pre>

It should say that each of those file we pulled out of the EFI is identical to the ones we loaded into the EFI.

### Step 4 - Sign the boot loader, and kernel

I've found it easiest to switch to the systemd-boot boot loader.  I believe there are steps you can take to make this work with grub, or other bootloaders, but I haven't tried.  So far I have tried this on Debian, Ubuntu 22.04 (installed with the KDE Neon install disk), and Pop!\_OS.  For other distributions it might be harder to switch, or require special steps not listed here, or even require that you keep Grub and figure out how to make it work with secure boot. &#x20;

To do the actual signing, I chose `pesign`, and to make that work `apt install python3-pefile`.  There seem to be a lot of options, but this one seems simple enough.  First, we need to put the db cert into a pkcs12 file, and then import it.

```
mkdir /etc/pki/pesign
openssl pkcs12 -export -out db.p12 -inkey db.key -in db.crt
pk12util -i db.p12 -d /etc/pki/pesign
```

Then we can install systemd-boot bootloader if needed.  This was already installed on Pop!\_OS for example, but it shouldn't hurt to do these steps again.  But note that once you start messing with bootloaders, you could leave yourself unable to boot normally.  Best to break here until you are ready to do the rest all in one session (if possible). &#x20;

```
# first make sure /boot/efi is mounted
mount /boot/efi
mkdir -p /boot/efi/loader/entries
```

Edit the  `/boot/loader/loader.conf` file to look like this:

```
default *latest*
timeout 3
editor  no
```

This will make whatever config option has "latest" in the name will be the default option, but the menu will be shown for 3 seconds, and editing the options won't be allowed.  The boot menu options come from conf files in `/boot/loader/entries`, which is empty if you just created it, or contains your distro's supplied files for now.  So lets actually install systemd-boot

```
bootctl install --path=/boot/efi
```

Then run `efibootmgr` to see if the boot loader is fully set up.  This will show a "BootOrder", which is what order the EFI is set to load things from the EFI partition, and bellow that an explanation of what each number means. &#x20;

Now we need to sign the boot loader:

```
cd /boot/efi/EFI/systemd 
mv systemd-bootx64.efi systemd-bootx64.efi.unsigned 
pesign -i systemd-bootx64.efi.unsigned -o systemd-bootx64.efi -s -c "my own kernel-signing key"
```

### Step 5 - The kernel and initrd

Now we need to turn the kernel into an EFI executable.  As part of doing this, we will package up the kernel, the initrd, the command line for the kernel, an optional splash screen image, and an optional microcode update file into a single EFI executable.  This is nice because signing this single file covers all those parts.  For details about this process see [https://wiki.archlinux.org/title/Unified\_kernel\_image#Preparing\_a\_unified\_kernel\_image](https://wiki.archlinux.org/title/Unified_kernel_image#Preparing_a_unified_kernel_image)

Go, get the `prepare-signed-kernels` script from [https://github.com/crowtrobot/sign-kernel-for-secure-boot](https://github.com/crowtrobot/sign-kernel-for-secure-boot) and put it somewhere that makes sense to you (mine is in /root/bin).  If you don't already have ukify.py, you will also need to download that from [https://github.com/systemd/systemd/blob/main/src/ukify/ukify.py](https://github.com/systemd/systemd/blob/main/src/ukify/ukify.py) and put in the same place.  The ukify.py program comes with systemd, but only versions newer than is included in ubuntu 22.04, or Pop!\_OS, or Debian 11, so I had to manually get it for all my test machines.  In the prepare-signed-kernels script, there are some variables you need to set to tell it what the kernel command line should be, where it should find ukify.py, and some other options. &#x20;

Look in your /boot/efi directory for existing kernels.  Pop!\_OS for example will have put some kernels here already, and there isn't a lot of space in the EFI partition.  So I had to `apt remove kernelstub`, and remove the kernels it had installed with `rm -fr /boot/efi/EFI/Pop_OS-deaccdaa-8769-4f11-b906-e3038f03d6ce`.   This made room for the signed kernels. &#x20;

So, finally we can try running `prepare-signed-kernels` .  This script should bundle up, kernels from /boot and their initrds, sign them, put the signed kernels in the efi directory, and add configs for systemd-boot to load them.  If that appears to have worked, it's time to `reboot` and verify. &#x20;

### Step 6 - Automate the signing

Well since that's working and you're now booting, you will want to turn secure boot on and reboot again to make sure everything is working.  Once it's all working, you want to make this happen automatically without you needing to manually tell the system to sign new kernels or updates to the initrd.  To do that we will link to this script from a few directories that run scripts whenever a kernel is installed or removed, or an initrd is modified.  We will call the link zz-prepare-signed-kernels because we want this to run after everything else, and they run in roughly alphabetical order. &#x20;

```
mkdir -p /etc/initramfs/post-update.d
ln -s /root/bin/prepare-signed-kernels /etc/initramfs/post-update.d/zz-prepare-signed-kernels
ln -s /root/bin/prepare-signed-kernels /etc/kernel/postinst.d/zz-prepare-signed-kernels
ln -s /root/bin/prepare-signed-kernels /etc/kernel/postrm.d/zz-prepare-signed-kernels
```

And to quickly test that, try update-initramfs -uk all, and it should update the initrd for all kernels, and then rebuild the kernel bundles, and sign them again. &#x20;

### Step 7 - What about other kernel modules (DKMS)?

This one is optional.  Not all systems need to load kernel modules that didn't come with the kernel.  But if you have an nvidia GPU, or use virtual box for virtualization, you will have kernel modules that didn't come with the kernel.  When secure boot is on, the kernel will only load signed modules.  The modules that come with the kernel will be signed already, but we need to configure DKMS to sign new modules as they are created and installed. &#x20;

To do this, we need the DB key in a format that the DKMS tools expect (the DER format).  DKMS has a built-in mechanism for signing modules it builds, but it assumes that it will be making it's own key and loading it into the MOK.  When the kernel loads, it imports some keys from places like DB and MOK into the "platform" keyring, and these keys can then be used to validate signed kernel modules. &#x20;

Since we aren't using shim we don't have a MOK, but it is ok for DKMS to think we do, because if it thinks it is signing with the MOK key, but it is instead the DB key, the signature comes out the same. &#x20;

```
cd /etd/sb_keys
openssl x509 -in db.crt -out db.der -outform DER
cd /var/lib/shim-signed/mok
mv MOK.der MOK.der.old
mv MOK.priv MOK.priv.old
```

You might not already have a MOK.der and MOK.priv, but if you do we will rename them.  We could probably just delete them. &#x20;

```
ln -s /etc/sb_keys/db.der /var/lib/shim-signed/mok/MOK.der
ln -s /etc/sb_keys/db.key /var/lib/shim-signed/mok/MOK.priv
```

Now reinstall DKMS to get it to rebuild

```
sudo apt install --reinstall nvidia-dkms-545 xserver-xorg-video-nvidia-545
```

It should re-make the modules, and reinstall them.  Then `reboot` and the modules should be inserted and working. &#x20;



#### Some info to help with troubleshooting module signing

To see what keys can sign kernel modules (what keys are in the platform keyring):&#x20;

```
sudo keyctl list %:.platform
```

The name after "asymmetric: " should be your key.  In this doc we used the name "my own kernel-signing key"



To see what signed a module you are looking at (the nvidia-drm module in this case): &#x20;

```
modinfo nvidia-drm
```

The "signer:" line should be your key, "my own kernel-signing key" if you used the name from this doc without changing it. &#x20;

