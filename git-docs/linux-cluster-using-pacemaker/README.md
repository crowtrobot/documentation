---
description: >-
  Setting up an NFS server on a Linux Failover cluster with pacemaker and
  corosync on Debian 12 (Bookworm).
---

# Linux cluster using pacemaker

## Background

I couldn't find instructions for Debian 12, so this is cobled together from older instructions. Mostly from [https://wiki.debian.org/Debian-HA/ClustersFromScratch](https://wiki.debian.org/Debian-HA/ClustersFromScratch)

In my ongoing effort to reduce downtime and maintenance times for my VMs, I have decided to replace my NFS server with a failover cluster pair of NFS servers. The plan here is to keep things as simple as possible, while allowing the NFS servers to be updated and rebooted without disrupting the servers that depend on this NFS.

It might help to have some background on the NFS server that I will be replacing. This is a very simple NFS setup. I have a couple of servers in my server VLAN that exist to make some audio or video files available to the internet. For various reasons the files are not stored in the VMs, the files are in a directory in cephfs.

These server VMs that have some public exposure are isolated in their own vlan, and they are not allowed to access the more trusted networks, and the ceph storage is on one of those more trusted networks. To make the files from the cephfs directory available to these VMs I decided to make a small NFS VM. The VM has an interface in the servers vlan, and an interface in the network with ceph. The VM mounts the cephfs space read-only and then shares the files out to the servers via NFS.

## The setup

I already have an ansible playbook that makes and configures the NFS server, so the plan is to use that to setup NFS on both of the new servers, and make sure they both have the same configuration.  The pacemaker configuration is going to be a little hard to automate with an ansible role, but for now I will make the cluster manually. &#x20;

So first we need 2 VMs., and 3 IPs.  One IP will be the cluster IP, and it will be used by whichever is currently the active node. &#x20;

### Step 1 - Basic setup of both VMs

This was easy, just clone the template, assign the IP, and hostname.  The hostname is going to matter for the cluster setup later, so whatever name is assigned here will go into the corosync and pacemaker configs.  I picked "nfs-node1" and "nfs-node2" because I don't need server names to be creative. &#x20;

### Step 2 - Install programs

Then install NFS and the NFS config (/etc/exports), and make sure that NFS is working on both machines separately.  The NFS configuration needs to be identical on both machines.  It's easier to troubleshoot any problems making it work now, before bringing the cluster stuff into it. &#x20;

Also install pacemaker and the pacemaker configuration CLI tool wth  `apt install pacemaker crmsh` , even though we won't be using them yet.  We also want to stop NFS from starting automatically, we need it to be started only by pacemakter, so `sudo systemctl stop nfs-server ; sudo systemctl disable nfs-server` .

### Step 3 - Configure Corosync

Now we will need to make a key for the corosync encryption by running `corosync-keygen`. This makes the `/etc/corosync/authkey` file, which we will need to copy to the other node later.

Starting with the default config in `/etc/corosync/corosync.conf`, set the cluster name, turn on the encryption settings, and add the "two\_node: 1" option to be ready for just 2 nodes. In the "bindnetaddr" set the network range of the trusted network interface. Make sure the node names match the hostnames of the servers.  The config looks like this:

```
totem {
        version: 2

        # Corosync itself works without a cluster name, but DLM needs one.
        # The cluster name is also written into the VG metadata of newly
        # created shared LVM volume groups, if lvmlockd uses DLM locking.
        cluster_name: nfs_cluster

        # crypto_cipher and crypto_hash: Used for mutual node authentication.
        # If you choose to enable this, then do remember to create a shared
        # secret with "corosync-keygen".
        # enabling crypto_cipher, requires also enabling of crypto_hash.
        # crypto works only with knet transport
        crypto_cipher: aes256
        crypto_hash: sha1
        interface {
                  ringnumber: 0

                  bindnetaddr: 172.16.10.0

                  mcastaddr: 239.255.1.1
                  mcastport: 5405
                  ttl: 1
        }
}

logging {
        # Log the source file and line where messages are being
        # generated. When in doubt, leave off. Potentially useful for
        # debugging.
        fileline: off
        # Log to standard error. When in doubt, set to yes. Useful when
        # running in the foreground (when invoking "corosync -f")
        to_stderr: yes
        # Log to a log file. When set to "no", the "logfile" option
        # must not be set.
        to_logfile: yes
        logfile: /var/log/corosync/corosync.log
        # Log to the system log daemon. When in doubt, set to yes.
        to_syslog: yes
        # Log debug messages (very verbose). When in doubt, leave off.
        debug: off
        # Log messages with time stamps. When in doubt, set to hires (or on)
        #timestamp: hires
        logger_subsys {
                subsys: QUORUM
                debug: off
        }
}

quorum {
        # Enable and configure quorum subsystem (default: off)
        # see also corosync.conf.5 and votequorum.5
        provider: corosync_votequorum
        two_node: 1
        expected_votes: 2
}

nodelist {
        # Change/uncomment/add node sections to match cluster configuration

        node {
             # Hostname of the node
             name: nfs-node1
             # Cluster membership node identifier
             nodeid: 1
             # Address of first node
             ring0_addr: 172.16.10.25
        }

        node {
             name: nfs-node2
             nodeid: 2
             ring0_addr: 172.16.10.26
        }
}
```

Copy the `/etc/corosync/authkey` and `/etc/corosync/corosync.conf` files to the other node, and restart corosync on both sides with `systemctl restart corosync`.

### Step 4 - Basic configure Pacemaker

The Pacemaker config can quickly get very complicated, so the provide a control CLI to help.  This means it's probably going to be hard to make the cluster config work with ansible, but that's a future problem. For now, it's just going to be manual configuration.

To start configuring the cluster, `sudo crm configure`. We will temporarily disable stonith, and turn it on later after we have configured the fencing. When we turn stonith back on later I will explain what it does. &#x20;

```
    property stonith-enabled=no
    property no-quorum-policy=ignore
```

Now we will create a cluster IP as a resource managed by pacemaker. This IP will be held by one of the nodes at a time, and will be assigned to "eth1" on that node (that's the interface on the servers VLAN in this example). This is the cluster IP, not an IP already assigned to either node.  This resource will be called "IP-nfs" and will be of type `ocf:heartbeat:IPaddr2` , which is a cluster IP. The "params" are parameters setting what IP address to use and what NIC to assign it to, etc.

```
 primitive IP-nfs ocf:heartbeat:IPaddr2 \
 params ip="172.16.10.25" nic="eth1" cidr_netmask="24" \
 meta migration-threshold=2 \
 op monitor interval=20 timeout=60 on-fail=restart
```

Then we will set another resource, the nfs server. We only want this started on the node with the cluster IP, and the IP has to be there first. If the NFS is already running when the IP gets added, it won't be listening on that new IP.

```
primitive NFS-rsc ocf:heartbeat:nfsserver \
  meta migration-threshold=2 \
  op monitor interval=20 timeout=60 on-fail=restart
```

So now we have the IP and nfs server setup in the cluster, we need to set the dependence so that NFS only runs on the node with the IP, and only after the IP is up

```
colocation lb-loc inf: IP-nfs NFS-rsc
order lb-ord inf: IP-nfs NFS-rsc
```

And then save and activate the config with: `commit`

So that was the easy part, `sudo crm status` or `sudo crm_mon` should show that there are 2 nodes in the cluster that are working, and one has the IP and NFS server. Now we need the cluster to handle possible problems.

### Step 5 - Configure Fencing

Since there are only 2 nodes, we can't use a "most votes wins" sort of cluster setup. That means that if things go wrong in just the right way, it is possible for each node to think that the other is broken, and to try to take control (a problem called split-brain). Both nodes trying to claim the cluster IP at the same time is a sure way to make sure that nothing works, so we need some sort of conflict resolution to try to make sure that no matter what (as much as possible) we are left with either just one node standing, or both working together. That's where fencing comes in.

This isn't a swordfight between the nodes, but a process by which either node can cut the other one off (or fence it off) from resources. We will use stonith type fencing. Stonith stands for "shoot the other node in the head". For this I will use the fence\_pve from the fence-agents package to let either node connect to proxmox and tell it to reboot the other node.

Rather than take a tangent here, I will detail how I created a proxmox user with minimal permissions in this [separate document.](creating-a-proxmox-user-for-fencing.md)  I called my user nfs\_fence.

```
sudo crm 
configure
property stonith-enabled=yes
```

So in just within this config we will turn stonith back on. Then we will set up the fence actions. First the action "fence nfs-node1" which will use fence\_pve. The parameters for this script that we will be using are:

* ip - This is the IP of the proxmox host to connect to.
* pvs\_node\_auto - Tells the script to automatically figure out which host from the proxmox cluster holds the VM we are after.
* ssl\_insecure - Accept a self-signed certificate from the proxmox host
* plug - For some reason, this is used to identify the VM (by its number)
* username - The username to log into proxmox with
* password - The password for that user to log into proxmox with
* vmtype - Set if the proxmox VM is a VM or container
* action - What action to take with the VM (reboot, or power off, etc.).
* delay - Setting a higher delay for one node makes sure that both nodes don't reboot each other at the same time. One will reboot and the other remain up.

```
primitive fence_nfs-node2 stonith:fence_pve \
params ip="172.16.1.21" pve_node_auto="true" ssl_insecure="true" \
plug="674" username="nfs_fence@pve" password="password" action="reboot" \
vmtype="qemu" delay="15" \
op monitor interval=60s meta target-role="Started" is-managed="true"
```

And for the other one it's basically the same but without the delay.

```
primitive fence_nfs-node1 stonith:fence_pve \
params ip="172.16.1.21" pve_node_auto="true" ssl_insecure="true" \
plug="673" username="nfs_fence@pve" password="password" action="reboot" \
vmtype="qemu" \
op monitor interval="60s" meta target-role="Started" is-managed="true"
```

And now we add some rules. We don't want a VM responsible for fencing itself, so we setup some location rules:

```
location l_fence_nfs-node1 fence_nfs-node1 -inf: nfs-node1
location l_fence_nfs-node2 fence_nfs-node2 -inf: nfs-node2
commit
```

### Step 6 - Test and fix Fencing

Now, here I found an interesting problem. I tried to test the fencing:

```
sudo crm
node fence nfs-node2
```

The fencing failed. In the log it said:

```
warning: fence_pve[8110] stderr: [   File "/usr/share/fence/fencing.py", line 975, in fence_action ]
warning: fence_pve[8110] stderr: [     print(outlet_id + options["--separator"] + alias) ]
warning: fence_pve[8110] stderr: [           ~~~~~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~ ]
warning: fence_pve[8110] stderr: [ TypeError: unsupported operand type(s) for +: 'int' and 'str' ]
```

There is a bug report for this: [https://github.com/ClusterLabs/fence-agents/issues/580](https://github.com/ClusterLabs/fence-agents/issues/580)

The fix, at least for now:

```
sudo cp -p /usr/share/fence/fencing.py /usr/share/fence/fencing.py.original
sudo sed -i 's/print(outlet_id/print(str(outlet_id)/g' /usr/share/fence/fencing.py
```

So once it's clear that fencing is working, you are done.  We can now install updates and reboot without interrupting the servers, as long as we reboot only one node at a time and make sure that the cluster is in a good state before rebooting the other. &#x20;
