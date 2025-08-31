---
description: Bringing secure boot back with the ZFS root setup.
---

# Making Secure Boot Work Again

Based on the work in [Make Secure Boot Yours](../protecting-my-laptop-from-offline-attacks/make-secure-boot-yours.md), but with some simplifications because there's new tools in systemd to help. &#x20;

## Make keys and certs and add them to EFI

This is the same as we did in  [Make Secure Boot Yours](../protecting-my-laptop-from-offline-attacks/make-secure-boot-yours.md).

## Sign systemd-boot

Also the same as we did in  [Make Secure Boot Yours](../protecting-my-laptop-from-offline-attacks/make-secure-boot-yours.md).

## Configure ukify to sign

Edit the /etc/kernel/uki.conf&#x20;

```
[UKI]
SignKernel=yes
SecureBootPrivateKey=/etc/kernel/secure-boot-key.pem
SecureBootCertificate=/etc/kernel/secure-boot-certificate.pem
SecureBootSigningTool=sbsign
```

Copy those keys in:

```
cp /etc/sb_keys/db.key /etc/kernel/secure-boot-key.pem
cp /etc/sb_keys/db.crt /etc/kernel/secure-boot-certificate.pem
```

And then we will rebuild the kernel image to get signed kernels installed

```
update-initramfs -ck all
```

And it should work. &#x20;
