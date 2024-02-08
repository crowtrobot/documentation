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

There is no defense here for the personal touch of rubber-hose cryptanalysis.

With patience and prolonged access someone might copy a kernel, and keep it for a while until a sufficiently bad vulnerability is found in it, then load that old kernel back into the system and boot with it. This seems unlikely, but if you don't like that option existing, you could invalidate each old kernel by adding its hash to the dbx (the blocked list, files matching the hash or signing certificate listed in the DBX are not trusted, even if validly signed by a trusted cert).

Depending on how your system boots, you might need to take steps to sign the boot loader (for example grub) and the kernel. Maybe you also need to configure the boot loader to validate the signature on the kernel, or maybe not. I'm running Pop!\_OS and it doesn't use grub, but instead uses kernelstub to allow the EFI to directly boot a kernel.

You should take some care to make sure the initrd is also validated. For this doc I will pack the initrd into the same file as the kernel and it will get validated by the same signature.

## How Secure Boot works

Sources/references:\
https://ubuntu.com/blog/how-to-sign-things-for-secure-boot\
https://wiki.debian.org/SecureBoot https://wiki.gentoo.org/wiki/User:Sakaki/Sakaki%27s\_EFI\_Install\_Guide/Configuring\_Secure\_Boot

There's 4 main parts to secure boot (and an optional 5th):

1. The PK validates the KEK.\
   The PK (platform key). This certificate has to sign any updates to the other parts once it exists. This is the machine's root cert. The EFI will only accept the KEK if it was signed by this cert (usually this is the hardware vendor's key, but in this doc we will make our own)
2. The KEK which validates the db and dbx\
   The KEK (key exchange key). This Certificate must sign anything being updated in db or dbx. The KEK can sign executables, but usually it signs certs that then sign executables.
3. The DBX is a blacklist of things that aren't accepted, even if otherwise validated.\
   The dbx is the forbidden signatures database. Revoked certs and hashes of forbidden binaries goes here.
4. DB is which validates the bootloader, or shim.\
   The signature database (the db). This db can have certificates, and hashes of files. Basically something is ok for use if it's hash is in this DB, or if it is signed by a cert in db, and isn't in the DBX.
5. If using the shim, it might have it's own MOK which validates the next boot loader it loads.\
   Optional MOK - The machine owner's key. I think this is only in secure boot shim. This is the DB of certs that the shim will accept. So the shim can be signed by Microsoft, and then it can check that your boot loader or kernel is signed by a cert it trusts (for example, ubuntu can use their cert to sign their kernels without needing Microsoft to sign them, because Microsoft signed the shim).

Secure boot (when enabled), makes sure the machine will only load EFI executables which fit one of these:

* are unsigned, but have a hash in db and not also in dbx
* or are signed, where that signature can be verified to come from a cert in db, and does not not an entry in dbx
* or are signed, where that signature is verifiable by a public key in db, or a public key in KEK, and where neither that key, not the signature itself, appears in dbx.

## Why not just use shim

That's probably fine most of the time, but I did want to dig into the depths here. And how can it claim the "secure" part of the "Secure Boot" name if it allows windows to be booted? :smile:

## The process

Be careful through this. There will be a part where the EFI will only accept signed kernels, but you won't have any kernels signed yet. If you have to reboot during that time you will need to turn Secure Boot off in your EFI setup. Some systems also won't let you put Secure Boot into setup mode while Secure Boot is on, but oddly there also seem to be some that require it to be on.

### Step 1 - Make the certificates.

First, make the certs:

```
openssl req -new -x509 -newkey rsa:2048 -subj "/CN=My own platform key/" -keyout PK.key -out PK.crt -days 3650 -nodes -sha256 
openssl req -new -x509 -newkey rsa:2048 -subj "/CN=My own key-exchange-key/" -keyout KEK.key -out KEK.crt -days 3650 -nodes -sha256 
openssl req -new -x509 -newkey rsa:2048 -subj "/CN=my own kernel-signing key/" -keyout db.key -out db.crt -days 3650 -nodes -sha256 
openssl req -new -x509 -newkey rsa:2048 -subj "/CN=my own dbx key/" -keyout dbx.key -out dbx.crt -days 3650 -nodes -sha256
```

These certs don't seem to work for signing kernel modules. I tried adding the oid that marks a cert as module signing only, and adding another cert to the DB. It seems my kernel doesn't want to use certs it reads from db for module signing. The cert is in the .platform keychain, but isn't being used for modules. I found that of the 3 kernel modules I have from DKMS, one won't load at all, and the other 2 load and do nothing because they don't apply to my hardware (they do things for other System76 computers like enable fan and LED controls). So, since I don't need to load any DKMS modules, I didn't figure out how to get them signed so they will load.

### Step 2 - Make a new PK, KEK, and db

Then we need to make the .auth file that the BIOS expects. This is some special format for the certificates, I think it is just a list of certs signed by a cert. It needs a unique ID, but it doesn't matter too much what that ID is.

```
cert-to-efi-sig-list -g "$(uuidgen)" PK.crt PK.esl
```

Then I think this command signs that file, using the key and cert for the PK.

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

And once more for the DBX (if you don't make one, you can't later add things to it).

```
cert-to-efi-sig-list -g "$(uuidgen)" dbx.crt dbx.esl 
sign-efi-sig-list -a -k KEK.key -c KEK.crt dbx dbx.esl dbx.auth
```

If you look around the internet there are steps for exporting the original DBX so you can import it into your new one. I didn't bother. If I encounter malware old enough to be known to the DBX that came with the computer, but I haven't signed it with my certificate, it will be just as un-trusted as it would be if it was in the DBX. I just need to be careful not to sign suspicious things.

Some BIOSs want the certificates in .der format, but I'm going to skip those steps here because I don't think I will need that.

### Step 3 - Add these keys to the EFI

I went into the EFI setup, to the secure boot settings, and put it in the setup mode (which clears the keys so you can add new ones). I was going to load in with the BIOS UI, but it didn't like the files on my flash drive. Turns out it did want the .der files for the certs. I'll just load from inside Linux instead

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

I think `efi-updatevar -e -f dbx.esl dbx` would normally also make a dbx, but it seems my system won't let me do a dbx when running in custom mode. I can restore the factory default dbx, but I can't update it or add to it. TODO: Need to look into this some more, it seems like I should be able to add to the DBX.  This did work with another PC. &#x20;

The -e says that these are signed with this key, but since we are in setup mode we don't need these things to be signed, so can leave it blank, I guess. Anyway, now we load the PK.\
`efi-updatevar -f PK.auth PK`

This should take us out of setup mode because there's now a PK.

So lets verify. We will dump out these values and compare them:

```
efi-readvar -v PK -o new_PK.esl 
efi-readvar -v KEK -o new_KEK.esl 
efi-readvar -v db -o new_db.esl 
diff -s new_PK.esl PK.esl 
diff -s new_KEK.esl KEK.esl 
diff -s new_db.esl db.esl
```

It should say that each of those file we pulled out of the EFI is identical to the ones we loaded into the EFI.

### Step 3 - Sign the boot loader, and kernel

For this we will try `pesign`. It seems there are a few tools that can be used, but this is the one that seems simplest. First, we need to put the db cert into a pkcs12 file, and then import it.

```
openssl pkcs12 -export -out db.p12 -inkey db.key -in db.crt pk12util -i db.p12 -d /etc/pki/pesign
```

Then we can sign the boot loader:

```
cd /boot/efi/EFI/systemd 
mv systemd-bootx64.efi systemd-bootx64.efi.unsigned 
pesign -i systemd-bootx64.efi.unsigned -o systemd-bootx64.efi -s -c "my own kernel-signing key"
```

There's not enough space in my EFI partition for the signed and unsigned kernels to both exist, so I will replace the "previous-version" kernel with the unsigned.

```
cd ../Pop_OS-deaccdaa-8769-4f11-b906-e3038f03d6ce/ 
mv vmlinuz-previous.efi ~ 
mv vmlinuz.efi vmlinuz-previous.efi 
pesign -i vmlinuz-previous.efi -o vmlinuz.efi -s -c "my own kernel-signing key"
```

### Step 4, Pop!\_OS specific parts, signing kernels

So, Pop!\_OS uses a thing called kernelstub to make the kernel into an executable that EFI can load directly, or systemd-boot can load. It looks like they use systemd-boot to give you a menu to pick current, previous kernel, or recovery. If the kernel is not signed you get a red "access denied" error. The kernel isn't signed until I signed it above, but still the initrd is in the unencrypted EFI partition (the ESP) and it isn't signed. So Some extra steps are needed here to protect the initrd part of boot.

https://wiki.archlinux.org/title/Unified\_kernel\_image#Preparing\_a\_unified\_kernel\_image

The kernel, and initrd and some extra bits (boot splash screen, kernel cmdline, etc) can all be bundled up into a single file, and then that entire file signed. This way if someone tries to modify boot options, or replace something in initrd, the signature becomes invalid and the system doesn't boot.

So, to do this I will use the ukify program that is part of the systemd tools. Unfortunately it was added in a newer version than what I'm using. But since it is a relatively simple looking python script with only a few dependencies, it isn't hard to just download and use. So, I downloaded it from the github page into /root/bin, and installed the dependency `apt install python3-pefile` .

So with a simple-ish script the kernel and initrd from /boot (don't trust files coming from outside of the encrypted root if we don't need to) get bundled into an efi loadable file. This replaces the not-signed old version, and replaces the initrd with an empty file.

I've added the module.sig\_enforce=1 to tell the kernel to only accept signed modules (the kernel has it's own trusted cert and the modules come signed by the distribution).

The process basically looks like this:

```
./ukify.py build --linux /boot/vmlinuz-6.2.6-76060206-generic --initrd /boot/initrd.img-6.2.6-76060206-generic --cmdline "root=UUID=deaccdaa-8769-4f11-b906-e3038f03d6ce ro module.sig_enforce=1 loglevel=0 systemd.show_status=false splash resume=UUID=8aa04cb4-a9c3-450c-81d8-8f7b97c1df74" --signtool pesign --secureboot-certificate-name "my own kernel-signing key" --sign-kernel --output /tmp/kern
```

### Alternate Step 4 - add the hash of the kernel file to the DB so it is bootable

Instead of verifying the signature, secureboot could just verify that the hash matches on in DB (and doesn't match anything in DBX).

```
efi-updatevar -a -b /boot/efi/EFI/Pop_OS-deaccdaa-8769-4f11-b906-e3038f03d6ce/vmlinuz-previous.efi -k KEK.key db
```

I thought it was supposed to be just as easy to add a file to the DBX to prevent it being accepted, even if it is validly signed, but it seems my system won't let me update the DBX at all. I either get the factory one or none. But since I can just remove entries from db instead, I guess that's ok.

Removing something from db is as easy as `efi-updatevar -d 1 -k KEK.key db` . That's to remove the second entry (the first being 0, which in my case is the db or kernel signing cert).

If you are going to add in checking the TPM PCRs to help secure boot, you will find it more difficult when working this way, because the secure boot stuff (PK, KEK, and db) are all part of at least one of the PCRs, so every change to the db would change that PCR.

### Step 5 - Verify

Try removing the signature from the kernel and make sure that the system won't boot it. Maybe also sign with the wrong cert to verify it still isn't booted. To remove a signature:

```
pesign -r -i /boot/efi/EFI/Pop_OS-deaccdaa-8769-4f11-b906-e3038f03d6ce/vmlinuz.efi -o /tmp/kern
mv /tmp/kern /boot/efi/EFI/Pop_OS-deaccdaa-8769-4f11-b906-e3038f03d6ce/vmlinuz.efi
```

### Step 6 - Additional security measures

After I had this all working, a friend sent me this link. https://blastrock.github.io/fde-tpm-sb.html

That's much better written than my crap here, and applies to Debian where this was specific to Pop!\_OS. But the author of that did point out one part I missed. I should be setting "panic=0" in the boot cmdline. So, "`sudo emacs /etc/kernelstub/configuration`", and add a line in the user section:

```
"user": { "kernel_options": [ 
    "resume=UUID=8aa04cb4-a9c3-450c-81d8-8f7b97c1df74", 
    "systemd.show_status=true", 
    "module.sig_enforce=1", 
    "splash", 
    "loglevel=0" ],
...
```

Conclusion So, I have secure boot working to only allow booting kernels I've signed. Kernels installed by apt will be automatically signed. The initrd is packed into the same file, so it is also protected from tampering by that same signature.

The next step I think will be to change the LUKS key so it's mixing in a pin, yubikey, and TPM2 registers. Secure boot is supposed to protect the boot code from the EFI partition (the ESP) from tampering, since the key signature won't match the file if it has been modified. Mixing in the boot values from the TPM should validate everything that happened before the kernel was loaded. This should ensure that the firmware hasn't been tampered with, because if it has the number from the TPM register would be different, so we wouldn't get the right key for LUKS, and decrypting the drive would fail.

## Known Problems For This Implementation

If something triggers an update to the initrd (update-initramfs), it will break booting. This will replace the kernel and initrd in the EFI partition with the un-signed versions. I can add a script to /etc/initramfs/post-update.d like I did for the post kernel install, but the script I already wrote would need some changes. Also some testing would be needed to verify that when installing a new kernel I don't replace the files from kernel stub in the initrd stuff, and then fail to install it in the kernel postinstall. Not hard, I just haven't done it yet.

This seems to slightly upset kernelstub, and it might never get the previous kernel set up. It would be better for my script to entirely replace kernelstub I think, but that seems like a lot of work. It looks like sysemd is working on a program called kernel-install that seems like it would replace most of what I've done, and kernelstub. It looks like there's parts that depend on newer systemd that what I've got. I suspect that in the next major release of pop I might be able to switch up to using someone else's tested and better written code. This has been an effective excuse for me to put off the final fixes I was thinking about doing.
