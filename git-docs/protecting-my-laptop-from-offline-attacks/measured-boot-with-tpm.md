# Measured Boot with TPM

## Background on TPM

If you want to verify a computer is working, it's hard to do that when it requires trusting software running on the suspect machine.  If you don't know if you can trust some computer, you don't know if you can trust the log files on that computer.  For this reason it's common to set up remote logging so you can have some trust that the logs haven't been tampered with later. &#x20;

The TPM (at least TPM 2.0, I don't think the original version 1 TPM has this function) provides an analogous function, in spite of being built into the computer.  The TPM provides a limit set of commands you can give it, designed to limit the ways it's behavior can be tampered with.  Specifically here, this takes the form of the TPM providing a way to read values from PCR registers, and to "extend" them (provide a value to be hashed with the existing value, with the result replacing the existing value), but no way to put a specific value into a PCR. &#x20;

The TPM provides some Platform Configuration Registers.  The EFI hashes each component before it is loaded, and this hash gets submitted to the TPM.  The TPM takes these hashes, and hashes them into previous values to make a sort of running hash that should be very difficult to fake, and should reveal if anything in the chain of modules loaded as part of booting is different. &#x20;

## Verify the boot status

Verifying the value of the TPM register matches what is expected can be tricky. You don't want just a program that says "all good" because that output could be faked, and you don't want to store the PCR value to be validated in a way that could easily be with a new value. So it's probably best if the matching is done off device somehow.

That bring us to a clever little program called `tpm2-totp`. This program stores a totp (time-based one time password) secret in the TPM, "sealed" (encrypted with one of the PCR register's value as the key). The TPM provides mechanisms for such stored keys to do work within the TPM, but doesn't provide a way to read the key back. You can ask the TPM to use this key to calculate the HMAC of the time with this key, but you can't ask it to just give you the key. &#x20;

Early in boot (in the initrd, before you give up your drive's LUKS password if you have one) the PCR value is used to unlock this key, use it to generate an HMAC of the current time.  From that you end up with a 6 digit code, which you can then verify against the one generated in the TOTP app on your phone (google authenticator, or whatever you prefer).&#x20;

So if the PCR doesn't match, it shouldn't be able to decrypt the TOTP secret. For an attacker to hide tampering they would need to know the TOTP secret to be able to generate the correct 6 digit code. &#x20;

Another option for an adversary to fake the TOTP would be for them to in some way know (or manipulate) exactly when you will next use the computer, set the clock ahead and get the TOTP codes for that time, and then fake their output later after having done whatever tampering they were going to do.  So watch out for situation where someone may be manipulating you into using your computer at a specific time, or with a sense of urgency.  In fact that's generally good security advice.  Many attacks and scams alike rely on a sense of urgency, so if some situation is making you feel you need to act right now, stop and think about it for a minute or two. &#x20;

## Choosing which PCRs to use

You will need to see which PCRs will work for you.  Usually the defaults (which are 0,2,4,6) are good, but check that they are consistent on your machine. &#x20;

I found that on my system, some PCRs would have a different value every time it woke up from hibernate, and others don't appear to be used at all.  See the table here for what each PCR is supposed to represent: [https://uapi-group.org/specifications/specs/linux\_tpm\_pcr\_registry/](https://uapi-group.org/specifications/specs/linux\_tpm\_pcr\_registry/)

This should work for every boot, both normal cold boots, and resume from hibernate boots.  So if want to use hibernate, you should check that the PCRs you select report the same values for both states. &#x20;

After a normal  boot, check your PCRs with this command:&#x20;

```
sudo tpm2_pcrread
```

Then hibernate, and resume, and run the command again.  Check the output from both against each other and only use PCRs that show the same value both times. &#x20;

Also try again after changing a setting in the EFI setup (for example, after turning off secure boot).  You want to use PCRs that are the same after a boot and after resume from hibernate, but at least one that is different after booting with different EFI settings. &#x20;

If you are also going to customize your secure boot, be aware that the secure boot Keys and DB do get hashed and "extended" into some of the PCRs.  I know they are in PCR7, but it might be in others too.  So if you are going to approve individual kernels by just adding their hash to the DB, you probably want to leave out the PCRs that change when you do that.  (See the "Make Secure Boot Yours" page for details). &#x20;

## Steps to set this up

I suggest you set a password to let you re-seal the key if your system changes.  For example, after a firmware update, the PCRs will have different values.  If you set a password you can use it to seal the same key again with the new PCR values.  Without a password, you would instead have to create a new key to seal, and then add that key to your TOTP app. &#x20;

First we will create the TOTP secret, seal it in the TPM using the current PCR values. &#x20;

```
sudo tpm2-totp init -p 0,2,4,6 -P -
```

Type in whatever you want the password to be, hit enter, and then Ctrl-D. &#x20;

It will show the key as a QR code (it might be easier to scan on your phone if you shrink the font In you terminal).  Scan this code with your TOTP app on your phone.  Test that the code matches your phone when you run this command

```
sudo tpm2-totp show
```

On my system tpm2-totp set itself up to be included in the initrd, so all I had to do was update it with&#x20;

```
sudo update-initramfs -uk all
```

Then just reboot and check that if gives you the 6 digit code that matches your phone. &#x20;
