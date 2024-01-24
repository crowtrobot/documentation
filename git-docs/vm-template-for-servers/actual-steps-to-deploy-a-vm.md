# Actual Steps to Deploy a VM

At some point in the future I think I will be able to make ansible do the steps on proxmox to clone the VM if the reverse proxy VM doesn't exist. I already have the command line commands worked out, I just haven't figured out yet if there's a plugin or whatever for ansible, or if it would just need to ssh into the host, and run those commands.&#x20;

But until that time, the commands look like this (run on the Proxmox host):

1. `qm clone 900 208 --name rproxy --full true --pool "Servers" --storage ssd_rbd --target titanium`&#x20;
   1. This will clone VM 900 (my debian template), to VM 208. &#x20;
   2. It will name that VM "rproxy"
   3. It will make it a full copy not a linked clone.
   4. It will add the VM to the Servers pool.  This automatically means that the VM will will be backed up with the next round of backups. &#x20;
   5. Put the virtual disks in the ceph "ssd\_rbd" pool.
   6. And set the VM to run on the host titanium
2. `qm resize 208 scsi0 8G`&#x20;
   1. This will grow the disk by adding another 8GB to it.  I don't think the reverse proxy really needs this. &#x20;
3. &#x20;`qm set 208 --cpu cputype=x86-64-v3 --cores 2 --memory 1024 --tablet=0 --ciuser admin --ipconfig0 gw=192.168.69.1,ip=192.168.69.10/24 --net0 model=virtio,bridge=vmbr10 --numa 0 --hotplug disk,network,usb`&#x20;
   1. Set the VM to use "x86-64-v3" type CPU.  I used to use "host" but since the CPUs in my Proxmox hosts are not the same this could cause problems when moving the VM to another host. &#x20;
   2. Set the vCPU to 2 cores&#x20;
   3. Set the RAM to 10224MB&#x20;
   4. Set the mouse not to use tablet mode (which I think sends absolute coordinates instead of relative). This works around an issue that caused a few extra % of CPU when the machine was idle.
   5. Set the username for cloudinit to "admin"
   6. Set the IP and gateway
   7. Set the network to use the virtio NIC
   8. Connected that NIC to bridge vmbr10 (vlan10's bridge, vlan 10 is my servers isolated network)&#x20;
   9. Turn off numa.  I think it would need to be on to hot-add RAM, but I don't need to do that. &#x20;
   10. Enable hotplug for disks, network, and usb&#x20;
4. `ha-manager add "vm:208"`&#x20;
   1. Add the VM to the high availability system. &#x20;
5. `ha-manager set "vm:208" --group ti_first --state started`&#x20;
   1. Add this VM to the ti\_first group (which will run on the host titanium if it is up, otherwise on one of the others)
   2. and set the HA state to started (start the VM if it is ever seen to be off, like now).

And that's it.  From there the Debian template VM is now a VM with the right IP etc. that's going to be needed to be able to setup with ansible.  So all that's needed is to ssh in once to get the host key added to the known\_hosts, and then run `ansible-playbook --ask-vault-pass rproxy_playbook.yaml`
