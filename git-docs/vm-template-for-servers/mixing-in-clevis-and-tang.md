# Mixing in Clevis and Tang

As the saying goes "encrypt all the things"... or at least everything you don't have a good reason to store unencrypted. &#x20;

For various reasons (some of them being work related), I have recently been reading about how to set up a tang server, and use clevis on a machine to unlock a LUKS volume using keys from that Tang server.  It seemed like it couldn't be as simple as what I was reading, which meant it was time to actually try it for myself. &#x20;

I haven't dug in very deep yet into what's going on inside tang, but it was quite easy to set up.  There's not much more to it than just apt-get install tang.  It will create a pair of keys and you can use the tang-show-keys command to see what keys it has to offer (it will only show half of them, I assume that's the public keys).  Obviously if someone gets both the public and private keys they have everything that is locked with those keys, so I didn't deploy this VM with the normal template.  I did a fresh install of debian so I could have the files in a LUKS volume with a nice long password.  There will be a separate page about the setup of the tang server later. &#x20;

For the client side of things, we have clang.  Again I haven't dug as far into this yet as I would like, but it seems clang can do a few simple crypto things, and combine some into more complex crypto things.  For example, clang can encrypt a file using a key it gets from the TPM, or another source, and for our purposes that's going to be some sort of key exchange with Tang.  I think it will create a random number, and get the tang server to mix it with it's private key in some way that is repeatable, forming a secret that is dependent on knowing the random number, and the private key from tang, and using that as a key for something.  The key exchange with Tang is supposed to be safe from someone snooping on the network, which is a nice touch. &#x20;

There's some other stuff to investigate here too, like using Shamir's Secret Sharing in clevis to let any X out of Y tang servers be enough to unlock the drive (like any 2 out of 4 tang servers, or 1 out of 5 or whatever).  Or to require a tang server and the TPM, if you want to go that way. &#x20;

So, for the first test, I just made a regular ubuntu desktop VM with the root filesystem in a LUKS encrypted volume (using LUKS2 headers).  Then installed clevis, clevis-luks, and clevis-initramfs, and add the key from tang as a password in LUKS.  That is done with this command

&#x20;`clevis luks bind -d /dev/sda2 -s8 -y tang '{"url":"http://192.168.x.254/"}'`&#x20;

That tells clevis to make a password for LUKS on /dev/sda2 with the tang server at 192.168.68.254, and put it in key slot 8 in the LUKS header (so I can easily tell it apart from the normal passwords).  Now during boot it will ask for the LUKS password on the boot screen, but if I just wait a moment it will get the password from tang, and unlock LUKS.  Obviously if it can't (network down, or the tang server not available), it will still accept the password I had set before (keyslot 0). &#x20;
