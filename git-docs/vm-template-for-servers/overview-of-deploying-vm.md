# Overview of Deploying VM

The basic goal is to be able to replace a VM like the nginx reverse proxy quickly and easily. Basically the process would be:&#x20;

1. Clone the VM template to a new template. &#x20;
2. Set the VM's network to be connected to the correct VLAN. &#x20;
3. Set the cloud-init settings to tell the VM what IP it should have, and which SSH key(s) should be able to log into it.
4. Optionally grow the disk (if the 8GB of the template is too small). &#x20;
5. Power on the VM, and wait for it to prepare itself. When I created the template I installed cloud-init, and cloud-initramfs-growroot, which will automate a lot of very basic setup. &#x20;
   1. Cloud-initramfs-growroot is a relatively simple script that runs from the initrd before root is mounted to see if there is unused space on the drive after the end of the last partition. If there is, it will grow the partition and the filesystem to use that space, and then the system will continue to mount root. So no effort to grow the disk, but it doesn't work with complicated setups like LUKS or LVM, so when I add LUKS to the template I will need to make my own script to replace this one.
   2. Cloud-init for basic settings.  The host wraps up the cloud-init settings you specify into a file and presents it to the VM as a virtual CD.  The cloud-init program will see the CD drive with settings, and set things up automatically, like adding a user with the specified name and password (or lack of password), the specified ssh key(s) in the authorized\_keys, and the specified network settings (IP, gateway, DNS, etc.).&#x20;
6. `ansible-playbook –ask-vault-pass rproxy_playbook.yaml`
   1. This calls several roles I have created like common which will install programs I like to have on all my servers (screen, snmpd, emacs, etc.), and push configs for those programs onto the VM.
   2. Another role will configure the unattended-upgrades. Variables I set for the VM in the inventory decide if it will install updates automatically, and if so is it allowed to reboot automatically for significant upgrades like new kernels.&#x20;
   3. And finally it runs a role specific to the type of server this will be. In the case of the example above, the rproxy role will install nginx, certbot, and fail2ban. Then it will push the config for those services and restart them. And there's an extra step to ensure that if the certificates pushed in that config are too old, they get renewed.

So I'm never more than 5 minutes away from a new reverse proxy.&#x20;
