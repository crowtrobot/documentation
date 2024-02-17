# Root in LUKS Volume

It seems I lost the page about installing with the root volume in LUKS. A lot of distributions offer as an option during installation, so the page was a bit outdated anyway.

See [https://github.com/cornelinux/yubikey-luks](https://github.com/cornelinux/yubikey-luks) for an easy tool for using a yubikey to unlock your LUKS volume on boot. It would be a good idea to have a backup key, or a long backup password stored safely in case something happens to your yubikey.

If you are using something like Arch or Fedora that uses dracut to generate the inird, it's probably going to be easier to use systemd-cryptenroll to let something like a yubikey or TPM be used for unlocking LUKS.  On Debian and Ubuntu derivatives this doesn't work yet (unless you switch to dracut, which seems like a viable option).   You can use systemd-cryptenroll to add the key to the a keyslot of your LUKS, but these distributions don't use systemd-cryptsetup in the initrd, so the unlocking won't work until that's been resolved. &#x20;
